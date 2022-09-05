
## 基本概念

OAuth2解决了使用通用的账户系统，授权并登陆第三方程序的问题。比如使用新浪微博账号登陆知乎，比如使用google账户登陆stackoverflow。

### 角色(Roles)
- resource owner: 受保护信息资源的拥有者，比如新浪微博的用户，以下简称Owner
- resource server: 受保护资源的提供者，比如新浪微博，以下简称Server
- client: 第三方应用或者网站，比如知乎，以下简称Client
- authorization server: 授权服务器，可以是resource server本身或者单独出现，比如新浪微博登录API，以下简称Auth
  - authorize API (endpoint): 验证Owner，发放Auth Code，须支持GET请求
  - token API (endpoint): 验证Client，发放Token

### User Agent

授权流程中User Agent就是你使用的浏览器或者App。Client或者Auth通过重定向User Agent到不同的URL完成授权过程，所以它是整个授权过程中的交互核心，详见[授权类型](#授权类型)中的时序图描述部分。

### Token

#### Acess Token

Access Token是整个授权流程的最终目标。第三方Client从Auth成功获取后便可以使用它访问受保护Server资源。通常情况下Access Token是一个经过Hash或者加密的字符串([Bear Token - RFC6750](https://tools.ietf.org/html/rfc6750))，其内容对第三方Client不透明(opaque，即第三方Client无须关注其内容构成)。它本身可以包含经过签名（以便Resource Server验证）的完整的授权信息，也可以只是完整授权信息的id。

Client取得Token后通常将其作为Header设给Authroziation:

```
# bear类型
Authorization: Bearer mF_9.B5f-4.1JqM
# mac类型
Authorization: MAC id="h480djs93hd8",nonce="274312:dj83hs9s",mac="kDZvddkndxvhGRXZhvuDjEWhGeE="
```


#### Refresh Token

Auth可以在返回Access Token时附上Refresh Token，以便当Access Token过期或者失效后使用Refresh Token获取新的Access Token。Refresh Token的发放是**可选的**，且仅由Auth返回给Client，它不应该被发送给Server。

#### (Access Token) Scope

授权请求的可选参数，用于限定授权范围，通常为空格分隔的大小写敏感的列表字符串。具体的scope列表应该有Auth设计提供。Auth可以完全忽略请求中的scope，或者返回和请求不一致的scope。

### HTTPS

OAuth2强调必须使用HTTPS，否则明文传输auth code，token或者Client密码存在极大安全隐患。

### Client注册

第三方Client根据类型需要向Auth注册以通过[授权码方式中步骤D](#Authorization Code)的验证，进一步提升安全性，保护Refresh Token。

注册内容通常有：

- client_id: Auth上唯一表示该Client的字符串
- client_secret: 密码或者秘钥对。
- client type: 两种客户端类型之一，如果构成复杂则需要分开注册
  - confidential: 可以安全保存秘钥凭据的实现，比如web后端
  - public: 运行在Owner设备上的应用，比如单页应用，本地桌面程序
- redirection URIs: 请求完成是浏览器跳转返回的链接，URI越完整越安全。如果Client没有注册，Auth请求中必须包含redirect_uri。

其中必须注册的Client包括：
- 使用[Implicit授权](#Implicit授权)的confidential类型
- public类型

规范中Auth必须支持HTTP Basic Auth进行Client密码校验。


## 授权类型

### 授权码授权 (Authorization Code Grant)

> the transmission of the access token directly to the client without
  passing it through the resource owner's user-agent and potentially
  exposing it to others, including the resource owner.

安全性最高的授权模式：最终的访问令牌直接发送给client程序，绕过了UserAgent(浏览器)，所以无论是你自己或者任何浏览器中运行的其它程序均无法获得。


#### 流程

``` mermaid
sequenceDiagram
    participant O as Resource Owner
    participant C as Client
    participant UA as User Agent
    participant AS as Auth Server

    C->>+UA: A1. 使用新浪微博登陆知乎
    
    UA->>+AS: A2. 知乎重定向浏览器到新浪微博登录页并附加请求信息：
    Note over UA, AS: response_type=code client_id [redirect_uri] [scope] [state]

    O->>AS: B. 在浏览器中登录新浪微博，验证授权请求scope，比如获取你的邮箱

    AS-->>-UA: C1. 浏览器重定向到步骤A中的redirect_uri
    Note over UA, AS: 附加至查询参数:code=xx&state=xx 或失败:error=access_denied&state=xx

    UA-->>-C: C2. 知乎JS应首先执行，从URI中获取并删除auth code

    C->>+AS: D. POST请求：用code换access token
    Note over UA: code grant_type client_id redirect_uri 

    AS->>AS: authenticate client, validate code/redirect-uri

    AS-->>-C: E. access token, optional refresh token in body
```

#### HTTP请求示例

```
# Client授权请求，浏览器被重定向到登录页面
https://authorization-server.com/auth?response_type=code&
  client_id=CLIENT_ID&redirect_uri=REDIRECT_URI&scope=photos&state=1234zyx

# 登录、授权成功，Auth使用以下链接将浏览器重定向回Client
https://example-app.com/cb?code=AUTH_CODE_HERE&state=1234zyx

# Client使用code换取Access Token
POST https://api.authorization-server.com/token
  grant_type=authorization_code&
  code=AUTH_CODE_HERE&
  redirect_uri=REDIRECT_URI&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
```


### 隐式授权 (Implicit Grant)

> simplified authorization code flow optimized for clients implemented in a browser using a
  scripting language such as JavaScript.

一步完成Access Token获取，无须显式Client验证（仅通过redirection URI隐式验证）。Access Token会被加入redirection URI fragment(#哈希部分），并存留于User Agent，降低了安全性。

**该授权方式最早为为单页应用设计，但目前在新版规范中已经废弃！请使用code授权方式，但是不使用client secrect。即在使用code换取token时，不使用client_secrect**

#### HTTP请求示例

```
# Client授权请求
https://authorization-server.com/auth?response_type=code&client_id=CLIENT_ID&redirect_uri=REDIRECT_URI&scope=photos&state=1234zyx

# Auth 端在Client成功授权后，对浏览器重定向
https://example-app.com/cb?code=AUTH_CODE_HERE&state=1234zyx

# Client使用code换取Access Token
POST https://api.authorization-server.com/token
  grant_type=authorization_code&
  code=AUTH_CODE_HERE&
  redirect_uri=REDIRECT_URI&
  client_id=CLIENT_ID
```

### 客户程序凭证授权(Client Credentials Grant)

第三方程序(Client)通过已经注册的凭证直接向Auth申请Access Token以访问由自己控制的资源(比如API)。

最常见的情况为[微信小程序可以获取Access Token的过程](https://developers.weixin.qq.com/miniprogram/dev/api/token.html#%E8%8E%B7%E5%8F%96-accesstoken)：

```
GET 'https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appId=xxxxxxxxx&secret=xxxxxxxxxxxxx'
```

### 资源所有者密码授权(Resource Owner Password Credentials Grant)

当且仅当资源所有者与第三方程序有极强的信任关系，或者第三方为高优先级程序(比如操作系统)时才可以使用的授权方式。
另外一种情况就是讲已有的凭证进行转换升级，比如将已经保存的老版本OAuth凭据转换为Access Token。


## 参考文章

- [The OAuth 2.0 Authorization Framework - RFC6749](https://tools.ietf.org/html/rfc6749)
- [理解OAuth 2.0 - 阮一峰](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
- [Understanding OAuth2](http://www.bubblecode.net/en/2016/01/22/understanding-oauth2/)
- [OAuth 2 Simplified](https://aaronparecki.com/oauth-2-simplified/)
