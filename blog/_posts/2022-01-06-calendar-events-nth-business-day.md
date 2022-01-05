---
layout: post
title: "Creating calendar events on the nth business day each month"
published: true
post_date: 2022-01-04 12:00
---

## Problem

We have monthly reports that are due each month and I wanted to create an automated reminder that they were due using Slack and Google Calendar using Zapier.

Unfortunately, the reports are due in a pattern that Google Calendar repeating events doesn't support:

- The 10th business day of the month. E.g. in February 2022 the 10th business day is Monday 14th
- The first business day that is on or after the 10th day of the month. E.g. 10th October 2021 is a Sunday, so the report would be due the next day on Monday 11th October

## Solution

Luckily, while Google Calendar doesn't let you create these types of events in web app, it does let you add a custom event that has been crafted somewhere else.

I found [a great article](https://medium.com/scrubbi/creating-a-google-calendar-event-that-repeats-on-the-first-weekday-of-every-month-9ca0113eedea) that really explained the process well and how these custom events needed to work. Using this I was able to create the calendar events for the two scenarios I was after

### 10th Business Day of the month

```
BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
RRULE:FREQ=MONTHLY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR;BYSETPOS=10
SUMMARY:Brisbane Report Due
LOCATION:msa-reporting
DTSTART;VALUE=DATE:20210701T090000
SEQUENCE:0
DESCRIPTION:@here The Brisbane report is due today! 
END:VEVENT
END:VCALENDAR
```

The `RRULE` is what defines the recurrence behaviour. Breaking that line down to see how it works:

- `FREQ=MONTHLY` - repeat monthly
- `INTERVAL=1` - repeat each month
- `BYDAY=MO`,TU,WE,TH,FR - only on weekdays
- `BYSETPOS=10` - the 10th day that matches the above rule

### First Business Day on or after the 10th day of the month

```
BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
RRULE:FREQ=MONTHLY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR;BYMONTHDAY=10,11,12,13;BYSETPOS=1
SUMMARY:Melbource Report Due
LOCATION:msa-reporting
DTSTART;VALUE=DATE:20210701T090000
SEQUENCE:0
DESCRIPTION:@here The Melbourne report is due today! 
END:VEVENT
END:VCALENDAR
```

This one is a bit more complicated. Breaking that line down to see how it works:

- `FREQ=MONTHLY` - repeat monthly
- `INTERVAL=1` - repeat each month
- `BYDAY=MO,TU,WE,TH,FR` - only on weekdays
- `BYMONTHDAY=10,11,12,13` - AND only on the 10th - 13th day of the month (handling when the 10th lands on a weekend)
- `BYSETPOS=1` - the first day the matches the previous two constrains (weekday && 10-13 day of the month)


## Constraints

Unfortunately, neither of these rules don't account for weekdays that aren't business days such as public holidays. For example, in January 2022 the custom event for the 10th business day will show the Friday 14th, however, Monday the 3rd is a public holiday so, in fact, the 10th business day is Monday 17th

## References

I found the following links helpful when working on this problem: 

- [Creating a Google Calendar Event that Repeats on the FIRST WEEKDAY of Every Month](https://medium.com/scrubbi/creating-a-google-calendar-event-that-repeats-on-the-first-weekday-of-every-month-9ca0113eedea)

- [https://www.kanzaki.com/docs/ical/vevent.html](https://www.kanzaki.com/docs/ical/vevent.html)
- [https://www.kanzaki.com/docs/ical/recur.html](https://www.kanzaki.com/docs/ical/recur.html)
- [http://www.ietf.org/rfc/rfc2445.txt](http://www.ietf.org/rfc/rfc2445.txt)