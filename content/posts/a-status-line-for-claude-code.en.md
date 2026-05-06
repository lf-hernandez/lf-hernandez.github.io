---
title: "A Status Line for Claude Code"
date: 2026-05-03T00:00:00-07:00
summary: "How I set up Claude Code's status line to stay on top of context window usage."
tags: ["claude-code", "developer-tooling", "ai"]
categories: ["tooling"]
---

I started using Claude Code in earnest around December 2025. I gave in to the hype to see what everyone was talking about.

At first I'd ask it to review existing sources and see if it could find bugs or areas for improvement as well as to discuss architectural decisions. It would do a decent job, at least to my eyes. Often times there would be legitimate suggestions, sprinkled in with some that weren't necessary. All that to say, it was producing valuable output most of the time and I appreciated the fact that it was surfacing things worth a second look.

Over time I used it on side projects to knock out features I'd been putting off. During those longer sessions I'd find myself re-prompting, digging back in, wondering why the output felt off. Around the same time, I started noticing compaction triggering periodically. This is where my stance became more critical and I was frustrated and felt I could not trust it to build things reliably.

It took a conversation with a good friend who works at a company going all in on agentic engineering to name it, the context window. I'd been flying blind, only feeling the consequences when I inevitably hit the limit mid-task. This post will go over one way I improved my claude code workflow to ensure I got better output.

## What is the context window?

At the risk of over-simplifying the matter, let's start by drawing an analogy...

When we think of carrying out a task, say building furniture, there are a few things we are cognizant of in our brains working memory:

* The instructions themselves, filled with step-by-step diagrams on how each piece goes together
* The various nuts and bolts and tools that come packaged and are required to build
* Our basic know-how around the said tools (e.g. when we want to tighten a screw we grab the appropriate screwdriver based on the screw and turn right, inversely, if we want to loosen we turn left, etc.)
* Our previous experiences building similar type of furniture (we are able to draw on those memories and apply patterns)
* If we have a friend or partner helping, we keep track of what we're currently doing in relation to them, and try to partition the work so that we can build it faster
* and so on...

Our working memory is pulling from visual instructions, physical tools, experiences and past memories, and coordinating with others, all this while we are physically building. This is our over-simplified context window. And one could imagine, if we start throwing more inputs, like say unrelated conversations that require us to pull from other sources (say someone walks in and asks if we made a reservation for the big birthday dinner, we have to then think about who they're referring to, what date their birthday falls on, what restaurant, and whether we made the reservation or not, maybe we have to pull out our phone and check our inbox for a confirmation, etc.). As you can imagine, this will probably cause a degradation in our ability to carry out the original task (I have a bookshelf with an upside down base panel to prove this). That is more or less the crux of the problem. When that context window, which has a finite set of tokens it works with, hits a predetermined threshold things start to break down.

 

Ok, now for an official definition:

> The "context window" refers to all the text a language model can reference when generating a response, including the response itself. This is different from the large corpus of data the language model was trained on, and instead represents a "working memory" for the model. A larger context window allows the model to handle more complex and lengthy prompts, but more context isn't automatically better. As token count grows, accuracy and recall degrade, a phenomenon known as *context rot*. This makes curating what's in context just as important as how much space is available.
>
> source: https://platform.claude.com/docs/en/build-with-claude/context-windows


So let's break out a few concepts:

* LLMs can process up to a fixed number of tokens (the context window) when generating a response
* Claude Opus 4.7 has a 200k token context window; Google Gemini 1.5 has 1M tokens, for example
* As the session approaches the limit, quality of the output degrades. In other words, it is not an acute phenomenon, it will actually degrade before it hits the threshold
* Once the context window is full, Claude Code automatically compacts. It summarizes everything up to that point and passes it to a new session as the starting context
* We are relying on compaction to capture everything, which is probably not going to happen, so again we want to be cognizant that even with compaction we may lose some context

I want to pause here and be clear about my aim for this post. The main thing I wanted to get out of this is visibility, with that I could start to measure my usage during any given session and decide how I want to manage the strategy. Whether I should spin up a separate session because the conversation is starting to veer off into a separate enough topic that keeping it in the current session would affect the quality of the response, whether I want to manually compact, whether I want to craft a prompt that would manage token usage better, etc.

So that brings us to the status line.



## What is Claude Code's status line?

> The status line is a customizable bar at the bottom of Claude Code that runs any shell script you configure. It receives JSON session data on stdin and displays whatever your script prints, giving you a persistent, at-a-glance view of context usage, costs, git status, or anything else you want to track. Status lines are useful when you:
>
> - Want to monitor context window usage as you work
> - Need to track session costs
> - Work across multiple sessions and need to distinguish them
> - Want git branch and status always visible
>
> source: https://code.claude.com/docs/en/statusline

## What it helps with

What the status line offers is pretty simple, it shifts managing the context window from reactive to proactive.

Before I had any visibility, I only noticed the context limit when things were already breaking down. The output felt less precise. I'd catch myself re-prompting more than I thought I should. Suggestions would miss the mark to the point where I was left scratching my head. By then you're already past the window where it's easy to do anything about it.

With the status line you get a read at a glance. A few things that actually helped:

- **Budgeting before big tasks**: before kicking off a large refactor or asking Claude to work across multiple files, a quick look tells me whether I'm starting at 10% used or 75% used. Those are completely different situations and it changes how I approach what's next.
- **A signal before the cliff**: quality degrades before you hit the limit, not at it. The color thresholds give you a nudge to wrap up or compact before things actually start to break down.
- **Model confirmation**: I switch between Sonnet and Opus depending on what I'm working on. The model name in the status line means I can verify what's actually running instead of assuming.
- **An understanding of cost**: per-session cost is rarely a decision driver in the moment, but seeing it accumulate over time builds a sense of what different types of work actually cost. That's the kind of thing that's hard to develop any other way (data ftw).

The thresholds I settled on, pulled from the docs and other people's setups:

- **Green (< 70%)**: plenty of room, start anything
- **Yellow (70–89%)**: finish what you're on, hold off on kicking off new multi-file work
- **Red (≥ 90%)**: wrap up, run `/compact`, or open a fresh session

## The setup and choices

The script lives at `~/.claude/statusline.sh`. Claude Code runs it on every render, passes JSON session data to stdin, and displays whatever the script prints.

Here's what mine shows:

```
felipe@ubuntu  trinity  [claude-sonnet-4-5]  ctx [██░░░░░░░░] 12k/200k (6%)  $0.003
```

Breaking each piece down:

1. **user@host + directory**: mirrors my terminal prompt, I like to know where I'm at. It's a nice touch to be able to stay visually consistent between my terminal and a Claude session
2. **[Model]**: which model is currently running. I tend to switch often so it's nice to know and verify which model is doing what
3. **ctx bar + ratio + %**: the bar gives a quick visual read, the raw count (12k/200k) helps when sizing up headroom, and the percentage is what maps to the thresholds
4. **$cost**: per-session running total

And the script itself (with the help of claude of course):

```bash
#!/bin/bash
input=$(cat)

if ! command -v jq >/dev/null 2>&1; then
  printf '%s@%s | jq not found\n' "$(whoami)" "$(hostname -s)"
  exit 0
fi

MODEL=$(echo "$input" | jq -r '.model.display_name // "unknown"')
DIR=$(echo "$input"   | jq -r '.workspace.current_dir // ""')
PCT=$(echo "$input"   | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
MAX=$(echo "$input"   | jq -r '.context_window.context_window_size // 200000')
COST=$(echo "$input"  | jq -r '.cost.total_cost_usd // 0')

USER_NAME=$(whoami)
HOST_NAME=$(hostname -s)
SHORT_DIR="${DIR/#$HOME/~}"
SHORT_DIR="${SHORT_DIR##*/}"

PCT=${PCT:-0}; PCT=${PCT//[^0-9]/}; PCT=${PCT:-0}
MAX=${MAX:-200000}; MAX=${MAX//[^0-9]/}; MAX=${MAX:-200000}

USED_TOK=$(( PCT * MAX / 100 ))
fmt_k() { awk -v n="$1" 'BEGIN{ if(n>=1000) printf "%.0fk", n/1000; else printf "%d", n }'; }
CTX="$(fmt_k $USED_TOK)/$(fmt_k $MAX)"

BAR_WIDTH=10
FILLED=$(( (PCT * BAR_WIDTH + 50) / 100 ))
EMPTY=$(( BAR_WIDTH - FILLED ))
BAR=""
i=0; while [ $i -lt $FILLED ]; do BAR="${BAR}█"; i=$((i+1)); done
i=0; while [ $i -lt $EMPTY  ]; do BAR="${BAR}░"; i=$((i+1)); done

if   [ "$PCT" -ge 90 ]; then C='\033[31m'
elif [ "$PCT" -ge 70 ]; then C='\033[33m'
else                         C='\033[32m'
fi
B='\033[1m'; D='\033[2m'; R='\033[0m'

printf "%b%s@%s%b %b%s%b  %b[%s]%b  %bctx [%s] %s (%s%%)%b  %b\$%.3f%b\n" \
  "$D" "$USER_NAME" "$HOST_NAME" "$R" \
  "$B" "$SHORT_DIR" "$R" \
  "$D" "$MODEL" "$R" \
  "$C$B" "$BAR" "$CTX" "$PCT" "$R" \
  "$D" "$COST" "$R"
```

## Context engineering habits worth building

Having the number is only useful if you actually do something with it.

**Compact at natural boundaries, not mid-task.** `/compact` works best when there's a clean summary to be made. For example, right after finishing a feature or before switching to a different problem. Triggering it mid-debugging means you risk losing the thread on something you haven't resolved yet. Common sense but worth noting.

**Check `/context` when the percentage surprises you.** If you glance down and see you're at 65% after what felt like a short session, `/context` gives you a full token breakdown. Useful for figuring out what's actually eating the budget (long file reads, verbose outputs, pasted content, etc)

**Invest in `CLAUDE.md`.** It gets loaded at the start of every session, so when you hit the limit and need to start fresh or compaction kicks in, Claude already knows your project. You're not re-explaining everything from scratch. Survives restarts, survives compaction. Worth the upfront time.

**Don't kick off complex multi-file work past 80%.** You'll hit the limit mid-task, which is a worse situation than pausing to compact first. If you're in the yellow zone and about to start something that touches a lot of files, just compact and start clean.
