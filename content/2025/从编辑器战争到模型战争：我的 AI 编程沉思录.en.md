+++
title = "From Editor Wars to Model Wars: Reflections on AI-Assisted Programming"
date = 2025-09-24
description = "Honest thoughts after months of using Cursor at $200/month: the productivity boost is real, and so is the career anxiety. What changes when AI writes better code than you do?"
tags = ["AI", "programming tools", "Cursor", "career", "productivity"]

[extra.comments]
issue_id = 9

[[extra.faq]]
question = "Is Cursor worth $200 per month?"
answer = "It depends on how you use it. If you treat it as a glorified autocomplete, probably not. But if you involve it in architecture discussions, requirements analysis, and code review, the productivity gain is substantial. The real question is whether the time you save creates enough value to justify the cost."

[[extra.faq]]
question = "Will AI coding tools replace programmers?"
answer = "The 'translate requirements into code' kind of work is genuinely at risk. But understanding what users actually need, making architecture decisions, and exercising judgment about tradeoffs — AI can't do those yet. The role is changing, but it's not disappearing."

[[extra.faq]]
question = "Do I still need to learn fundamentals if AI writes my code?"
answer = "Absolutely. You need to read AI-generated code, spot mistakes, and know when it's wrong. Without solid fundamentals, you can't even tell when AI is confidently giving you nonsense. Fundamentals are the prerequisite for steering AI effectively."

[[extra.faq]]
question = "Is switching from VSCode to Cursor difficult?"
answer = "Nearly seamless. Cursor is a VSCode fork, so keybindings, extensions, and settings all carry over. The real learning curve is building the habit of conversing with AI, not the tool itself."

[[extra.faq]]
question = "How should programmers adapt to the AI coding era?"
answer = "Shift from 'person who writes code' to 'product engineer.' Rebalance your skills: 30% coding ability, 25% product thinking, 25% architecture, 10% prompt craft, 10% design sense. The core shift is from 'how to build it' to 'what to build and why.'"
+++

I got another Cursor Pro bill yesterday. Twenty bucks.

I did the math — I've spent close to $200 on Cursor this month alone. It stings a little, but I don't regret it. It doesn't feel like paying for software. It feels like buying power-ups in a game you can't stop playing.

After a few months with Cursor, my feelings are complicated. On one hand, it's genuinely transformative. On the other, there's this nagging sense that the ground is shifting under my feet. I think every programmer of our generation is going to have to reckon with this.

<!--more-->

---

## A Brief History of Programmer Arguments

Programmers love to argue. This is not news.

Ten years ago, it was Vim vs. Emacs. Then Sublime vs. Atom. Then VSCode won and everyone finally settled down. We could actually focus on writing code.

But starting in 2024, a new war began — the **model wars**.

The question is no longer "what editor do you use?" It's "what model do you use?" GPT-4 or Claude? Cursor or Copilot? The old editor wars were about identity — you might lose face, but not your livelihood. The model wars are directly tied to your productivity. The stakes are orders of magnitude higher.

---

## The Good: AI Coding Is Genuinely Transformative

When I first heard about Cursor, I was skeptical. I'd been on VSCode for years. Every shortcut, every extension, every config was muscle memory. Who wants to migrate for a fancy autocomplete?

But after actually using it for a while, I couldn't go back.

### You're Not Alone Anymore

The most immediate change: **coding stopped being a solo activity**. Before, if I hit an unfamiliar library or a tricky piece of logic, I'd either spend ages on Google, or grit my teeth through documentation, or scroll through Stack Overflow hoping to find an answer from 2019 that was still relevant.

Now I just ask Cursor. Most of the time, it gives me a solid answer. When it doesn't, I push back, and it usually corrects itself. It's a completely different experience from digging through docs.

### Speed You Can Feel

Last month I built a small tool for a friend. Idea to working product — two evenings. A year ago, just setting up the scaffolding and writing boilerplate would've taken several days.

It's not just about writing faster. Projects I used to dismiss as "too complex" or "too steep a learning curve" are suddenly approachable. Machine learning, blockchain contracts, mobile development — things I wouldn't have even attempted before, I can now at least prototype. AI flattened the learning curve dramatically.

### A Tireless Partner

Cursor is like having a technical partner who's available 24/7. Sure, it occasionally hallucinates an API that doesn't exist — with absolute confidence, no less. But most of the time, it's genuinely helpful. It doesn't get tired, doesn't get annoyed, and it's just as enthusiastic at 3 AM as at 3 PM.

For solo developers, that "always available" feeling is priceless.

---

## The Anxiety: The Rules Changed

But let's be honest — the anxiety is real too.

### "Writing Code" Is Depreciating

Here's the blunt truth: **the skill of writing code is losing market value**. I used to think my edge was typing fast, shipping clean code, and squashing bugs efficiently. Those felt like core competencies. Now AI does all of that faster and without getting tired.

What's scarier is that AI isn't just a code generator anymore. It understands requirements, analyzes architecture, and debugs. The work of "translating requirements into code" — that's genuinely at risk.

### People Around Me Feel It

A friend of mine does frontend at a small company. He's worried. His boss has been experimenting with AI tools to build simple pages, and the results are decent. They still need a human to polish things, but the trend is clear — that kind of work is worth less than it used to be.

This isn't fearmongering. Simple CRUD apps, static pages, form validation — this "entry-level" work is being chipped away. Not gone tomorrow, but the direction is unmistakable.

### Users Don't Care About Your Stack

There's another shift that's hard to ignore: **what users expect from products has changed**.

People used to care about beautiful interfaces and smooth interactions. Now? They care about whether the thing is useful. Whether it actually solves their problem.

Look at how people choose AI tools — nobody cares how pretty your UI is if the model behind it is weak. GPT-4 works well? I'll use ChatGPT. Claude is better? I'll switch to Claude. Cursor delivers? I'm on Cursor. Zero brand loyalty.

What does this mean? The moat of "engineering execution" and "polished UI" is getting shallower. The real moat is now "can you come up with a good idea?" and "can you solve a real problem?"

---

## Respeccing Your Skill Tree

If the rules changed, your skill allocation needs to change too.

Here's how I think the breakdown should look now:

**30% Coding ability** — Mostly about reading AI-generated code, spotting issues, and knowing how to fix them. You still need solid fundamentals. If you can't tell when AI is wrong, you're in trouble.

**25% Product thinking** — Knowing what users actually need and which features matter. AI can't replace this yet — it requires deep domain knowledge and genuine empathy for users.

**25% Architecture** — The big-picture technical decisions still need a human. AI can implement details, but how to split the system, how data flows, where to make tradeoffs — that's still on you.

**10% Prompting skill** — This matters more than most people think. For the same requirement, someone who prompts well gets dramatically better code from AI. It's a brand new skill, and it's evolving fast.

**10% Design sense and UX** — AI-generated interfaces are usually ugly and lack user empathy. Someone still needs to care about how things feel.

---

## From "Coder" to "Product Engineer"

The pure "coder" role is in danger. But the "product engineer" role is more valuable than ever.

What's a product engineer? Someone who understands both the technology and the product. Who can identify real user needs, design sensible technical solutions, and rapidly validate ideas.

AI made this role more valuable because the distance from idea to product just collapsed. What used to take weeks of prototyping now takes days or even hours. Whoever validates faster and iterates sooner wins.

This is also why I think we're in the golden age for indie developers. One person plus AI can do what a small team used to do. The catch: you need to be a generalist who can handle a bit of everything, not a specialist who only knows one language.

---

## Practical Advice

I'm still figuring this out myself, but here's what I've learned so far:

**Don't abandon the fundamentals.** AI is powerful, but computer science basics, algorithms, and system design still matter. Without them, you can't spot AI's mistakes, let alone guide it toward better code. AI is an amplifier — to amplify something, you need something worth amplifying.

**Learn to be AI's product manager.** Using AI to code is like managing a brilliant but occasionally confused junior engineer. You need to know how to assign tasks, check the work, and pull it back on track when it drifts.

**Stay a student.** This industry moves fast. Today Cursor is on top; tomorrow there might be something better. The only constant is change itself. Don't bet on a specific tool — bet on your ability to learn.

**Build more, worry less.** Instead of anxiously debating whether AI will replace you, go build something interesting with AI. You'll learn what it can and can't do through practice, not theory. Anxiety doesn't create value. Shipping does.

---

## Final Thoughts

AI did change the rules of the game. But **creativity and judgment** — those are still human territory. At least for now.

The goal isn't to race AI at writing code — that's a game you'll lose. It's to find your place in this new landscape. Make AI your tool, not your replacement.

For programmers, this is both the best of times and the worst of times. It all depends on how you choose to adapt.

My strategy is simple: keep paying for the tools, keep learning, keep building. That $200 a month? Consider it tuition.
