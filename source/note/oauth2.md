## OAuth2——鉴权码模式

### 背景介绍
试想有这样一个场景，作为一款App开发者，你想要通过*github*或*google*等账号登录，要如何实现？

首先，你不可能直接拿到用户*github*的账号和密码，因为这会存在密码泄露的问题。即使用户信任你的应用，你的App与第三方账号用户密码的一致性也很难保证

其次，假设你的App在登录时，通过*github*进行密码验证后，直接返回用户信息可以吗？这样第一次登录是没问题的，但很显然无法保证登陆的时效性。你的App不能一直保存用户信息，长时间不操作应该能自动将用户退出；一次登录，永久有效显然也不够安全

所以，需要这样一个机制：安全地使用第三方应用的用户信息，并能够实时校验用户的有效性
### OAuth2协议原理
OAuth2是OAuth协议的2.0版本，它就是一种安全的方式去授权第三方应用（你自己的应用）去获取其他应用（*github*）的资源（用户信息）

#### 角色
OAuth2协议定义了以下几种角色：  

**User Agent**：浏览器   
**Client**：第三方应用   
**Resource Owner**：资源所有者，即用户   
**Authorization Server**：授权服务器，即提供第三方登录服务的服务器，如*github*   
**Resource Server**：用户资源信息的服务器
#### 流程
授权码模式是OAuth2协议流程最严密、功能最完整实现方式，其流程如下所示：
![oauth2流程](asserts/img/oauth2_flow.png "OAUTH2 流程")

(A)：用户访问客户端，客户端跳转到授权服务器提供的授权页面上

授权服务器提供的重定向URL格式如下：
```
https://ip:port/xxx?client_id=client_id&redict_url=redict_url
```
其中：   
`redirect_uri`：表示重定向URI，必选项，表示授权完成后会跳转的地址   
`client_id`：表示客户端ID，必选项，在授权服务器上进行申请

(B)：用户选择是否给客户端授权   

(C)：用户给予授权，授权服务器将用户导向客户端事先指定的重定向地址，同时附上一个授权码（一般是重定向地址后跟上一个code参数），例如：
```
redirect_url&code=xxx
```   

(D)：客户端附上授权码和重定向地址，去请求*Access Token*（令牌）   

这里如果是前后端分离项目，一般通过后端去请求令牌，因为后端往往也需要保存令牌。前端访问后端的接口，后端去请求令牌后再返回给前端，这样前后端都保存了令牌

请求令牌的接口格式如下：
```
https://ip:port/xxx?client_id=client_id&redict_url=redict_url&code=code&grant_type=authorization_code&secret=secret
```
其中：   
`grant_type`：表示使用的授权模式，必选项，此处的值固定为`authorization_code`     
`code`：表示上一步获得的授权码，必选项   
`redirect_uri`：表示重定向URI，必选项，且必须与A步骤中的该参数值保持一致   
`client_id`：表示客户端ID，必选项   
`secret`：密钥，在授权服务器上获取

(E)：授权服务器返回*Access Token*、*Refresh Token*（可选）和*expire time*（可选）   
`Access Token`：访问令牌，通过此令牌可以获取用户资源     
`Refresh Token`：更新令牌，通过接口调用，可以在不重新授权的情况下，获取下一个令牌   
`expire time`：表示令牌的有效时间

#### 最安全的模式——授权码
授权码模式是OAuth2协议流程最严密安全的一种实现方式，主要体现在以下几点：
1. 授权服务器直接提供重定向请求，用户可以在该网页上进行授权，没有密码泄露的风险   
2. 授权服务器先返回授权码，而不是令牌，可以避免令牌被窃取   
3. 即使授权码被截胡，攻击者因为没有拿不到密钥，所以无法获取令牌（密钥只在后端保存）
4. 客户端去请求令牌时，要传入上一步的授权码和重定向地址，授权服务器会与请求授权码传入的重定向地址进行比对。当二者一致时授权服务器才认为此次请求是合法的，并传回令牌

### 资源服务器
#### 资源访问
客户端根据令牌向资源服务器安全地访问资源   

例如，在你的个人App中，你想要用户完成登录以后才能进行其他操作（资源的访问）。那么你个人的App后端就相当于是资源服务器，前端是客户端   

在完成上述令牌的申请流程后，前端访问后端的所有接口，都需要附上令牌（*Header*方式）。后端通过校验令牌的合法性，从而安全地访问后端资源
#### 令牌校验
访问资源需要校验令牌的合法性，可以想到有两种实现方式：
1. 通过授权服务器的接口（端点）进行校验
2. 资源服务器自己维护每个用户的令牌，每次接口调用时去判断是否有效

##### 通过授权服务器校验
授权服务器提供接口，每次访问资源时，去调用该接口并附上令牌（*Header*方式）。该接口会返回令牌是否有效
通过这种方式可以很好的实现资源的安全访问（前提是要授权服务器能提供接口）  

如果后端是SpringBoot开发，可以通过配置资源服务器，并设置*filter chain*来指定哪些接口需要校验，这里不做展开

##### 通过资源服务器校验

当授权服务器不提供接口时，资源服务器需要自行保存令牌，并在每次必要请求时，判断令牌是否有效。常用的方式是通过*redis*来保存令牌

### 参考
[理解OAuth2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)   
[单点登录原理与简单实现](https://www.cnblogs.com/ywlaker/p/6113927.html)