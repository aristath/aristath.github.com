---
layout: post
title: "My AI stack"
permalink: "blog/my-ai-stack"
categories:
  - AI
  - Hardware
---

Wow, I haven't written a post here in 6 years... I think it's time I start again. :D

I was using Claude, ChatGPT, Cursor, and others... All of them at once. Each of them has its pros and cons, each of them is best at something, and they all have some gaps that another one fills beautifully. However, the bills just started piling up. And it's not like I could just cancel one of them because I kept hitting the limits of the max plans on all of them. At some point, it became clear that paying "rent" for AI was unsustainable long-term. So an investment was made to become somewhat independent. 

The initial (**substantial**) funding for this journey was from my employer, [emilia.capital](https://emilia.capital), and then I continued by expanding the build with more compute over the course of months.

---

## The Hardware

* CPU: AMD Ryzen 9950X
* RAM: 64GB (2x32GB)
* Motherboard: Asrock X870e-Taichi
* GPUs: 4x Radeon AI PRO R9700 (32GB each) + 1x RTX3090 (24GB) for a total of 152GB VRAM
* NVMe: 4TB for storage and models + 1TB for the system.

I started with 2x R9700 GPUs. Over the course of the following months I managed to get 2 more R9700 GPUs (one every couple of months) and a used NVIDIA RTX3090. 

Of course they can't all connect to a motherboard with just 2 PCIe ports, so the 4 AMD GPUs are now connected to a bifurcation card (R34A), and the RTX3090 is connected to an M.2 slot via an M.2-to-OcuLink-to-PCIe adapter.

Future expansion: I can still connect 2 more GPUs: One on the remaining M.2 slot, and another one via USB4. I already have the adapters for these, just not the GPUs yet - looking on Vinted daily for opportunities.


The reason I went with AMD and not NVIDIA is twofold:

1. Price. The obvious one! An NVIDIA RTX 5090 cost ~€5-6k for 32GB of VRAM. An AMD R9700 cost somewhere between a 3rd and a 4th of that, for the same amount of VRAM. The RTX3090 I got was used from Vinted and I got it for €800.
2. Power consumption. The AMD GPUs are simply not as power-hungry as NVIDIAs.

Would it be better to go for NVIDIA GPUs instead of AMD? Well... I would definitely have a faster system, **but** I would pay for it many times over in the electric bill, and performance is not the only metric that matters. There are lots of ethical concerns in the middle too. 

Pros: Lots of VRAM.

Cons: vLLM is basically pointless on this setup. PCIe speed has dropped significantly due to the bifurcation and all the adapters, which means that tensor-splitting a model is painful. My router (explained below) _can_ run vLLM/safetensors backends but due to the way my hardware evolved I don't bother (not to mention that ROCm made running vLLM a PITA). Instead, I'm using llama.cpp with Vulkan (not Cuda for the NVIDIA + ROCm for the AMDs, that was painfully slow).

---

## The Software

This is where things get a little more interesting. Had to build a couple of projects in order to accommodate my personal workflow.

### [ergon.studio](https://github.com/aristath/ergon.studio)

([Link to GitHub repo](https://github.com/aristath/ergon.studio))

This is a plugin for OpenCode which does the following:

#### Agents

The plugin adds multiple agents and sub-agents. Some of them are public-facing, others are internal only.

The way I usually work is this:

- I use the [`scout`](https://github.com/aristath/ergon.studio/blob/main/agents/scout.md) agent for discussion, brainstorming and [planning](https://github.com/aristath/ergon.studio/blob/main/agents/scout.md#planning-mode). Tried to make it think a bit like I do when planning, so it's a multi-step process. Kinda crude, but it works. 
- Once a plan is finalized, I switch to the [`orchestrator`](https://github.com/aristath/ergon.studio/blob/main/agents/orchestrator.md) agent which handles the actual implementation and coding. The orchestrator calls multiple sub-agents as needed, and also implements a best-of-N flow when needed.

These are the main agents, the others are mostly internal: 

a team of specialists (architect, coder, reviewer, tester, critic, researcher, design-reviewer, quality-controller) that the orchestrator delegates to. The one I'd single out is the quality-controller. After any coding task, the orchestrator **has to** run it - it bundles the reviewer, the design-reviewer and a completion checklist and returns either APPROVED or REJECTED. The orchestrator can't skip it, and after 3 rejections it stops and asks me what to do.

#### Memory Steward

Memory usually fails in coding harnesses because the agent needs to judge when to call memories, when to save a memory etc. 
It's a fragile thing. 

The memory-steward does something slightly different: It has a small model running at all times (currently it's a `qwen3.5-4b` model but that's easy to change from the [markdown file of the steward](https://github.com/aristath/ergon.studio/blob/main/prompts/steward.md)). The key idea is that the main coding model never has to decide anything about memory - the steward handles both sides, retrieving _and_ saving.

What basically happens is this: When I send a message to OpenCode, it doesn't go directly to OpenCode... It goes to the memory-steward service first. The steward rewrites my message into a tight search query, looks up relevant memories (via a memory MCP - currently OpenMemory, but it's easy to switch that), and injects whatever it finds into that turn's context before the main model ever sees the message. This happens on every single turn.

The rewrite step is more important than it sounds. Memory lookup is an embedding search, and embedding search is surprisingly sensitive to phrasing - two messages with the exact same intent but worded differently will land in different places. If I search with my raw, rambling message, I get worse hits than if I search with a clean 3-8 word query. So the steward's first job is to strip my message down to what I'm actually looking for, _then_ search.

Then, after the exchange is done, the steward looks at it again and makes a separate judgement call on whether anything in it is worth saving. If so, it writes it back to the memory MCP. So retrieval runs constantly in the background, and saving is a deliberate, judged decision - and neither one depends on the main model remembering to ask.

#### Scratchpad

Local models that can run on my hardware don't have 1M context, so I need to account for that - as well as preferences persistence on a per-project basis.

The scratchpad is a [simple skill](https://github.com/aristath/ergon.studio/blob/main/skills/scratchpad/SKILL.md) that does exactly what you'd expect: It's a scratchpad, allowing the LLM to write to a file things we've discussed so that they stay pinned. It's also used for internal notes of the LLM itself. Scratchpad notes are appended to the system prompt so the model can always see them.

When OpenCode trims the conversation to stay under the context limit, the scratchpad gets re-injected, so the things I pinned survive the compaction even though the rest of the conversation got summarized away. That's the whole point - it's the bit of context that _doesn't_ get thrown out.

#### Handoff

This is just a handoff document that allows me to switch between agents and persist long sessions. 

Think of it like the plan documents that Claude creates. Same concept, different implementation. 

This is just [another skill file](https://github.com/aristath/ergon.studio/blob/main/skills/handoff/SKILL.md). When I make a plan with the `scout` agent, it writes the handoff document. When I then switch to the `orchestrator` agent, it reads that file and starts the implementation.

---

### [Inference-router](https://github.com/aristath/inference-router)

([Link to GitHub repo](https://github.com/aristath/inference-router))

Too many models, too many configurations, it's hard to keep track of everything. I needed to be able to easily load multiple models in VRAM, switch between them easily, be able to edit the configs from a web-UI and so on. This project does something similar to what llama-router does, but more tailored to my own flows. It is a fully transparent proxy, and basically instead of pointing my OpenCode config to llama.cpp directly, I point it to this project instead.

- **Web UI** to manage the models
- **Allow setting aliases for models**. This way I can have a "coder" or "brainstormer" alias, and just assign it to a model that is already in my config. Small but important convenience because it allows me to just define 4-5 "models" in the OpenCode config, and then manage how they are assigned from the web-UI. The alternative would be to have 20-30 models in the OpenCode config and navigate those. Tried it for a few months - not convenient. This allows me to be more flexible without breaking anything.
- **Infinite loops recovery**. This one is important and saved me lots of frustration. Some models are amazing (`qwen3.6-27b` for example), but occasionally they fall into infinite thinking loops or infinitely looping tool calls. The inference-router handles both, with two separate guards. Thinking loops are caught by a streaming detector that watches the output for repeating text (a tandem-repeat detector running on the token stream); when it trips, it replays the partial output and injects a corrective prompt, then retries. Looping tool calls are caught by a separate cross-turn guard that watches for a repeating `tool_call → result → tool_call` pattern across messages. The injected nudge isn't subtle, and it escalates if the model keeps looping - the first interrupt asks it to step back and identify the real next action, and by the third it's down to "emit only the next required action, no thinking, no commentary".
- **Speculative decoding**. The router supports both MTP draft-heads and external draft models (plus context checkpoints for the hybrid-recurrent Qwen3.5 models), to improve performance for the models I use most.
- **Automatic VRAM management**. It's not just "load a model" - it reads the GGUF metadata, estimates how much VRAM the model needs, and evicts idle models under pressure (scored by how long they've been idle and how big they are). It also auto-generates the `--tensor-split` to greedily fill across the GPUs, so I don't have to hand-tune placement every time.
- **Mixed-vendor GPUs**. My rig is 4x AMD + 1x NVIDIA, and the router enumerates all of them - AMD via Vulkan, NVIDIA via `nvidia-smi`, and even Intel Arc if I had one plugged in (I have one but it's on a different dedicated assistant device - so I know it works). The same physical card can be addressed as either a CUDA device, a ROCm device, a Battlemage device, or a Vulkan device depending on what I want a model to use. Some models run better on ROCm in AMD GPUs, others (most) run better on Vulkan, CUDA makes the GPU fans go nuts while Vulkan is more quiet and so on... So I can define the backend on a per-model basis.
- **Binary presets**. I rebuild llama.cpp constantly (new commits, new flags, experimental branches). Instead of editing the binary path on every single model whenever I rebuild, I define the binary once as a "preset" and point models at it. Rebuild, update the preset, and every model picks up the new build automatically.

---

### [orchestrator](https://github.com/aristath/orchestrator) - the one that started it all

([Link to GitHub repo](https://github.com/aristath/orchestrator))

Before any of the above existed, there was this. It's worth mentioning because it's where the whole journey started, and because the idea behind it is still useful on its own if you're not ready to go fully local yet.

Back when I was still paying for Claude, ChatGPT and the rest, the thing that seemed insane was paying frontier-model prices for grunt work. Generating boilerplate, applying a review's feedback, writing tests, reshaping a JSON config, or simple repetitive tasks... none of that needs a frontier model. Nobody needs a frontier model for simple stuff.

So the idea was simple: **keep the expensive model as the senior engineer who thinks, decides and verifies - and delegate the mechanical work to local models, at zero marginal cost.**

In practice it's an MCP server. You register it with Claude Code (or any MCP-compatible agent) and it exposes the local models as a set of tools: `generate_code`, `edit_code`, `improve_code`, `simplify_code`, `generate_tests`, `review_code`, `plan_task`, `transform_json`. Claude stops writing code itself and instead orchestrates: it plans the task, hands the actual typing to a local coder model, sends the result to a local reviewer model, feeds the review back for a fix, and only then verifies the result itself. The frontier model goes from writing every line to making decisions about lines other models wrote - which is a lot fewer tokens, and a much smaller bill.

It worked well enough that the obvious next question was: if the local models are doing all the real work anyway, do I even need the frontier model in the loop? That question is what led to [ergon.studio](#ergonstudio) and the fully-local setup. So orchestrator was a stepping stone - but a genuinely useful one, and if you've got some local compute lying idle and a Claude bill you'd like to shrink, it's a low-effort way to do exactly that.

IMPORTANT NOTES: 
1. I haven't used that project in quite some time. Some things are hardcoded (like for example the models slugs etc), so you'll probably need to edit a couple of things to make it suit your needs.
2. Claude, ChatGPT etc have to make a "conscious" decision to actually use this MCP. Most times they don't. So if you use this project (or a concept similar to this project), you should also add a couple of lines in the `CLAUDE.md`/`AGENTS.md` etc files to **force** them to delegate tasks.

---

## Can local, open-source models really do what frontier models can?

Well... Yes and no.

Frontier models are genuinely amazing. An open-source model with a few billion parameters cannot compare to a frontier model that is (probably) trillions of params!! However, the harness is sometimes more important than the model itself. Think of it this way: You have a brilliant person on one hand, and an average one on the other hand. You can tell to the brilliant person "build this", and they will. They will make lots of assumptions along the way, lots of things that you probably won't like and won't catch. The end result is usually great and they work FAST! And then you have the average person. The one that needs hand-holding. That's what the harness does... it's the handholding part. That's what the [`scout`](https://github.com/aristath/ergon.studio/blob/main/agents/scout.md) agent for example does in ergon.studio when I want to make a plan! It breaks the process down into steps:
1. Imagine the optimal solution without being tied to the current codebase
2. Strip it down, remove all the inherent over-engineering fluff that LLMs usually add
3. Compare to the current codebase, find how we can achieve the "ideal" we imagined, what needs to happen to the codebase
4. Start making a high-level plan. Start from the goal and work backwards. No specifics, just the overarching architectural decisions etc.
5. Iterative zoom-in. Start from the abstract plan and zoom-in, one step at a time, making it more specific
6. Find the friction points, risks, uncertainties, etc. Define them, find solutions to issues BEFORE they become a problem.
7. Make the plan
8. Assume you're wrong. Review the plan, look at things more holistically, and iterate.

These are steps that I specifically defined, in order to make the average person produce code that is on par with a frontier model.

Does it work? Yes, it does. However, you need to adjust your workflow significantly. With a frontier model you can build projects with one-shot prompts. You give them a goal and they start working towards that goal. With local models you can't do that efficiently. I mean sure you can, but not if you want genuinely good results. So my current workflow is not "one-shot" builds. I am the senior engineer, the LLM is the junior. I know what I want to accomplish, I have the architecture in my head, the LLM helps write the code one step at a time. I don't make a grand plan, I make smaller plans, one at a time. The grand plan lives in my head, the LLM gets parts of it. So I have for example 3 terminals open:
1. An OpenCode session where I have a high-level discussion with the LLM about the overall architecture and what I want to achieve. This is using the scout agent from ergon.studio. No touching code, just a freeform discussion.
2. An OpenCode session where I'm working on the current task (running the orchestrator agent from ergon.studio) - this one does the actual coding on the current task. The orchestrator reads the HANDOFF.md file that is already on-disk, and works on it.
3. An OpenCode session where I'm planning the next step in the grand plan. This is again using the scout agent from ergon.studio. Discussing, finding the best solution to what we want to achieve, then once session #2 finishes we write the plan to the HANDOFF.md file, session #2 starts working on it, I reset session #3 and start discussing the next step in the grand plan.

**So yes, local models can actually accomplish what a frontier model does - just not using the same mental model or process.**

Keep in mind that local models get released quite often... this is what makes this really exciting for the future! 

The limitations I faced and all the things I needed to do to get around them are because of the current state of OSS LLMs. It was different 6 months ago, and it will be different 6 months from now. Friction keeps reducing, and you can think of OSS LLMs as the frontier models from 6-12 months ago. We were doing lots of things last year with Claude, right? Since then Claude has moved on... But we can get similar intelligence to last year's Claude with Qwen3.6-27b and others today - or whatever you can run locally. Want to use Claude Opus 4.8? Sorry, can't do unless you have the budget for half a TB VRAM. Then you could probably run HUGE local LLMs that are really frontier and on par with Opus.

---

That's all for this post, hope it helps someone. When I started this journey most of this stuff did not exist, nowadays you can probably find multiple similar alternatives to what I did - but regardless, it was worth documenting these.

This post was written by a human.
