+++
title = "From Editor Wars to Model Wars: Reflections on AI-Assisted Programming"
date = 2025-09-24
description = "Honest thoughts after months of using Cursor at $200/month: the productivity boost is real, and so is the career anxiety. What changes when AI writes better code than you do?"
tags = ["AI", "programming tools", "Cursor", "career", "productivity"]

[extra.comments]
issue_id = 11
[[extra.faq]]
question = "Is Cursor worth $200 per month?"
answer = "Depends how you use it. As a fancier autocomplete, probably not. But if you bring it into architecture discussions, requirements analysis, and code review, the time it saves is real. The real question is whether that saved time creates enough value to cover the cost."

[[extra.faq]]
question = "Will AI coding tools replace programmers?"
answer = "The 'translate requirements into code' kind of work is genuinely at risk. But understanding what users actually need, making architecture decisions, and judging tradeoffs — AI can't do those yet. The role is changing, not disappearing."

[[extra.faq]]
question = "Do I still need to learn fundamentals if AI writes my code?"
answer = "Yes. You have to read AI-generated code, spot mistakes, and know when it's wrong. Without solid fundamentals you can't even tell when AI is confidently handing you nonsense. Fundamentals are what let you steer it."

[[extra.faq]]
question = "Is switching from VSCode to Cursor difficult?"
answer = "Barely. Cursor is a VSCode fork, so keybindings, extensions, and settings all carry over. The real learning curve is building the habit of talking to the AI, not the editor itself."

[[extra.faq]]
question = "How should programmers adapt to the AI coding era?"
answer = "Shift from 'person who writes code' to 'product engineer.' Lean harder on product thinking and architecture, keep your coding fundamentals sharp enough to catch AI's mistakes, and pick up prompting along the way. The core shift is from 'how to build it' to 'what to build and why.'"
+++

I got another Cursor Pro bill yesterday. Twenty bucks.

I did the math — close to $200 on Cursor this month alone. It stings a little, but I don't regret it. It doesn't feel like paying for software. It feels like buying power-ups in a game you can't stop playing.

A few months in, my feelings are mixed. It's genuinely changed how I work. It's also left me with this nagging sense that the ground is moving under me. I think every programmer my age is going to have to deal with this one way or another.

<!--more-->

## A brief history of programmer arguments

Programmers love to argue. This is not news.

Ten years ago it was Vim vs. Emacs. Then Sublime vs. Atom. Then VSCode won and everyone settled down and we could get back to actually writing code.

Then 2024 kicked off a new one: the model wars.

The question stopped being "what editor do you use?" and became "what model do you use?" GPT-4 or Claude? Cursor or Copilot? The old editor wars were about identity — you might lose face, but never your livelihood. The model wars are tied straight to your output. The stakes are a lot higher.

## The good part: it really does change things

When I first heard about Cursor I was skeptical. I'd been on VSCode for years. Every shortcut, every extension, every config was muscle memory. Who migrates editors for a fancy autocomplete?

Then I actually used it for a while, and I couldn't go back.

The most immediate change was that coding stopped being a solo thing. Before, when I hit an unfamiliar library or some gnarly logic, I'd burn time on Google, grind through docs, or scroll Stack Overflow hoping a 2019 answer still applied. Now I just ask Cursor. Most of the time it gives me something solid. When it doesn't, I push back and it usually corrects itself. It's a completely different experience from digging through docs.

The speed is something you feel. Last month I built a small tool for a friend — idea to working thing in two evenings. A year ago, just the scaffolding and boilerplate would've eaten several days.

And it's not only about typing faster. Projects I used to write off as "too complex" or "too steep to bother" are suddenly within reach. Machine learning, blockchain contracts, mobile dev — stuff I wouldn't have attempted, I can at least prototype now. The cost of trying something new dropped a lot.

The other thing is it doesn't get tired. Cursor is like a technical partner who's around at any hour. Sure, it occasionally invents an API that doesn't exist, with total confidence. But most of the time it's actually useful, and it's just as game at 3 AM as at 3 PM. For a solo developer, that "always there" feeling is worth a lot.

## The anxiety: the rules changed

Let me be honest about the other side, because it's real too.

The blunt version: writing code is losing market value. I used to think my edge was typing fast, shipping clean code, and killing bugs efficiently. Those felt like core skills. AI does all of that faster now, and it never gets tired. Worse, it's not just a code generator anymore — it reads requirements, reasons about architecture, debugs. The "translate requirements into code" job is genuinely exposed.

People around me feel it. A friend does frontend at a small shop and he's worried. His boss has been building simple pages with AI tools and the results are decent — still needs a human to polish, but the direction is obvious. That's not fearmongering. Simple CRUD, static pages, form validation, the entry-level stuff — it's getting chipped away. Not gone tomorrow, but the trend is clear.

And what users want changed too. People used to care about a pretty interface and smooth interactions. Now they mostly care whether the thing is useful, whether it actually solves their problem. Look at how people pick AI tools — nobody cares how nice your UI is if the model behind it is weak. GPT-4 works well? I'll use ChatGPT. Claude's better? I switch. Cursor delivers? I'm on Cursor. Zero brand loyalty. So the moat of "clean engineering execution" and "polished UI" is getting shallower, and the real moat moves to "do you have a good idea" and "can you solve a real problem."

## Where I'd put my effort now

If the rules changed, where you spend your energy should change too.

Coding ability still matters, but the job shifts toward reading what the AI wrote, catching what's wrong, and fixing it. You need the fundamentals — if you can't tell when AI is wrong, you're in trouble. On top of that I'd lean much harder into product thinking: knowing what users actually need and which features matter, which still takes real domain knowledge. Architecture stays human too — AI can fill in the details, but how to split the system, how data flows, where to make tradeoffs is on you. Prompting counts for more than people admit; the same request, prompted well, gets you dramatically better code, and the skill is still evolving. And someone has to care about how things feel, because AI-generated interfaces tend to be ugly and tone-deaf to users.

## From "coder" to "product engineer"

The pure "coder" role is in danger. The "product engineer" role is more valuable than ever.

A product engineer understands both the tech and the product — can spot a real user need, design a sane technical solution, and validate it fast. AI made that role more valuable because the distance from idea to product collapsed. What took weeks of prototyping now takes days, sometimes hours. Whoever validates faster and iterates sooner wins.

It's also why I think this is a great moment for indie developers. One person plus AI can cover what used to need a small team. The catch is you have to be a generalist who can handle a bit of everything, not a specialist who only knows one language.

## What I've actually learned

I'm still figuring this out, but a few things have stuck.

Don't drop the fundamentals. AI is an amplifier, and to amplify something you need something worth amplifying. CS basics, algorithms, system design — they're what let you catch AI's mistakes and guide it toward better code.

Treat yourself as AI's manager. Coding with AI is like managing a brilliant but occasionally confused junior engineer: assign the task, check the work, pull it back when it drifts.

Stay a student. Today Cursor's on top; tomorrow it might be something else. Don't bet on a specific tool, bet on your ability to learn.

And build more, worry less. Instead of debating whether AI will replace you, go build something with it — you'll learn what it can and can't do by doing, not theorizing. Anxiety doesn't ship anything.

## Final thoughts

AI changed the rules. Creativity and judgment are still ours, at least for now. The point isn't to race AI at writing code — that's a race you lose. It's to find where you fit in the new setup and use AI as a tool rather than getting replaced by it.

My strategy is simple: keep paying for the tools, keep learning, keep building. That $200 a month? I'm calling it tuition.
