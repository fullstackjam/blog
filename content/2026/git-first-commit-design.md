+++
title = "Git 的第一次提交：五个历久弥新的设计"
date = 2026-06-26
description = "Git 的第一次提交，11 个文件 1244 行。翻完发现里面几个设计决定，二十年过去几乎没动。逐个回味它们妙在哪，后来又怎么变的。"
tags = ["Git", "系统设计", "源码阅读", "内容寻址", "开源"]

[extra.comments]
issue_id = 9
+++

Git 用了这么多年，原理一直似懂非懂。前阵子突发奇想，把 Git 仓库切到了它自己的第一次提交：

```
commit e83c5163316f89bfbde7d9ab23ca2e25604af290
Author: Linus Torvalds <torvalds@ppc970.osdl.org>
Date:   Thu Apr 7 15:13:13 2005 -0700

    Initial revision of "git", the information manager from hell

 11 files changed, 1244 insertions(+)
```

1244 行，11 个文件。翻完有点意外——好几个设计决定，二十年过去几乎没动，今天还在我每天用的 Git 里跑着。

<!--more-->

先说一个最出乎我意料的。

我一直以为暂存区（staging area）是 Git 后来才补的设计——它是新手最绕不过去的概念，怎么看都像后加的复杂度。翻开第一版才发现，它第一天就在，还是整个系统的两根支柱之一。README 开头就讲：Git 只有两个抽象，一个对象库，一个"当前目录缓存"（current directory cache）——后者就是今天的暂存区。

更狠的是它的文件头。第一版给这个索引文件定的魔数是 `DIRC`，DIRectory Cache 的缩写：

```c
#define CACHE_SIGNATURE 0x44495243	/* "DIRC" */
```

二十年过去，你现在 hexdump 一下自己任何一个仓库的 `.git/index`，头四个字节还是它：

```
$ hexdump -C .git/index | head -1
00000000  44 49 52 43 00 00 00 02  ...  |DIRC....|
```

格式从 version 1 升到了今天的 version 2、3、4，那四个字节一个没动。

这样的设计，我在那 1244 行里数了数，至少五个，二十年几乎没碰过。逐个回味它们妙在哪、后来又怎么变的。

## 名字就是内容

存文件的时候，常规做法是给每个文件分个自增 ID，再维护一张"ID → 文件路径"的映射表。大多数数据库都这么干。

Git 走了另一条路。它把文件内容本身算一个 SHA-1 哈希，拿这串哈希直接当对象名——内容寻址（content-addressable storage）。

```bash
$ printf 'hello' | git hash-object --stdin
b6fc4c620b67d95f953a5c1c1230aaab5db5a1b0
```

内容是 `hello`，对象就叫 `b6fc4c…`，存在 `.git/objects/b6/fc4c…` 里。名字不是分配的，是从内容算出来的。

这样一来，内容一样哈希就一样，重复文件天然只存一份。改一个字节哈希就全变了，不管是你自己改的还是文件被谁偷偷动了，一比就知道。而且哈希跟机器、跟时间无关，两台机器各算各的结果必然一致，分布式同步的地基也在这儿。

第一版这里还有个跟今天正好相反的地方：它算哈希，算的不是原始内容，是 zlib 压缩之后的字节。Linus 在 README 里写得很直白：

> The SHA1 hash is always the hash of the _compressed_ object, not the original one.

源码也确实这么干——`read-cache.c` 里先 deflate 压缩，再对压缩结果算 SHA-1。今天反过来了：现在的 Git 对压缩前的内容算哈希，再压缩存盘。所以同一份 `hello`，拿第一版和今天分别算，SHA-1 不是一个值。名字从内容算，这个大方向没变；但"从哪份内容算"，压缩前还是压缩后，换过一次。

再后来，2017 年 Google 演示了 SHA-1 碰撞（SHAttered），社区想换成 SHA-256。可这一换，换了快十年还没换动——SHA-1 至今仍是默认，GitHub 到现在都不支持 SHA-256 仓库，整个生态卡在"平台不支持所以没人迁、没人迁所以平台不上"的死结里。一个第一天定下的格式，二十年后想动都动不了，因为所有人、所有工具都长在它上面了。

不过就算哪天真换成了 SHA-256，动的也只是哈希算法——内容寻址这个思路本身没动过。Docker 的镜像层、IPFS、区块链，用的也都是它。

## 存快照，不存改动

我以前一直以为 Git 跟 SVN 一样，存的是 diff——每次 `git commit` 记的就是"改了哪几行"。毕竟 `git diff` 看到的就是差异，很容易这么想。

实际上不是。**Git 每次提交存的是一个完整的目录快照**，不是差异。要看某个版本的文件，直接按哈希取出来就行，不需要从头回放任何东西。

这不是很浪费空间吗？其实不会。Git 的**逻辑模型和物理存储是分开的**——概念上每个 commit 是完整快照，但实际存储不会把所有内容重复存一遍：

1. 没变的文件，blob 的哈希不变，直接复用，不重复存。
2. 变了的文件，一开始确实各存一份完整的 blob（loose object）。
3. 对象多了以后，Git 会打包成 packfile，相似的对象之间做 delta 压缩——到这一步，物理上就又是存差量的了。

随便跳到任意版本，空间也不会炸。

<img src="/images/git-logical-vs-physical.svg" alt="Git 逻辑模型 vs 物理存储：模型没动，零件换了一轮" style="width:100%;max-width:900px;" />

第一版其实只有 loose object，packfile 和 delta 压缩都是后来加的。再后来大文件（视频、模型权重）delta 压不动，又外挂了 Git LFS。存储方式换了好几轮，但"每个 commit 是完整快照"这个模型一直没变。

## blob 不认识文件名

大多数版本控制系统把文件名和文件内容绑在一起存，Git 把它们拆开了。

blob 对象里**只有文件内容，没有文件名**。文件叫什么名、什么权限、对应哪个 blob，这些信息记在另一种对象——tree 里。tree 就是某个时刻的目录结构快照。

看一下真实的 tree 长什么样：

```bash
$ git cat-file -p e9760c66
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    a.txt
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    b.txt
```

注意 `a.txt` 和 `b.txt` 的 blob 哈希是一样的——因为内容都是 `hello world`。Git 只存了一份 blob，tree 里两个名字指向同一份。

<img src="/images/git-object-model.svg" alt="Git 对象模型：commit → tree → blob，同内容文件共享 blob" style="width:100%;max-width:900px;" />

内容和名字分开之后，**改文件名不会产生新的 blob**。把 `a.txt` 重命名为 `renamed.txt`，blob 不变，只有 tree 变了：

```bash
# 重命名后的 tree
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    b.txt
100644 blob 3b18e512dba79e4c8300dd08aeb37f8e728b8dad    renamed.txt
```

blob 哈希一模一样。你只是换了个名字，Git 根本不需要重新存文件内容。

还有个有意思的副产品：Git 不跟踪重命名，但 `git log --follow` 能追踪文件改名历史。它靠的是比较前后两个 tree，发现一个 blob 从一个名字消失、又在另一个名字下出现，内容一样——那大概就是改名了。重命名是算出来的，不用专门存。

这个"靠 blob 没变来认出改名"的思路，也不是后人补的。Linus 在第一版 README 里就写了：

> you can see trivial renames or permission changes by noticing that the blob stayed the same.

## Merkle DAG：改一个字节，整条链都知道

Git 的三种对象（blob、tree、commit）互相引用，构成一个有向无环图（Directed Acyclic Graph, DAG）。每个对象的名字是内容的哈希，而内容里又包含了子对象的哈希——这就是 Merkle DAG。

前面已经看到了：commit 指向 tree，tree 指向 blob，每层都靠哈希串起来。所以如果你篡改了历史中某个文件的一个字节，那个 blob 的哈希会变，包含它的 tree 哈希跟着变，再往上 commit 哈希也变，后面整条链全部跟着变。

`git fetch` 靠这个工作——哈希对得上，内容一定没被动过，不用逐个文件比对。`git push --force` 之所以危险也是因为这个：你改了一个 commit 的内容，它后面的整条链就全废了。

Linus 第一天就想清楚了这条链能拿来做什么。第一版 README 有一整节叫 TRUST，结论就一句话：

> "git" itself only handles content integrity, the trust has to come from outside.

Git 只保证内容没被篡改，信任得从外部来——你用 GPG 签名顶层对象的哈希，整条历史就都可信了，因为中间任何一处被动过，顶层哈希都对不上。今天的 signed tag、signed commit，就是这套设计。

Merkle 树本身是 1979 年的东西，不是 Git 发明的。但 Linus 把它用在版本控制上确实合适。后来证书透明度日志（Certificate Transparency）和区块链做的也是这件事——拿 Merkle 结构保证历史改不了。

## 机制与策略分离

第一版 Git 编译出来只有 7 个底层命令：`init-db`、`update-cache`、`write-tree`、`commit-tree`、`cat-file`、`show-diff`、`read-tree`。`git add`、`git commit`、`git push` 一个都没有。

Linus 管这些叫 plumbing（水管）——底层机制，各自只做一件事：算哈希、写对象、读对象。没有交互设计，不考虑用户友好，就是一堆零件。

后来 Junio Hamano 接手，在 plumbing 之上封装了 porcelain（瓷器）——`git add` 本质上就是 `update-cache`，`git commit` 其实就是 `write-tree` + `commit-tree` + 更新分支引用。

连对象类型的名字都变过。第一版 README 里，commit 这种对象从头到尾叫 changeset；但同一次提交的代码中，写进对象的类型标签已经是 commit 了（`commit-tree.c`）。文档和代码第一天就没对齐，最后代码赢了。

底层只给机制，上层自己定策略，Unix 一直是这个思路。Linus 没把界面、交互流程、分支策略写死在代码里，所以 Junio 接手以后能在上面重新设计用户体验，底层一行没动。

到了今天，GitHub、GitLab、lazygit、各种 IDE 插件，有的直接调这些底层命令，有的像 libgit2 那样按同一套对象格式重新实现了一遍。界面和交互各家完全不同，底下对着的是同一个模型。plumbing 基本没动，porcelain 翻了好几轮。

## 写在最后

这五个设计有个共同点：它们都在选"不做什么"。不存文件名到 blob 里，不存 diff 到 commit 里，不把用户界面写进底层。每砍掉一个不该有的东西，系统就少一个将来要改的地方。

哈希算法可以换，压缩方式可以换，传输协议可以换，但这些做减法的决定不需要换。能扛住时间的设计，往往不是加了什么，是一开始就没加。

现在 AI 新模型新框架一周冒出来好几个，追都追不完。但扛住了时间的设计不会因为又出了个新框架就过期。花点时间看看这些东西，不亏。
