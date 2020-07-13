---
title: 聊聊 API 签名方式
tags:
  - API 签名
  - 认证
categories:
  - 技术
copyright: true
date: 2019-12-24 22:41:31
---



现在越来越多的公司以 API 的形式对外提供服务，这些 API 接口大多暴露在公网上，所以安全性就变的很重要了。最直接的风险如下：

- 非法使用 API 服务。（收费接口非法调用）
- 恶意攻击和破坏。（数据篡改、DOS）

因此需要设计一些接口安全保护的方式来增强接口安全，在运输层可添加 SSL 证书，上 HTTPS，在应用层主要是通过一些加密逻辑来实现。目前主流的两种是在 HTTP Header 里加认证信息和 API 签名。

<!-- more -->

## HTTP 简单身份认证

在 HTTP 请求的 Header 中添加认证字段例如：

`Authorization: 3F2504E04F8911D39A0C0305E82C3301`

服务器处理前取出该字段进行校验即可。

Spring Boot 项目直接实现一个拦截器就可进行判断：

```java
String token = request.getHeader("Authorization");
if (!Strings.isNullOrEmpty(token)) {
    hasAuth = redisTemplate.hasKey("userToken:" + token);
}
```

这类方法实现比较简单，可以做基本的身份认证，防君子不防小人，可通过中间人攻击获得 Authorization。使用 HTTPS 安全性会得到提高，但是无法抵御重放攻击造成的影响，例如 DDOS。

## API 签名认证

API 签名的方式较前一种要复杂的多，但是可解决的安全问题也更多：

- 可校验请求者的合法性。
- 可以校验参数的完整性和是否被篡改。
- 可以防止重放攻击。

**part1：请求端加密**

API 使用者会获取到服务器颁发的 ak 和 sk 两个秘钥，ak 为公钥，sk 为私钥。

签名有以下规则：

1. 约定请求时会携带 ak 作为参数并放入 HTTP Header。
2. 请求参数处理：
   - GET：取出所有的参数，并根据 key 进行字典排序，拼装成如下格式。
   - POST：如果是 `application/x-www-form-urlencoded`，直接取出和 GET 参数进行排序拼接，如果是 `application/json`，则直接将整个 json 串 md5 加密后再 base64。

```
GET:
strToSign = uri + "\n" + k1=v1Z&k2=v2&k3=v3
POST:
strToSign = uri + "\n" + k1=v1&k2=v2&k3=v3 + "\n" + base64(md5(json_body))
```

3. 使用 HmacSHA256 算法，传入 sk 进行签名计算，`sign = HmacSHA256(K,strToSign)`，其中`K = sk`。
4. 组装 HTTP 请求，将 `X-Ca-Key=ak,X-Ca-Signature=sign` 添加到 HTTP Header 中进行请求。

一个简单实例如下图所示：

![image](https://blog-20190524.oss-cn-hangzhou.aliyuncs.com/images/talking-about-api-signature/697102-20191224200438391-700977733.png?x-oss-process=style/logo)

**part2：服务端解密**

同样的，服务端获取到请求的 ak 后，查询出对应的 sk，使用相同的规则进行签名计算，计算出的 sign 和传入的 sign 比对，就能够知道参数是否被篡改。

经过这样签名方式后，可以保证上文提到的，校验请求者的合法性和校验参数的完整性和是否被篡改。但是如果有人施加一个中间人攻击，就可以获取到请求报文，即便攻击者无法破译出签名规则，也可以将请求重放，也就是原封不动提交给服务器，那么如果发起恶意大规模攻击，就会使服务器产生拒绝服务。

### 更进一步

如果需要的安全性更高，可以采用 timestamp 和 nonce 来解决这个问题。

- timestamp：很好理解，调用者传入，服务端判断，一段时间失效。
- nonce：在信息安全中，nonce 是一个在加密通信只能使用一次的数字。

单一使用 timestamp，可以一定程度上减少重放攻击的频率，但是无法完全遏制。

单一使用 nonce，咋一眼看可以保证请求的唯一性，但实际上服务端，随着时间推移服务端无法存储大量的 nonce，需要进入淘汰环节，一旦旧的 nonce 被淘汰，那么攻击者依旧可以卷土重来进行重放攻击。

因此，将两者结合一起来就是最终的方案，服务端首先验证 nonce 是否存在，再校验时间戳是否在规定的期限内。如果旧 nonce 被清理，也有时间戳进行把关，使得请求无法被重放。

**part1：请求端生成 timestamp 和 nonce**

生成时需要保证短时间内生成 nonce 的唯一性。

将 timestamp 和 nonce 写入 HTTP Header 中。

**part2：服务端校验**

1. 数据库查询请求带上的 nonce 是否存在（推荐使用Redis，自带TTL功能）。
2. 如果不存在，且请求时间有效则为合法请求，同时将 nonce 写入，并记录时间；如果不存在，且请求时间超出规定时限，判断为恶意请求。
3. 如果已经存在，判断为恶意请求。

做足以上这几部基本上已经可以保证一定的安全性。当然还有更复杂的，可以阅读阿里的 [Open API 签名文档](https://help.aliyun.com/document_detail/29475.html?spm=a2c4g.11186623.2.13.2aa44ae01LxLfD)，根据项目自身对于安全性的需求可以适当进行简化。本文讲到的基本逻辑就是根据阿里的来的。

## API 签名与 HTTPS

这边还想提一下 HTTPS。之前看到一则知乎上的提问：[使用了 https 后，还有必要对数据进行签名来确保数据没有被篡改吗？](https://www.zhihu.com/question/52392988)

总结一下就是：

- API 签名保证的是应用的数据安全和防篡改，并且可以作为业务的参数校验和处理重放攻击。

- HTTPS 保证的是运输层的加密传输，但是无法防御重放攻击。

换句话说，HTTPS 保证通过中间人攻击抓到的报文是密文，无法很难破解。但仍然可以将报文重发，形成 DDOS。同时，如果不签名，只用 HTTP 简单认证，通过抓包，直接可以获取到 Authorization，就可以随意发起请求了。因此最安全的方法就是结合 HTTPS 和 API 签名。

## 总结

本文主要介绍了当前主流 API 签名方式，可以根据项目场景去挑选组合合适的方案。

博客还附带实现了一个根据上文规则描述的工具类，可以用于API签名，可见下面源码链接。如果需要使用 timestamp 和 nonce 的可在此基础上将这两个字段添加到 sortedMap 中一起拼接，并集成Redis。

## [源码](https://github.com/maoqyhz/tequila/tree/master/springboot-api-sign)


## 参考文献

- [Open API 中签名的使用](https://zhuanlan.zhihu.com/p/36986383)
- [aliyun - 请求签名说明文档](https://help.aliyun.com/document_detail/29475.html?spm=a2c4g.11186623.2.13.2aa44ae01LxLfD)
- [API 接口设计：防参数篡改 + 防二次请求](https://cloud.tencent.com/developer/article/1175758)