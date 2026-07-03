+++
title = "Git's First Commit: Five Designs That Held Up for Twenty Years"
date = 2026-06-26
description = "Git's first commit: 11 files, 1,244 lines. Several design decisions in there have barely moved in twenty years. A close look at what makes them good, and what changed around them."
tags = ["Git", "system-design", "source-code-reading", "content-addressing", "open-source"]

[extra.comments]
issue_id = 9
+++

I've used Git for years without really understanding it. A while back, on a whim, I checked out Git's own repository at its very first commit:

```
commit e83c5163316f89bfbde7d9ab23ca2e25604af290
Author: Linus Torvalds <torvalds@ppc970.osdl.org>
Date:   Thu Apr 7 15:13:13 2005 -0700

    Initial revision of "git", the information manager from hell

 11 files changed, 1244 insertions(+)
```

1,244 lines, 11 files. Reading them surprised me: several design decisions in there have barely changed in twenty years, and they're still running inside the Git I use every day.

<!--more-->

Let me start with the one that surprised me most.

I always assumed the staging area was something Git bolted on later. It's the concept new users trip over the most, and it looks exactly like accumulated complexity. Turns out it was there on day one, as one of the two pillars of the whole system. The README opens with it: Git has exactly two abstractions, an object database and a "current directory cache". The second one is today's staging area.

The file header is even better. The first version gave the index file a magic number, `DIRC`, short for DIRectory Cache:

```c
#define CACHE_SIGNATURE 0x44495243	/* "DIRC" */
```

Twenty years later, hexdump the `.git/index` of any repository on your machine and the first four bytes are still it:

```
$ hexdump -C .git/index | head -1
00000000  44 49 52 43 00 00 00 02  ...  |DIRC....|
```

The format went from version 1 to today's versions 2, 3, and 4. Those four bytes never moved.

I counted at least five designs like this in those 1,244 lines, all basically untouched for twenty years. Worth going through one by one: what makes each good, and what changed around it.

## The Name Is the Content

When you store files, the usual approach is to hand out auto-incrementing IDs and keep a table mapping IDs to paths. Most databases work this way.

Git went the other way. It computes a SHA-1 hash of the file content and uses the hash as the object's name. Content-addressable storage.

```bash
$ printf 'hello' | git hash-object --stdin
b6fc4c620b67d95f953a5c1c1230aaab5db5a1b0
```

The content is `hello`, so the object is called `b6fc4c…` and lives at `.git/objects/b6/fc4c…`. The name isn't assigned. It's computed from the content.

This buys a lot at once. Identical content produces identical hashes, so duplicate files get stored once, for free. Change one byte and the hash changes completely: whether you edited the file yourself or someone tampered with it behind your back, one comparison tells you. And the hash doesn't depend on the machine or the clock. Two machines hashing the same content always agree, which is the foundation distributed sync stands on.

There's one detail here the first version did exactly backwards from today. It hashed the zlib-compressed bytes, not the original content. Linus is explicit about it in the README:

> The SHA1 hash is always the hash of the _compressed_ object, not the original one.

The source does exactly that: `read-cache.c` deflates first, then runs SHA-1 over the compressed result. Today's Git does the reverse. It hashes the uncompressed content, then compresses for storage. So the same `hello` gets a different SHA-1 from the first version than from today's Git. Naming by content survived; which bytes to hash, before or after compression, got flipped once.

Then 2017 came, Google demonstrated a SHA-1 collision (SHAttered), and the community set out to move to SHA-256. Almost a decade in, that migration is still stuck. SHA-1 remains the default. GitHub still doesn't support SHA-256 repositories at all. The whole ecosystem sits in a deadlock: platforms won't support it because nobody migrates, and nobody migrates because platforms don't support it. A format decided on day one turned out to be nearly impossible to change twenty years later, because everyone and everything had grown on top of it.

Even if SHA-256 does land someday, what changes is just the hash algorithm. Content addressing itself never moved. Docker image layers, IPFS, and blockchains all run on it too.

## Snapshots, Not Diffs

For years I assumed Git stored diffs, like SVN, and that each `git commit` recorded "which lines changed". `git diff` shows you differences, so it's an easy thing to believe.

It doesn't. **Every Git commit stores a complete snapshot of the directory**, not a set of changes. To read a file at some version, you fetch it by hash. Nothing gets replayed from the beginning.

Isn't that a huge waste of space? No, because Git's **logical model and physical storage are two separate things**. Conceptually every commit is a full snapshot, but the storage doesn't duplicate everything:

1. Files that didn't change keep the same blob hash, so the blob is reused, not re-stored.
2. Files that did change get stored whole at first, one loose object each.
3. Once objects pile up, Git packs them into a packfile and delta-compresses similar objects against each other. At that layer, it's physically storing differences again.

Jump to any version you like; disk usage stays sane.

<img src="/images/git-logical-vs-physical.en.svg" alt="Git logical model vs physical storage: the model held, the parts got swapped" style="width:100%;max-width:900px;" />

The first version had loose objects only. Packfiles and delta compression came later. Later still, large files (video, model weights) turned out to delta-compress poorly, so Git LFS got bolted on outside. The storage machinery has been swapped several times. The model, one full snapshot per commit, hasn't moved.

## A Blob Doesn't Know Its Filename

Most version control systems store the filename and the file content together. Git split them apart.

A blob object holds **file content and nothing else, not even a filename**. What the file is called, its permissions, and which blob it points to all live in a different object: the tree. A tree is a snapshot of the directory structure at one moment.

Here's a real tree:

```bash
$ git cat-file -p e9760c66
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    a.txt
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    b.txt
```

Note that `a.txt` and `b.txt` have the same blob hash, because both contain `hello world`. Git stored one blob; the tree points two names at it.

<img src="/images/git-object-model.en.svg" alt="Git object model: commit → tree → blob, identical files share one blob" style="width:100%;max-width:900px;" />

Once content and names are separate, **renaming a file creates no new blob**. Rename `a.txt` to `renamed.txt` and the blob stays put; only the tree changes:

```bash
# the tree after the rename
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    b.txt
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    renamed.txt
```

Same blob hash, byte for byte. You changed a name; Git had no reason to store the content again.

There's a fun by-product. Git doesn't track renames, yet `git log --follow` can walk a file's rename history. It compares two trees, notices a blob that disappeared under one name and reappeared under another with identical content, and concludes that was probably a rename. Renames are computed, not stored.

That trick isn't a later invention either. Linus wrote it down in the first README:

> you can see trivial renames or permission changes by noticing that the blob stayed the same.

## Merkle DAG: Change One Byte and the Whole Chain Knows

Git's three object types (blob, tree, commit) reference each other and form a directed acyclic graph. Each object's name is a hash of its content, and that content contains the hashes of child objects. That's a Merkle DAG.

You've already seen the pieces: commits point to trees, trees point to blobs, every layer chained together by hash. Tamper with one byte of one file anywhere in history and that blob's hash changes, the tree containing it changes, the commit above changes, and everything downstream changes with it.

`git fetch` runs on this. If the hashes match, the content is untouched, no file-by-file comparison needed. It's also why `git push --force` is dangerous: rewrite one commit and the entire chain after it is invalidated.

Linus had already worked out what this chain was for. The first README has a whole section called TRUST, and its conclusion fits in one line:

> "git" itself only handles content integrity, the trust has to come from outside.

Git guarantees the content hasn't been tampered with; trust comes from elsewhere. GPG-sign the hash of a single top-level object and the entire history under it becomes trustworthy, because changing anything anywhere breaks the hash at the top. Today's signed tags and signed commits are exactly this design.

Merkle trees date back to 1979, and Git didn't invent them. But putting them under version control was the right call. Certificate Transparency logs and blockchains later did the same thing: use a Merkle structure to make history unforgeable.

## Separating Mechanism From Policy

The first version of Git compiled into seven low-level commands: `init-db`, `update-cache`, `write-tree`, `commit-tree`, `cat-file`, `show-diff`, `read-tree`. `git add`, `git commit`, `git push`: none of them existed.

Linus called these plumbing. Low-level mechanisms, each doing one job: compute a hash, write an object, read an object. No interaction design, no concern for user friendliness. A pile of parts.

Junio Hamano took over later and built porcelain on top: `git add` is essentially `update-cache`, and `git commit` is `write-tree` plus `commit-tree` plus a branch-ref update.

Even the object type names changed. In the first README, the commit object is called a changeset from start to finish. But the code in that very same commit already writes `commit` as the type tag (`commit-tree.c`). The documentation and the code disagreed on day one, and the code won.

Mechanism at the bottom, policy on top: that has been the Unix way all along. Linus didn't hard-code the interface, the workflows, or the branching policy, which is why Junio could redesign the whole user experience on top without touching a line underneath.

Today, GitHub, GitLab, lazygit, and every IDE plugin either call these low-level commands directly or, like libgit2, reimplement the same object formats from scratch. The interfaces differ everywhere you look; the model underneath is the same one. The plumbing has barely moved. The porcelain has been through several generations.

## The Common Thread

These five designs share one trait: each is a decision about what *not* to do. No filenames in blobs, no diffs in commits, no user interface in the core. Every piece cut away is one less thing the system will need to change later.

The hash algorithm can be swapped, the compression can be swapped, the transport can be swapped. The subtractions never need to be. Designs that survive twenty years usually aren't the ones that added something clever; they're the ones that left something out from the start.

Right now new AI models and frameworks ship faster than anyone can track. A design that has already survived twenty years won't expire because another framework came out. Time spent reading these things is time well spent.
