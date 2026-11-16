+++
title = "Building a Personal Reading System"
date = 2026-11-16
draft = false
description = "Fetch and score once at 08:50, keep the list in Notion, then turn the English I read into expressions I can use. A reading pipeline where each layer does one job."
tags = ["reading", "automation", "Horizon", "Notion", "AI"]
+++

I built myself a daily reading system on top of the open-source project Horizon. The problem it solves is simple: good articles are scattered everywhere and attention is limited. I want to see, each day, the few pieces worth reading carefully — and worth rereading later — and then add them to my own list.

<!--more-->

Every day at around 08:50 (Asia/Shanghai), GitHub Actions runs Horizon once: it pulls articles from RSS and other sources, scores and filters them with DeepSeek, generates that day's digest, and publishes it to [horizon.fullstackjam.com](https://horizon.fullstackjam.com). This is the only time collection runs. The public site is for browsing; the repository is [fullstackjam/horizon](https://github.com/fullstackjam/horizon).

Once the digest is generated, the reading assistant inside Grok Bot reads it and adds those articles to my Notion reading list, reusing the scores Horizon has already produced rather than rescoring everything. The list in Notion doesn't come only from this digest — I add articles by hand too. Those entries usually have no score, so around 10:00 the assistant uses a local Horizon MCP to fill in only the empty Score fields.

English-language articles newly added to the list go to another Grok Bot, English Accumulation. It pulls out expressions worth learning, records them in a phrase book, and uses spaced repetition to help me put them to use. Sentences I come across myself — in videos, blogs, or chats — can be sent to it directly.

The whole chain has four layers: GitHub handles the daily collection and digest, Cloudflare hosts the public reading list site, Grok Bot handles writing, scoring, and English accumulation, and Notion stores my reading list.

{{ mermaid(body="
flowchart TD
  subgraph github [GitHub]
    A[RSS and other sources] --> B[Actions: runs Horizon once a day around 08:50]
    B --> C[Daily digest]
    B --> D[Publish to gh-pages in fullstackjam/horizon]
  end

  subgraph cloudflare [Cloudflare]
    D --> E[horizon.fullstackjam.com]
  end

  subgraph grok [Grok Bot]
    C --> F[Reading assistant reads the digest]
    E --> F
    F --> G[Write to the Notion reading list]
    F --> H[Hand new English originals to English Accumulation]
    I[Articles I add by hand] --> G
    G -.->|around 10:00, if Score is missing| J[Local Horizon MCP scores a single article]
    J -.-> G
    K[Sentences, screenshots, videos, or blogs I send] --> H
    H --> L[Extract useful expressions into a phrase book]
    L --> M[Spaced repetition]
  end
") }}

For me, the key is that each layer does one job: Horizon finds and filters, Notion keeps things for the long term, and English Accumulation turns the English I've read into expressions I can actually use. Collection never runs twice, and no article in the same digest gets scored twice.
