---
layout: post
title: "A Simple SSH Proxy for Life Between Home and Work: go-home-or-away"
published: true
post_date: 2025-06-23 12:00
---

Moving a laptop between home and work networks can be a pain, especially when it comes to keeping SSH sessions alive and seamless. I recently ran into this problem with VSCode’s remote SSH feature: at home, I could connect directly to my servers, but at work, I needed to go through a proxy. My old solution was to maintain separate `.ssh/config` entries for each location, using `ProxyJump` for the remote connection. This meant restarting VSCode sessions every time I switched networks—a hassle I wanted to avoid.

### The Solution: go-home-or-away

To make my life easier, I built a small tool called `go-home-or-away`. It’s a conditional proxy for SSH, written in Go, designed to be used as a `ProxyCommand` in your SSH config. The idea is simple: if you can reach your target host directly, it connects straight through. If not, it automatically proxies your connection via a specified SSH server.

This is perfect for anyone who moves between networks with different access rules—like working from home, the office, or a coffee shop—without wanting to constantly tweak SSH configs or restart sessions.

### How It Works

The logic is straightforward:
- **When you’re “home”** (i.e., direct TCP connection to the target host/port works), it connects directly.
- **When you’re “away”** (direct connection fails), it falls back to proxying through your chosen SSH server using `ssh -W`.

Setup is simple and works on both Windows and Linux. Just drop the binary somewhere in your path and update your SSH config to use it as a `ProxyCommand`. The README in the repo has copy-pasteable examples for both platforms.

### Why Go?

I chose Go for this project because it lets you build a single, static binary—no need for messy scripts or extra dependencies. The implementation was pretty simple, and as a bonus, it gave me a chance to use Cline with Gemini 2.5 Pro to generate most of the code. (The code isn’t complex, but it was nice to automate the boring parts.)

### Setup & Usage

The setup I use is almost exactly what’s in the README. On Linux, I put the binary in `/usr/local/bin` and add a `ProxyCommand` line to my `~/.ssh/config`. On Windows, it’s the same idea—just point to the `.exe` in your config. I haven’t run into any issues so far; it “just worked” out of the box.

### Who’s This For?

If you, like me, move between networks and want your SSH sessions (and VSCode remote connections) to “just work” no matter where you are, this tool is for you. No more juggling multiple config entries or restarting sessions.

### Feedback & Future

I don’t have any big plans for new features, and I’m not looking for contributors right now, but feedback is always welcome.

If you’re interested in trying it out, the code and binaries are available here: [github.com/cfullelove/go-home-or-away](https://github.com/cfullelove/go-home-or-away)

### Conclusion

Sometimes the best tools are the ones that quietly solve a real problem without much fuss. `go-home-or-away` has made my daily workflow a little smoother, and maybe it’ll help someone else too.
