+++
title = "排查 Python 版本升级后的 SSL 问题：从 Python 3.8 到 3.11 的配置变化"
date = 2025-02-06
description = "Python 3.8 升级到 3.11 后 SSL 证书验证突然报错？这篇文章带你搞清楚 Python 3.10+ 的 SSL 默认行为到底改了什么，以及三种修复思路。"
tags = ["python", "ssl", "调试", "版本升级", "安全"]

[extra.comments]
issue_id = 8

[[extra.faq]]
question = "Python 3.8 升级到 3.11 后为什么 SSL 连接会报错？"
answer = "Python 3.10 开始，ssl 模块默认启用了更严格的证书验证：check_hostname 默认为 True，verify_mode 默认为 CERT_REQUIRED。之前在 3.8 中用 CLIENT_AUTH 目的创建的上下文不会强制验证，升级后同样的代码会因为证书链不完整或主机名不匹配而失败。"

[[extra.faq]]
question = "ssl.create_default_context() 在 Python 3.8 和 3.11 中有什么区别？"
answer = "主要区别在三个方面：1) 协议从 PROTOCOL_TLS 变为 PROTOCOL_TLS_CLIENT；2) check_hostname 默认从宽松变为强制 True；3) verify_mode 在 CLIENT_AUTH 场景下从 CERT_NONE 变为 CERT_REQUIRED。这些变化意味着 3.11 会严格验证服务器证书。"

[[extra.faq]]
question = "直接禁用 SSL 验证安全吗？"
answer = "不安全。禁用 check_hostname 和 verify_mode 会让连接容易受到中间人攻击（MITM）。这种做法只适合本地调试，绝对不应该用在生产环境。正确做法是修复证书链或者用 certifi 加载正确的 CA 证书。"

[[extra.faq]]
question = "怎么快速确认是 SSL 配置问题还是证书本身的问题？"
answer = "用 openssl s_client -connect host:port -showcerts 检查服务器证书链是否完整，再用 python -c 'import ssl; print(ssl.get_default_verify_paths())' 确认 Python 使用的 CA 证书路径。如果 openssl 验证通过但 Python 报错，大概率是 Python 的 CA 路径配置问题。"

[[extra.faq]]
question = "用 aiohttp 怎么正确配置 SSL 上下文？"
answer = "推荐用 ssl.create_default_context() 创建上下文，然后用 context.load_verify_locations(certifi.where()) 加载 CA 证书，再通过 aiohttp.TCPConnector(ssl=context) 传入。如果需要客户端证书，再额外调用 context.load_cert_chain()。"
+++

升级 Python 版本本来是个常规操作——直到你的 SSL 连接突然全部挂掉。

我在把项目从 Python 3.8 升到 3.11 的时候就遇到了这个问题。之前跑了一年多的代码，升级后直接报 `certificate verify failed`。排查了半天，发现不是证书坏了，而是 Python 3.10 开始悄悄收紧了 SSL 的默认配置。

这篇文章记录整个排查过程，以及我理解到的根本原因和几种修复思路。

<!--more-->

## 现象：升级后 SSL 直接炸了

项目用 `aiohttp` 做 HTTPS 请求，SSL 上下文是这样配的：

```python
import ssl

context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")
```

Python 3.8 下跑了一年多，没任何问题。升到 3.11 后，所有 HTTPS 请求都报错：

```
aiohttp.client_exceptions.ClientConnectorCertificateError:
  Cannot connect to host example.com:443 ssl:True
  [SSLCertVerificationError: certificate verify failed:
   unable to get local issuer certificate]
```

第一反应是证书过期了？检查了一下，没有。证书文件一模一样，服务器也没动过。那问题只可能出在客户端——Python 本身。

## 到底改了什么？

花了点时间翻 Python changelog 和 ssl 模块文档，总算搞清楚了。核心变化发生在 **Python 3.10**（PEP 644），3.11 继承了这些变化。

<img src="/images/python-ssl-changes.svg" alt="Python SSL 默认配置变化：3.8 vs 3.10+" style="width:100%;max-width:900px;" />

### 变化一：默认协议更严格

Python 3.8 的 `ssl.create_default_context()` 用的是 `PROTOCOL_TLS`，这个协议比较宽松，客户端和服务端都能用。

Python 3.10+ 改成了 `PROTOCOL_TLS_CLIENT`，这个协议专门用于客户端连接，并且**强制开启证书验证**。

### 变化二：check_hostname 默认开启

在 3.8 中，如果你用 `ssl.Purpose.CLIENT_AUTH` 创建上下文，`check_hostname` 实际上是 `False`——它不会验证服务器证书的主机名是否匹配。

3.10+ 里，不管你用什么 Purpose，`check_hostname` 默认都是 `True`。如果证书的 CN（Common Name）或 SAN（Subject Alternative Name）跟你连接的主机名不一致，直接报错。

### 变化三：verify_mode 默认 CERT_REQUIRED

3.8 用 `CLIENT_AUTH` 时，`verify_mode` 是 `CERT_NONE`——压根不验证服务器证书。这意味着即使证书链不完整、根证书缺失，连接照样能建立。

3.10+ 改成了 `CERT_REQUIRED`，要求完整的证书链验证。如果 Python 找不到对应的 CA 根证书，就会报 `unable to get local issuer certificate`。

### 一句话总结

**Python 3.8 默认信任一切，Python 3.10+ 默认不信任任何东西。** 你之前的代码"能跑"只是因为验证被关掉了，不是因为证书配置正确。

## 排查步骤

在知道原因之前，我是这样一步步定位的。如果你遇到类似问题，可以按这个顺序排查：

### 第一步：确认是 Python 的问题，不是证书的问题

用 `openssl` 直接检查服务器证书：

```bash
openssl s_client -connect example.com:443 -showcerts
```

如果输出里有 `Verify return code: 0 (ok)`，说明证书本身没问题，问题在 Python 这边。

### 第二步：检查 Python 用的 CA 路径

```python
import ssl
print(ssl.get_default_verify_paths())
```

输出类似：

```
DefaultVerifyPaths(
  cafile='/opt/homebrew/etc/openssl@3/cert.pem',
  capath='/opt/homebrew/etc/openssl@3/certs',
  ...
)
```

确认这些路径下确实有 CA 证书文件。macOS 上特别容易踩坑——系统 OpenSSL 和 Homebrew 装的 OpenSSL 用的是不同的证书路径。

### 第三步：对比 3.8 和 3.11 的 SSLContext 默认值

```python
import ssl

# 看看你的上下文到底长什么样
ctx = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
print(f"protocol: {ctx.protocol}")
print(f"check_hostname: {ctx.check_hostname}")
print(f"verify_mode: {ctx.verify_mode}")
```

在 Python 3.8 和 3.11 分别跑一遍，你就能看到差异。

## 三种修复方案

### 方案一：修复证书链（推荐）

这是最正确的做法。把完整的证书链配好，让验证正常通过：

```python
import ssl
import certifi

context = ssl.create_default_context(cafile=certifi.where())
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")
```

`certifi` 是一个维护良好的 CA 证书包，pip 可以直接装。这样做的好处是不依赖系统的 CA 路径，跨平台一致。

```bash
pip install certifi
```

### 方案二：手动配置 SSLContext

如果你需要更细粒度的控制，可以手动创建 SSLContext：

```python
import ssl

context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.load_verify_locations("/path/to/ca-bundle.crt")
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")

# 用于 aiohttp
import aiohttp
connector = aiohttp.TCPConnector(ssl=context)
async with aiohttp.ClientSession(connector=connector) as session:
    async with session.get("https://example.com") as resp:
        print(await resp.text())
```

### 方案三：禁用验证（仅限调试）

如果你只是想在本地快速调试，临时禁用验证：

```python
import ssl

context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.check_hostname = False
context.verify_mode = ssl.CERT_NONE
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")
```

**注意：** 这种做法会让连接暴露在中间人攻击风险下。只在本地调试时使用，绝对不要用在生产环境。你的 code review 流程应该能拦住这种代码。

## 一个容易忽略的坑：macOS 上的 CA 路径

macOS 用户特别需要注意：如果你用 Homebrew 装了 Python，它链接的 OpenSSL 可能跟系统自带的不一样。这意味着 Python 可能找不到系统 Keychain 里的 CA 证书。

最简单的解决办法就是用 `certifi`：

```python
import certifi
print(certifi.where())
# 输出类似: /path/to/python/site-packages/certifi/cacert.pem
```

然后把这个路径传给 `ssl.create_default_context(cafile=certifi.where())`。

## 总结

Python 3.10+ 收紧 SSL 默认配置是好事——它强制你正确处理证书验证，而不是假装问题不存在。但如果你从 3.8 直接升上来，这个变化会很突然。

关键 takeaway：

1. **`ssl.create_default_context()` 的行为在 3.10 变了** —— `check_hostname` 和 `verify_mode` 默认开启
2. **先确认是客户端问题还是证书问题** —— 用 `openssl s_client` 排除服务端因素
3. **优先修复证书链，而不是禁用验证** —— `certifi` 包是最省事的方案
4. **macOS 上注意 CA 路径** —— Homebrew Python 和系统 Python 用不同的证书路径

### 参考文档

- [Python 3.11 ssl 模块文档](https://docs.python.org/zh-cn/3.11/library/ssl.html)
- [PEP 644 — Require OpenSSL 1.1.1 or newer](https://peps.python.org/pep-0644/)
- [certifi — Python Certificate Authority Bundle](https://pypi.org/project/certifi/)
