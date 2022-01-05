---
layout: post
title: "Handling syslog clients sending blank fields"
published: true
post_date: 2022-01-05 12:00
---

## Problem

I recently starting centrally collecting logs on my home network using [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/), [Loki](https://grafana.com/docs/loki/latest/) and [Grafana](https://grafana.com/oss/grafana/) This lets you easily search and analyse logs for all sorts of things.

One of the services I wanted to collect logs for was my TP-Link Wireless Access Points managed by TP-Link Omada. However, it turns out that the APs weren't setting the `app-name` field correctly when sending logs to the remote server.

This resulted in errors in `promtail` that frequently reset the connection to `rsyslogd` that was acting as a syslog relay.

This was the error I could see in the logs:

`promtail`:

```
level=warn ts=2022-01-04T04:08:21.013648461Z caller=syslogtarget.go:216 msg="error parsing syslog stream" err="expecting an app-name (from 1 to max 48 US-ASCII characters) or a nil value [col 50]
```

`rsyslog`:

```
rsyslogd: omfwd: TCPSendBuf error -2027, destruct TCP Connection to promtail:514 [v8.36.0 try http://www.rsyslog.com/e/2027 ]
rsyslogd-2027: omfwd: TCPSendBuf error -2027, destruct TCP Connection to promtail:514 [v8.36.0 try http://www.rsyslog.com/e/2027 ]
rsyslogd-2007: action 'action 2' suspended (module 'builtin:omfwd'), retry 0. There should be messages before this one giving the reason for suspension. [v8.36.0 try http://www.rsyslog.com/e/2007 ]
rsyslogd-2359: action 'action 2' resumed (module 'builtin:omfwd') [v8.36.0 try http://www.rsyslog.com/e/2359 ]
```

## Environment

Syslog Clients:

- Omada Controller 4.4.8
- TP-Link EAP 620 HD (x2)

`rsyslogd` configured as a forwarder to `promtail`. `docker-compose` extract below:

```yaml
version: '3'

services:
#    -- snip --
  loki:
    image: grafana/loki:2.4.1
    restart: unless-stopped
    volumes:
      - ./loki-config.yaml:/mnt/config/loki-config.yaml
    command:
      - --config.file=/mnt/config/loki-config.yaml
  promtail:
    image: grafana/promtail:2.4.1
    restart: unless-stopped
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yaml:/mnt/config/promtail-config.yaml
    command:
      - --config.file=/mnt/config/promtail-config.yaml
  rsyslog:
    image: rsyslog/syslog_appliance_alpine
    restart: unless-stopped
    ports:
      - 514:514/udp
      - 514:514/tcp
    environment:
      - RSYSLOG_CONF=/config/rsyslog.conf
    volumes:
      - ./rsyslog.conf:/config/rsyslog.conf

```

`promtail` configuration:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      __path__: /var/log/*log
  pipeline_stages:
    - regex:
        expression: "^[\\w]+\\s+[\\d]+\\s+[\\d|:]+ (?P<host>[^\\s]+) (?P<tag>[^:\\[]+)"
    - labels:
        host:
        tag:
- job_name: syslog
  syslog:
    listen_address: "0.0.0.0:514"
    idle_timeout: 10m
    label_structured_data: yes
    labels:
      job: "syslog"
  relabel_configs:
    - source_labels: ["__syslog_connection_ip_address"]
      target_label: "ip_address"
    - source_labels: ["__syslog_message_severity"]
      target_label: "severity"
    - source_labels: ["__syslog_message_facility"]
      target_label: "facility"
    - source_labels: ["__syslog_message_app_name"]
      target_label: "appname"
    - source_labels: ["__syslog_message_hostname"]
      target_label: "host"
```

`rsyslog.conf` configured according to: https://grafana.com/docs/loki/latest/clients/promtail/scraping/#rsyslog-output-configuration

## Solution

The way I solved this problem was by configuring `rsyslog` to use a modified template that sets the `app-name` to `-` (nil) when the `app-name` field is blank. The resulting configuration is shown below. Note that this solution should work for any field (e.g. `hostname`) send from mis-behaving syslog clients.

```
...

:app-name, !isequal, "" {
    action(type="omfwd" protocol="tcp" target= "promtail" port="514" Template="RSYSLOG_SyslogProtocol23Format" TCP_Framing="octet-counted" KeepAlive="on")
}

# RSYSLOG_SyslogProtocol23Format but with app-name hard-coded to '-'
template(name="missingAppName" type="string" string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% - %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n")

:app-name, isequal, "" {
    action(type="omfwd" protocol="tcp" target= "promtail" port="514" Template="missingAppName" TCP_Framing="octet-counted" KeepAlive="on")
}

...
```

## References

I found the following links useful while working through this problem:

- https://github.com/grafana/loki/issues/4270
- https://datatracker.ietf.org/doc/html/rfc5424#page-8
- rsyslog documentation:
    - https://www.rsyslog.com/doc/v8-stable/configuration/templates.html
    - https://www.rsyslog.com/doc/v8-stable/configuration/properties.html
    - https://www.rsyslog.com/doc/v8-stable/configuration/converting_to_new_format.html
- https://serverfault.com/questions/1040972/rsyslog-forward-using-original-source-ip-over-tls
