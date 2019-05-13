# 从零开始部署错误报告系统-Sentry

当我们的前端项目部署后，通常需要对线上情况进行观察报警，发现测试过程中未发现的问题。Sentry就是这样一套错误监控系统





## 选择Sentry的原因

* 开源，可以自行在服务器上搭建

* 支持各种主流前端框架（React，Vue，Angular）

* 部署快捷




  

## 准备工作

* 我的操作系统：Centos7.2，4核8G
* Sentry版本：9.1
* 注意点：
  * 实践证明系统内存需要>=3G，否则在执行 docker-compose run --rm web upgrade 创建数据库和超级管理员时会出错
  * 项目不要部署在root目录下
  * docker下载镜像过慢,  可参考：[修改docker镜像源的方法](https://blog.csdn.net/skh2015java/article/details/82631633)
  * 经过实践，发现[daocloud](https://www.daocloud.io/mirror)下载速度较快-http://f1361db2.m.daocloud.io




  
## 开始搭建

Sentry搭建方式有两种：1.通过Docker；2.通过Python。

官方推荐Docker部署。

#### Docker安装

1.安装管理repository及扩展包的工具 

 `yum install -y yum-utils`

2.添加yum源

 `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`

3.安装并启动Docker

```
yum -y install docker-ce
systemctl start docker
```

4.安装pip，用于下载docker-compose

```
yum install epel-release
yum install -y python-pip
```

5.安装docker-compose

 ` pip install docker-compose`

6.安装git

`yum install git`

#### 创建sentry服务

1.拉取sentry仓库

`git clone https://github.com/getsentry/onpremise.git`

2.创建服务，根据README文件中步骤一步一步来

```
cd onpremise

#新建docker本地数据库
docker volume create --name=sentry-data && docker volume create --name=sentry-postgres
#创建env配置文件
cp -n .env.example .env
#构建服务
docker-compose build
#生成秘钥
docker-compose run --rm web config generate-secret-key
#将生成的秘钥添加到.env的SENTRY_SECRET_KEY
vim .env
#创建数据库，创建超级管理员，作为登入sentry的账号，创建过程按提示一步步来
docker-compose run --rm web upgrade
#启动所有服务
docker-compose up -d
```

安装过程没出错的话，打开`http://ip:9000`就能看到Sentry的登入界面

#### 扩展

docker下所有的volume都在host机器上的 `/var/lib/docker/volumes`目录下

```
列出volumes：
$ docker volume ls -qf dangling=true
删除volumes：
$ docker volume rm $(docker volume ls -qf dangling=true)
```





   
   
## 前端项目接入Sentry

![界面](https://github.com/qsz/blog/blob/master/前端工程/files/sentry_1.png)

前端框架React，vue，Angular都可以接入Sentry

### 以vue为例

1.在sentry后台新建vue项目

2.进入项目，按下图选择，获取项目dsn

![获取项目dsn](https://github.com/qsz/blog/blob/master/前端工程/files/sentry_2.png)

3.在vue项目中接入Sentry SDK

```js
import * as Sentry from '@sentry/browser';
import * as Integrations from '@sentry/integrations';
Sentry.init({
  dsn: 'https://PublicKey@ip/ProjectID',    
  integrations: [
      new Integrations.Vue({
        Vue,
        attachProps: true,
      })
   ] 
});

```

4.尝试在vue项目中抛出一个错误，在sentry后台中即可看到报错信息





## 扩展

### 配置邮件提醒

每次在后台查看报错信息，时效性太低，需要配置邮件提醒功能

编辑docker-compose.yml文件，添加环境变量

```
x-defaults: &defaults
  restart: unless-stopped
  build: .
  depends_on:
    - redis
    - postgres
    - memcached
    - smtp
  env_file: .env
  environment:
    SENTRY_MEMCACHED_HOST: memcached
    SENTRY_REDIS_HOST: redis
    SENTRY_POSTGRES_HOST: postgres
    SENTRY_EMAIL_HOST: smtp
     # 这里添加一下额外的变量
    SENTRY_SERVER_EMAIL: 'XXXX@163.com' #邮箱
    SENTRY_EMAIL_HOST: 'smtp.163.com'   #host 
    SENTRY_EMAIL_PORT: 25               
    SENTRY_EMAIL_USER: 'XXXX@163.com'   
    SENTRY_EMAIL_PASSWORD: 'XXXXXXX'   #邮箱授权码
    SENTRY_EMAIL_USE_TLS: 'true'
...
 web:
    <<: *defaults
    ports:
      - '9000:9000'
      # 这里也需要添加一下
    volumes:
      - ./config.yml:/etc/sentry/config.yml
```

重启Sentry服务 ` docker-compose up -d`

#### 注意点

* 163邮箱SENTRY_EMAIL_PORT建议用25，qq邮箱SENTRY_EMAIL_PORT建议用587
* SENTRY_EMAIL_PASSWORD是邮箱授权码，不是邮箱密码。如何获取邮箱授权码可以百度
* 实践中发现qq不用配置也可以发送邮件，不知道为啥。。。




   
### 比邮箱时效性更强-接入钉钉机器人

1.修改requirements.txt文件

```
vim requirements.txt
redis-py-cluster==1.3.4 #相关依赖需要1.3.4版本的redis-py-cluster
```

2.修改Dockerfile

```
vim Dockerfile
#添加以下配置
RUN pip install git+https://github.com/anshengme/sentry-dingding
```

尝试过以下三个插件，在我的部署中只有第一个能正常发送钉钉信息

```
https://github.com/anshengme/sentry-dingding
https://github.com/evilbs/sentry-dingtalk
https://github.com/L3T/sentry-dingding
```

3.重新构建启动服务

```
docker-compose build --no-cache  #不使用缓存
docker-compose up -d
```

4.在sentry项目的所有集成页面找到插件，启用，并设置`Access Token(可在钉钉机器人设置中webhook选项找到)`

![plugin](https://github.com/qsz/blog/blob/master/前端工程/files/sentry_3.png)

![AccessToken](https://github.com/qsz/sentry-dingding/blob/master/docs/images/options.png)

#### 扩展：自定义sentry-钉钉通知内容

Sentry通知功能基于`sentry.plugins.bases.notify`，可以下载插件`https://github.com/anshengme/sentry-dingding`，改写plugin.py中的信息，实现自定义钉钉通知内容




   
### 添加成员

一个项目不可能只有一个超级管理员，我们可以通过邀请机制注册新账号。

1.在后台管理中添加成员

![invite](https://github.com/qsz/blog/blob/master/前端工程/files/sentry_4.png)

2.此时还没有新建账号，点击刚添加的成员，打开邀请链接，即可进行注册

![invite](https://github.com/qsz/blog/blob/master/前端工程/files/sentry_5.png)




   
### 接入小程序

Sentry没有提供小程序相关SDK，搜索发现[有赞raven-weapp](https://github.com/youzan/raven-weapp)已经改写出了JavaScript版本的SDK，引用之

onerror中拦截未catch到的错误

```js
onError(msg) {
    Raven.captureException(msg, {
      level: 'error'
    })
}
```




   
### 设置时区

Sentry默认时区不是中国时区，改之

```
vim sentry.conf.py
#添加以下配置
SENTRY_DEFAULT_TIME_ZONE = 'Asia/Shanghai'
```




&nbsp;
### sourcemap

待解决后更新...
