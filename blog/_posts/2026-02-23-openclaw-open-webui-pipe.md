---  
layout: post  
title: "Building an OpenClaw interface in Open WebUI"  
published: true  
post_date: 2026-02-23 12:00  
---  

I use Open WebUI as my primary AI chat interface. It gives me a ton of flexibility, the UI is highly functional, and it feels incredibly familiar to anyone used to modern AI workflows.  

With OpenClaw coming onto the scene recently, I was keen to try it out. Initially, I had it hooked up via Telegram, but I quickly felt constrained by having to rely on a single continuous conversation thread. My only workaround was to clunkily create new group chats with the bot just to have separate channels for different topics. 

I realized I really missed the ability to literally context-switch between different projects—exactly the way I do natively inside Open WebUI.  

So I thought: *How can I bring these two tools together?*  

### Finding the Right Connection

Seeing that OpenClaw has its own chat web UI powered by a custom WebSocket protocol, I figured that was the best place to start.  

I did briefly consider using the standard `/chat/completions` API functionality built into the OpenClaw Gateway. However, that approach came with a few major drawbacks. First, I really wanted to be able to see the agent's tool calls in real time, and the Gateway integration didn't surface these well. Second, standard chat completions are inherently "stateless." This means the full conversation context would have to be sent by Open WebUI (or any other chat completions client) on every single turn. Given the architecture of OpenClaw, I didn't think forcing it into a stateless paradigm would work very well for complex agentic tasks.  

### Enter Open WebUI Pipes

To bridge this gap, I turned to Open WebUI's **Pipes**. 

Pipes are an incredibly powerful feature in Open WebUI that allow you to create custom AI server backends using arbitrary Python code. Instead of routing a user's prompt to a standard LLM endpoint, a Pipe intercepts the incoming messages and lets you define exactly how to process them. You can connect to external services, trigger custom logic, and yield streaming responses back to the user interface.

For this project, my custom Pipe needed to behave like a native Open WebUI model on the front end, while acting as an OpenClaw WebSocket client on the back end. It had to forward incoming user messages to OpenClaw, wait for the agent to do its work, capture the stream of text and native tool calls, and accurately translate all of that back into a format Open WebUI could display.

### The Implementation Hurdles

Getting this to work took a bit of trial and error. 

The trickiest part out of the gate was sorting out the device pairing. The documentation on how OpenClaw establishes its WebSocket authentication isn't very thorough right now. After tracing through the expected handshakes and going through many iterations, I finally got the Pipe client successfully paired and communicating with OpenClaw.  

The second major hurdle was figuring out how to neatly format the responses cleanly back into Open WebUI, specifically the tool calls. I wanted the internal tools that OpenClaw was executing to be fully visible in Open WebUI, exactly like they are when you use a native LLM with standard tool integration. 

The caveat here, of course, is that these tools live entirely within OpenClaw's execution environment, not within Open WebUI. Translating OpenClaw's internal tool-use payloads into the specific UI state Open WebUI expects required some trickery and fighting with the UI rendering logic. After a lot of tweaking, everything finally snapped into place!  

<img width="1015" height="369" alt="Open WebUI running OpenClaw with visible tool calls" src="https://github.com/user-attachments/assets/423d081f-cf3f-4028-8435-cbdc14ea92d6" />  

### Check it out

If you want to bring the power of OpenClaw into your Open WebUI workspace, feel free to check out my implementation. 

I've published the code in a gist here:  
[https://gist.github.com/cfullelove/7c6fa74e16d0a8f355e6d5ddb6d8e5fb](https://gist.github.com/cfullelove/7c6fa74e16d0a8f355e6d5ddb6d8e5fb)
