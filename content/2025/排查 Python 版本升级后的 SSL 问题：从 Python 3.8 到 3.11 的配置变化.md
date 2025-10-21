+++
title = "排查 Python 版本升级后的 SSL 问题：从 Python 3.8 到 3.11 的配置变化"
date = 2025-02-06

[extra.comments]
issue_id = 8
+++

在进行 Python 项目的 SSL/TLS 连接配置时，很多开发者可能遇到过类似的问题：在升级 Python 版本后，之前正常工作的代码突然开始出现 SSL 错误。这个问题对于使用 Python 3.8 版本的开发者来说，在升级到 Python 3.11 时尤为常见。在本文中，我将分享我在升级过程中遇到的 SSL 配置问题，并提供解决方案，帮助遇到同样问题的开发者理解问题的根本原因及其解决方法。
<!--more-->

## 问题背景

在我的项目中，我使用了 `ssl.create_default_context()` 来配置 SSL 上下文，并在客户端与服务器之间建立安全连接。之前在 Python 3.8 中，这段代码可以正常工作：

```python
context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")
```

然而，当我将 Python 版本从 3.8 升级到 3.11 后，代码出现了一个问题，错误信息为：

```
aiohttp.client_exceptions.ClientConnectorCertificateError: certificate verify failed: unable to get local issuer certificate
```

这个错误表明 SSL 证书验证失败，具体是因为无法获取本地的证书颁发机构（CA）证书。最初，我认为可能是证书本身存在问题，但经过进一步排查，我发现问题出在 Python 升级后默认的 SSL 配置发生了变化。

## 通过排查找到问题的根本原因

### 1. **默认 SSL 上下文配置的变化**

在 Python 3.8 中，`ssl.create_default_context()` 默认启用了 `PROTOCOL_TLS` 协议，且在很多情况下，并没有严格的证书验证，尤其是 `check_hostname` 和 `cert_required` 的默认设置是比较宽松的。这意味着即使证书链不完整，或者证书主机名验证失败，程序仍然可以顺利运行。

然而，在 Python 3.11 中，`ssl.create_default_context()` 默认使用 `PROTOCOL_TLS_CLIENT` 协议，并启用了更严格的证书验证。特别是：
- `check_hostname` 默认为 `True`，即要求服务器证书中的主机名与连接的服务器主机名匹配。
- `cert_required` 默认为 `True`，即默认会验证服务器证书的有效性，确保证书链完整。

因此，Python 3.11 中会在连接时更严格地检查证书，导致如果证书链不完整或证书不受信任，连接将失败，报出 `certificate verify failed` 错误。

### 2. **证书验证和主机名验证的默认开启**

在 Python 3.11 中，`SSLContext` 对象的默认配置已经发生了变化。具体来说：
- 默认启用了主机名验证（`check_hostname = True`），如果证书中的主机名和实际主机名不匹配，就会抛出异常。
- 默认启用了证书验证（`verify_mode = ssl.CERT_REQUIRED`），如果证书无效或缺失根证书，也会导致连接失败。

这些默认设置增加了 SSL 连接的安全性，但也可能导致原本在 Python 3.8 中能够正常运行的代码在 Python 3.11 中出现问题。

## 解决方案

为了让代码在 Python 3.11 中正常工作，并避免过度修改已有的证书配置，我采取了以下解决方法：

### 1. **显式设置 SSL 上下文配置**

在 Python 3.11 中，显式设置 `SSLContext` 的协议版本以及证书验证选项，可以恢复与 Python 3.8 类似的行为。具体代码如下：

```python
import ssl
import aiohttp

# 创建 SSL 上下文，指定客户端协议
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)

# 禁用主机名验证
context.check_hostname = False

# 禁用证书验证
context.verify_mode = ssl.CERT_NONE

# 加载证书
context.load_cert_chain(certfile="mycertfile", keyfile="mykeyfile")

# 创建 aiohttp 会话并发起请求
async with aiohttp.ClientSession(connector=aiohttp.TCPConnector(ssl_context=context)) as session:
    async with session.get('https://example.com') as response:
        print(await response.text())
```

### 2. **理解解决方案的安全影响**

虽然这种做法解决了在 Python 3.11 中由于证书验证问题导致的错误，但它禁用了 SSL/TLS 的证书验证和主机名验证。在生产环境中，禁用证书验证会导致潜在的安全风险，例如容易受到中间人攻击（MITM）。因此，建议在生产环境中：
- 使用有效的证书链和根证书。
- 确保服务器的证书与主机名匹配，避免禁用 `check_hostname` 和 `verify_mode`。

## 总结

在将 Python 版本从 3.8 升级到 3.11 后，由于默认的 SSL 上下文配置发生了变化，导致原本正常工作的代码无法再通过证书验证。这是因为 Python 3.11 默认启用了更严格的证书验证和主机名验证。通过显式设置 `SSLContext` 的属性，我们可以恢复 Python 3.8 中的行为，解决了证书验证失败的问题。

然而，禁用证书验证和主机名验证虽然解决了问题，但也带来了安全风险。因此，在正式环境中，我们应确保证书链的完整性并启用相关的验证，以保障 SSL/TLS 连接的安全。

希望这篇博客能帮助到其他遇到类似问题的开发者，并提供一个参考解决方案。

### 参考文档

- [Python 3.11 ssl 模块文档](https://docs.python.org/zh-cn/3.11/library/ssl.html)
