# nodejs实现鉴权

​		相信一些刚接触前端开发的同学，会遇到类似这种问题：与后端同学约定某个系统需要登入才能开发，OK，那我们就登入，输入用户名，密码，完成。这时产品同学找我们开会，等我们开完会，再继续开发时，发现接口返回401，需要身份验证。。what？我们不是登入过了吗？这时，后端同学会告诉我们，你的Cookie里token过期了。what？一脸懵逼，token是啥？token过期了又是啥？

​		这个问题其实就是鉴权的知识，本文我们就来学习一下。

​		（本文的node版本：v10.15.3,	koa版本：v2.7.0）



## 鉴权的常见实现方式

​		鉴权通俗的将就是指验证用户是否拥有访问系统的权利。常见的实现方式有以下几种：

* session-cookie
* Token 验证
* HTTP Basic Authentication
* OAuth(开放授权)



### session-cookie

这种验证方式是比较传统的实现方式，实现流程大致如下：

1. HTTP是无状态的，它并不记录用户的身份，后台为了能够识别用户身份，需要在数据库中储存一份用户的登入信息。前端登入后，后端把一个session标识符返回给前端，并将该session标识符保存到浏览器的cookie中

2. 前端在下次发送请求时，把携带session标识符的cookie发送给后端

3. 后端通过该标识符，从数据库中查找用户如用户信息、授权权限等等信息，由此判断用户是否登入以及是否有权限

用图表示为

![session-cookie](./files/session-cookie.png)

#### node实现

接下来用nodejs的koa框架来模拟实现`session-cookie`验证。需要用到[koa-session](https://www.npmjs.com/package/koa-session)库

```javascript
const Koa = require('koa');
const Router = require('koa-router');
const session=require('koa-session'); 

const app = new Koa();
const router = new Router();

const localRedis = {}; // 模拟一个本地的redis数据库
app.keys = ['some secret hurr'];
const CONFIG = {
    key: 'koa:sess', 
    maxAge: 86400000,
    autoCommit: true,
    overwrite: true,
    httpOnly: true, 
    signed: true, 
    rolling: false, 
    renew: false, 
};
app.use(session(CONFIG, app))
// 登入接口
router.get('/loginCookie', (ctx, next) => {
    const name = ctx.query.name
    ctx.session.name = name;  
    // ctx.session.name作为session标识符，将用户登陆信息保存到数据库中
    localRedis[ctx.session.name] = {name};
    return ctx.body = '登入成功'
});
// 验证是否登入
router.get('/checkCookie', (ctx, next) => {
    if(!ctx.session.name) {
        return ctx.body = '验证失败'
    } 
    // 用session标识符去数据库中获取用户登陆信息
    const name = localRedis[ctx.session.name].name
    if(!name) {
        return ctx.body = '获取用户信息失败'
    }
    return ctx.body = `${name} 身份验证成功!`
});


app
  .use(router.routes())
  .use(router.allowedMethods()); 
app.listen(3000);
```

我们首先打开`http://localhost:3000/checkCookie`, 发现会提示"验证失败"。

接着请求登入接口并模拟输入用户名`http://localhost:3000/loginCookie?name=qqq`，然后再打开`http://localhost:3000/checkCookie`, 返回 “qqq 身份验证成功!”。此时说明我们鉴权通过。

我们查看浏览器cookie信息

![session-cookie2](./files/session-cookie2.jpg)

此时删除cookie，我们在访问`http://localhost:3000/checkCookie`, 提示`验证失败`

由此可知登入标示信息的确是存入到cookie中。

通过观察`session-cookie`的实现，我们可以发现存在以下不足

* 依赖浏览器cookie，而cookie容易受到跨站点请求伪造攻击
* 需要把session储存到数据库中做持久化缓存，占用了大量的存储空间



### Token 验证 

`Token 验证`实现流程大致如下:

1. 用户登入
2. 后台验证通过后，根据用户登入信息会生成一个有时效性的token，并将此token返回给用户
3. 用户将此token存储在本地
4. 下次每个请求都会带上此token，后段通过解析token，验证token中的包含的用户信息，以此完成权限验证

上述过程中的token是根据`JSON Web Token（缩写 JWT）`实现

实际的jwt数据类似下面

![jwt](./files/jwt.jpg)

由三部分组成

> ```javascript
> Header.Payload.Signature
> ```

三个部分依次表示

> - Header（头部）
> - Payload（负载）
> - Signature（签名）

这三部分通过加密包含了用户的信息。` JWT`的具体介绍大家可以参考下面的链接`阮一峰老师的文章`

#### node实现

需要用到[jsonwebtoken](https://www.npmjs.com/package/jsonwebtoken)库（ jsonwebtoken是node版本的JWT的实现）

```javascript
const jwt = require('jsonwebtoken');

const secret = 'shhhhh' // 自定义一个密钥，用于加密与解密
// 生成token
function signToken(name) {
    const expire = Date.now() + 1 * 60 * 1000; // 设置过期时间，此处设置成一分钟
    const data = { name: name, expire }        // 设置用户信息
    return jwt.sign(data, secret)
}
// 解析token
function verifyToken(token) {
    return jwt.verify(token, secret) || {}
}
// 登入，生成token api
router.get('/signJwt', (ctx, next) => {
    return ctx.body = signToken(ctx.query.name);
});
// 鉴权，解析token api
router.get('/verify', (ctx, next) => {
    const token = ctx.query.token
    const decoded = verifyToken(token);
    if(decoded && decoded.name && decoded.expire) {
        // 判断token是否过期
        if(Date.now() > decoded.expire) {
            return ctx.body = 'token过期'
        } 
        return ctx.body = `${decoded.name} 登入成功`
    }
    return ctx.body = '登入失败'
});
```

首先我们模拟用户登入, 访问`http://localhost:3000/signJwt?name=qqq`并带上用户名。接口返回如下token，可看到该段token由三部分组成，符合jwt的数据格式

> ```
> eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoicXFxIiwiZXhwaXJlIjoxNTYzMDAxMzk3MTg1LCJpYXQiOjE1NjMwMDEzMzd9.EbIr9ZuUmBqfM61LhzxiAPM2muw2n9Db5Q9TrBhTJTY
> ```

接下来访问鉴权接口，并带上token`http://localhost:3000/verify?token=xxxxx`，提示“qqq 登入成功”。可以看到，我们鉴权接口通过解析token，成功返回了用户名。

接下来我们等待一分钟后，再次访问鉴权接口，这次提示“token过期”。这是因为我们设置的token过期时间为1分钟，通过解析token，发现已经过期。

与`session-cookie` 相比，`Token 验证 `不局限于cookie，可以规避CSRF攻击；token的承载的信息更丰富，可扩展性强；本身已经包含用户信息，不需要储存到数据库。但`Token 验证`中需要频繁地对token进行加密解密，因此比`session-cookie`更消耗服务器性能

**参考**

- [JSON Web Token 入门教程](http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

### HTTP Basic Authentication

在http请求中，用户的登入信息除了携带到`header`的`Cookie`中，还可以携带到`Authorization`中，如下所示

> Authorization: Basic Y2h5aW5ncDoxMjM0NTY= 

该实现流程如下：

服务器端向客户端返回 [`401`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/401)Unauthorized，未被授权的） 状态码，并在  [`WWW-Authenticate`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/WWW-Authenticate) 首部提供如何进行验证的信息，其中至少包含有一种质询方式。之后有意向证明自己身份的客户端可以在新的请求中添加 [`Authorization`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Authorization) 首部字段进行验证，字段值为身份验证凭证信息。通常客户端会弹出一个密码框让用户填写，然后发送包含有恰当的 `Authorization`  首部的请求

常见的验证方案就是-基本验证方案，即`Basic Authentication`

Basic认证通过核对用户名、密码的方式，来实现用户身份的验证,  并且使用 base64 算法进行编码,	即如下形式

```
Authorization Basic Buffer.from('userid:password').toString('base64')
```

#### node代码实现

需要借助[koa-basic-auth](https://www.npmjs.com/package/koa-basic-auth)库

```javascript
const auth = require('koa-basic-auth');
router.get('/basicAuth',  (ctx, next) => {
    try {
        return ctx.body = `身份验证成功!`;
    } catch (err) {
        // 401表示未授权
        if (401 == err.status) {
          ctx.status = 401;
          ctx.set('WWW-Authenticate', 'Basic');
          ctx.body = 'cant haz that';
        } else {
          throw err;
        }
    }
});

// 此处设置只有 用户名和密码 是qqq，123456的用户才有权限
// Buffer.from('qqq:123456').toString('base64') = "cXFxOjEyMzQ1Ng=="
app.use(auth({ name: 'qqq', pass: '123456' })); 
```

浏览器访问`http://localhost:3000/basicAuth`， 出现提示登入的弹窗。

![authentication](./files/authentication.png)

只有输入

> qqq
>
> 123456

才会提示"身份验证成功!"

接下来我们使用 base64 算法进行编码对用户名密码编译

```javascript
Buffer.from('qqq:123456').toString('base64') = "cXFxOjEyMzQ1Ng=="
```

打开`postman`访问`http://localhost:3000/basicAuth`，并设置头部

`Authorization Basic cXFxOjEyMzQ1Ng==`

可以看到结果

![Authenticationpostman](./files/Authenticationpostman.jpg)

如果我们删除`Authorization`, 再次请求，就会提示“Unauthorized”

**参考**

* [一文读懂HTTP Basic身份认证](https://juejin.im/entry/5ac175baf265da239e4e3999)
* [HTTP 身份验证](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication)



### OAuth(开放授权)

`OAuth`是一个关于授权（authorization）的开放网络标准,	用来授权第三方应用，获取用户数据

> 简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用

在我们生活中，最常见的就是QQ、微信、微博等可以授权第三方应用，直接用QQ、微信、微博登入。

接下来就举一个github授权的例子，来看看整个OAuth(开放授权)的整个流程

1. 我们登入自己的github账号，依次点击 `settings ->Developer settings->OAuth Apps`， 点击`new OAuth Apps`。创建一个OAuth应用。

![oaut](./files/oath1.jpg)

2. 创建完成后，可以得到 Client ID和Client Secret，在后续的授权中需要用到

![oauth2](./files/oauth2.jpg)

3. 编写代码

   ```javascript
   const axios = require('axios');
   
   const githubConfig = {
       clientID: '50af179e2b9e6394736e',
       clientSecret: 'd0dd21b935b3249596ee953e2f9bece087e829a5'
   }
   
   // 访问/github 会重定向到Github登录链接
   router.get('/github',  (ctx, next) => {
       // loginPath是Github提供的登录链接
       const loginPath = `https://github.com/login/oauth/authorize?client_id=${githubConfig.clientID}`
       ctx.redirect(loginPath);
   });
   
   // Github登录链接授权后的跳转地址，会携带授权码`code`
   router.get('/home',  async (ctx, next) => {
       const params = {
           client_id: githubConfig.clientID,
           client_secret: githubConfig.clientSecret,
           code: ctx.query.code
       }
       // 获取access_token令牌的接口
       const accessUrl = 'https://github.com/login/oauth/access_token';
       // 获取用户信息的接口
       const userUrl = 'https://api.github.com/user?access_token=';
       const access = await axios.post(accessUrl, params)
       const access_token = access.data.split('&')[0].split('=')[1]
       // 根据access_token令牌获取用户信息
       const user = await axios.get(`${userUrl}${access_token}`)
       return ctx.body = `hello ${user.data.login}`
   });
   ```

4. 浏览器访问`http://localhost:3000/github`,会跳转到登入授权地址![oauth3](./files/oauth3.jpg)

5. 点击授权后，会跳回到`http://localhost:3000/home?code=xxx`。然后根据url地址中携带的授权码，依次获得令牌、用户信息。可以看到返回信息`hello xxx`，表示第三方成功获得了用户的授权信息。

6. 当我们再次访问`http://localhost:3000/github`, 不需要再次授权，会直接重定向到`http://localhost:3000/home`, 并返回信息`hello xxx`。

**参考**

* [第三方登入例子-GitHub授权登入](https://www.jianshu.com/p/a9c0b277a3b3)
* [OAuth 2.0 的一个简单解释](http://www.ruanyifeng.com/blog/2019/04/oauth_design.html)



## 小结

对常见的鉴权实现方式就学习到这里，其中的知识点，如`JSON Web Token`等可以继续深挖。了解鉴权实现方式，相信以后在和后端同学对接中，我们会显得游刃有余。