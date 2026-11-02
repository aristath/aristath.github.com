---
layout: post
title: "Context management in agents"
permalink: "blog/context-management-in-agents"
categories:
  - AI
---

In my [previous post](/blog/ai-independence-journey) I wrote about my local AI setup, the hardware, and some of the things I built to make it useful. One of the projects I've been working on since then is Clara, a self-hosted personal assistant. I have not open-sourced her codebase yet, but it's a project I've been working on for the past year for my own personal needs. OpenClaw and all alternatives just did not do what I needed, and the mental load they were carrying was just too much for me. There's a lot going on in that project, but for this post I want to focus on one specific part: How I chose to keep a conversation going without constantly filling the model's context window.

The obvious solution (and what I was using for the first few months) is context compaction. At some point the conversation gets too large, so the "traditional" approach is to summarize it and continue from that summary. It's a reasonable approach, it's what we are familiar with because it's what everyone else does, and it's relatively easy to implement. But for Clara I ran into issues with that approach and wanted something a little different.

What "traditional" context-window management would look like for a human - very rough approximation:
- I know everything that I said and the interactions I had, until my head is about to explode
- When I reach my breaking point, I fall into a coma for a couple of minutes
- When I wake up from my coma I have a vague impression of what happened previously but otherwise I'm a blank slate.

That's what every agent I have used does... They fill the context, then when it reaches 80%, 90% or whatever the harness sets and they reach that "breaking point", they compact the context and start over. The new context is empty and contains just the summary of what previously happened. When the context fills again, we repeat the process. We compact the previous summary plus any messages that arrived since then to a new summary, and start over.

That's... completely inefficient. It's insane. It's lazy and extremely lossy!

I want to be able to have long conversations, go off on tangents, return to something we discussed earlier, change my mind, and expect the assistant to follow along. I want her to be able to follow along with my ADHD brain. Sometimes we're working on something, other times we're just talking, other times it's both. A conversation doesn't always have a goal or a list of tasks that can neatly fit in a handoff document.

So what I did, was think about how I perceive **my own** thinking processes:
- I am always focused on the current discussion, I have the details of what we are discussing **now**. 
- What we discussed half an hour ago is there, but details fade in my brain. 
- Something that we discussed 5 hours ago is not even in there, but if I think about it a bit, I can recall the details.

So I decided to model the way Clara works, after how I perceive my own brain functions.

## TL;DR

Instead of summarizing the whole conversation, we compact each message individually. Clara gets the latest 20 messages verbatim. Older messages stay in context as individual summaries when they have at least 50 words; shorter messages keep their original wording. Clara has tools to read the originals when she needs more detail. When the conversation grows beyond her 256k context window, we drop the oldest messages from context. They stay saved, and she can retrieve them with the same tools.

---

## What Clara does

The complete conversation stays in a Markdown file. That's the original record, and that's what I see in the chat interface. Compaction doesn't change it.

We prepare a separate summary for each long message, both mine and Clara's. When building a request for the model, we keep the latest **20 messages verbatim**. That's 20 messages total, not 20 pairs of user/assistant exchanges.

For older messages, we use their individual summaries. Messages with fewer than **50 visible words** keep their original wording. There's little benefit in shortening "Yes, but please keep the old version too".

The messages stay in the order they were sent. So Clara can still follow the exchange: I suggested something, she replied, I changed my mind, she replied again. Each message has its own place in the discussion and its own reference to the original.

**Every message is compacted individually from the original message.** We don't keep summarizing the same summary as the conversation grows.

Here's a made-up example in the actual file format. Clara reads a configuration file and replies with what she found. The reply has more than 50 words, so it gets its own summary.

### The message saved in the Markdown file

This is one complete message block. The conversation's header and its other messages aren't shown here.

```markdown
<!-- clara:message -->
{"id":2747,"role":"assistant","created_at":"2026-11-02T08:15:00Z","version":2,"simple":false}

<!-- clara:part -->
{"type":"text"}
<think>I should read the configuration before answering.</think>
<!-- clara:part -->
{"type":"tool_call","id":"call-1","name":"read_file","args":{"path":"config.json"}}

<!-- clara:part -->
{"type":"tool_result","id":"call-1","name":"read_file"}
{"retries":3,"timeout_seconds":30}
<!-- clara:part -->
{"type":"text"}
The settings allow three retries and a thirty-second timeout per request. I have only read the file; I haven't changed it. These settings are stored in config.json. If you want to change either value, we can edit that file, but for now both values remain exactly as they were when I opened it.
<!-- clara:compacted -->
{"version":4,"source":"60043087f4f8353d2802e46a565d70d30f2439905567f3440d65adcc9fb1e0f2","recipe":"8ed51fab90e528f60aeddc3500eb84d9b7fd07002a3b15dc6f3d9a3109a55090","messageId":2747,"methods":["subconcious"],"createdAt":1793607360000}

<!-- clara:segment -->
{"type":"marker"}
{"kind":"tool_result","name":"read_file","call_id":"call-1","status":"completed","compacted":true,"result_id":"message:2747:result:call-1","instruction":"Result compacted. Retrieve the original with read_tool_result({\"result_id\":\"message:2747:result:call-1\"})."}
<!-- clara:segment -->
{"type":"text"}
Read config.json: three retries, thirty-second timeout. No changes made.
<!-- clara:end-message -->
```

The `part` blocks hold the original pieces: reasoning, the tool call and its arguments, the tool result, and Clara's reply. The `compacted` block holds the summary's metadata, followed by `segment` blocks for the shorter version. The hashes identify the original content and the compaction settings. Both versions live in the same message record.

### What Clara sees while it's in the latest 20

The saved summary is already there, but Clara still gets the full visible reply and the tool result. The reasoning is left out. The labels in square brackets below show the API message roles; they aren't part of the text.

```text
[system, name=clara_runtime]
<clara_runtime>
{"kind":"message_source","session_id":"example-chat","message_id":2747,"role":"assistant","created_at":"2026-11-02T08:15:00Z"}
</clara_runtime>

[system, name=clara_runtime]
<clara_runtime>
{"kind":"tool_result","name":"read_file","call_id":"call-1","status":"completed","result_id":"message:2747:result:call-1","result":"{\"retries\":3,\"timeout_seconds\":30}"}
</clara_runtime>

[assistant]
The settings allow three retries and a thirty-second timeout per request. I have only read the file; I haven't changed it. These settings are stored in config.json. If you want to change either value, we can edit that file, but for now both values remain exactly as they were when I opened it.
```

### What Clara sees once it's older than the latest 20

Now she gets the shorter reply. The tool result becomes a record of what ran, its status and how to read the original output. The message keeps the same source ID, with `compacted: true` added to tell her she's seeing a shorter version.

```text
[system, name=clara_runtime]
<clara_runtime>
{"kind":"message_source","session_id":"example-chat","message_id":2747,"role":"assistant","created_at":"2026-11-02T08:15:00Z","compacted":true}
</clara_runtime>

[system, name=clara_runtime]
<clara_runtime>
{"kind":"tool_result","name":"read_file","call_id":"call-1","status":"completed","compacted":true,"result_id":"message:2747:result:call-1","instruction":"Result compacted. Retrieve the original with read_tool_result({\"result_id\":\"message:2747:result:call-1\"})."}
</clara_runtime>

[assistant]
Read config.json: three retries, thirty-second timeout. No changes made.
```

The Markdown file stays the same. Moving past the latest 20 changes which version goes into Clara's context, not what we keep on disk.

## When compaction happens

When Clara finishes replying, the conversation goes into a background queue. When Clara isn't busy replying, the *subconscious* model prepares shorter versions of the messages, one at a time, starting with the newest. Yes, I said "subconscious". That's a whole different discussion... Clara has a subconscious but that's a topic for another post, not relevant to the context-window topic of this post.

When I open an older conversation, it gets queued too. Messages that already have a valid summary are skipped.

The latest 20 messages still reach Clara in their original wording, even when their summaries are ready. When a message falls outside those 20, its summary can be used in the next request. Until a summary is ready, we keep the original text.

So we don't wait for the context window to reach 80% or 90%. We prepare the summaries as we go and use them as messages get older.

## How a message gets compacted

Clara's subconscious model handles the compaction. We send it the original text and ask it to summarize it. 

This is the prompt:

> Summarize the meaning of this message into a single short paragraph using as few words as possible, preserving the conversational thread.

We send that prompt followed by the message's original text. The model returns a shorter paragraph. Both the prompt and the model are configurable.

Only messages with **at least 50 visible words** go through this. The count includes the words we actually wrote, not reasoning, tool results or metadata. A large tool result doesn't make a short reply qualify as a long message.

We separate the message's visible text from its tool activity and reasoning before sending it to the compactor. If there's text before and after a tool result, those text sections are shortened separately. The tool activity stays in its place.

The summarization request uses `temperature: 0`, `thinking: false`, and a 256-token output limit per text section. We accept a complete summary when it's at least **40% shorter**, counting characters. The accepted summary is saved beside the original message and reused on later requests.

## Getting the original back

A summary tells Clara what a message was about. When she needs the details, she can read the original.

In the configuration example above, the message reference gives her both IDs: message `2747` in conversation `example-chat`. Clara can pass those IDs to `read_chat_messages`:

```json
{
  "session_id": "example-chat",
  "message_ids": [2747]
}
```

The summary keeps the settings and the fact that nothing changed. Reading message `2747` brings back her full reply, including the offer to edit the file. She can also request surrounding messages, or search the conversation files to find something whose ID she doesn't already have.

## Tool results and reasoning

Clara sees recent tool results, within their output budget. As they get older, we replace the full output with a small record: The tool's name, its result status and a reference to the saved output.

That record is built by code. We don't ask the summarization model to decide whether an action succeeded or what its result ID should be.

When Clara needs an older result, the record tells her how to read it:

```text
Result compacted. Retrieve the original with read_tool_result({"result_id":"message:2747:result:call-1"}).
```

If she needs to check a command's output, she reads the saved output. She doesn't have to run the command again.

**Saved reasoning is omitted from the conversation context**, including for the latest 20 messages. Keeping those messages verbatim means keeping their visible wording. The model can still generate new reasoning for its next response; we just don't keep replaying all its old thinking.

Reasoning also stays out of the text sent to the compactor and the text used for automatic memory retrieval (yet another big topic, I will write a separate post for the memory layer).

## When the chat gets too big

Eventually, even with shorter messages, the conversation will outgrow the context window.

When that happens, **the oldest messages leave Clara's context**. The newer ones stay. The originals remain saved in the chat file, and Clara can read them again with the same tools.

Nothing needs to be summarized into one new account of the whole conversation. We simply keep moving the rolling context-window forward as the chat grows.

Clara's 256k window also has to hold instructions, tools and other context, with room left for the response. With about 13k set aside as a safety margin and 64k reserved for the response, that leaves roughly **180k for input**. These are rounded numbers, and that input space is shared by everything we send, not just the conversation.

There's also a persistent conversation note. Clara can use it to keep track of decisions, useful references and things we're still working on. That note is loaded separately from the message history, so it stays available when older messages leave the window.

So the chat can keep growing on disk while the model works with a smaller part of it. The note helps with continuity, the retained messages carry the recent discussion, and the tools let her go back further.

## What else Clara sees

The conversation is only part of the request. Clara also gets her identity and instructions (`SOUL`), user context (`USER`), any selected agent or skill instructions, and the tool definitions available for the current step.

After the history, Clara gets the current TODOs, calendar entries, presence and conversation note. We refresh this information for each model call.

Relevant saved memories can also appear beside the messages that bring them up. This is where embeddings come in: We embed the original visible wording of a message and use it to search the saved memory collection. Selected memories are then included as text in Clara's context.

The compactor itself doesn't use embeddings. Reading an original message by its ID doesn't need them either. Embeddings support memory recall, which deserves its own post. For this one, the important part is that Clara sees the retrieved memory text, not the vectors, and that those memories also take space in the window.

## The tradeoffs

What I gain is a more gradual way of letting details leave the context. The current discussion stays detailed. Older messages become shorter, then eventually leave the window. I can go off on a tangent and come back, and Clara has a way to read what we actually said. That is much closer to how I experience a long conversation myself.

But summaries are still summaries. They lose detail. A model summarizing one message has less context than a model reading the whole exchange, so it can miss why something mattered or what a phrase referred to. Keeping each message separate doesn't solve that.

Having the originals available helps, but Clara still has to recognize when she needs to read them. If she treats a summary as everything she needs to know, she can still get things wrong. The tools give her a way to check; using them is still her decision.

There's a cost too. Individual summaries can keep repetition that a single summary would remove, so they may take up more tokens. Preparing them takes compute, and reading an older message takes another tool call. Background work is still work, even when I'm not waiting for it to finish.

For me, that's a worthwhile trade. I want to keep talking, follow different threads, and return to details without having to explain everything again. Keeping the exchange recognizable, with the original words available when needed, matters more to me than getting the smallest possible summary.

## Seeing what the model received

Clara has a context inspector that shows the last request sent to the model. I can see which messages were included, which were shortened, the tool observations, recalled memories and current state. There's also a preview using the current conversation and settings.

The normal chat interface still shows the full original conversation. The inspector shows the version sent to the model. Having both makes it much easier to understand what Clara had in front of her when she answered.

---

The implementation keeps evolving, I keep polishing and revising the details, but overall it is significantly better than what I have experienced with other agents.

Hope that helps someone!


This post was written by a human, and edited by Clara.