# npm私服搭建的3种方式

在工作中，我们经常会开发公共的模块，但又不希望对外发布，此时我们可以搭建npm私服。我尝试了以下三种方式，大家可以对比采取一种进行搭建。

## gitlab + npm

每个公司都有自己的代码管理仓库，比较常见的是gitlab。通过gitlab可以快速的搭建自己的npm私服

### 使用方式

* 安装和更新: `npm install —save git+ssh://xxx.git`

*  npm 包提交: `git push`
* 注意点：
  * npm 版本需要 6.x，npm 5版本在实践中发现，每次`npm install`安装公共库(如`vue`)，`node_modules`中已经安装的私有库会被删除
  
  * `npm update`不会更新私有库
  
  * 如果老项目还是npm 5版本 ，升级后部分老的依赖包存在兼容性问题
  

如果npm升级成本不大，可以采用该方法，无需申请服务器，搭建最快捷

<br/>

## docker + verdaccio

verdaccio 目前有人在积极维护，利用docker搭建也很方便

### 搭建方式

按上一篇[在Centos环境中从零开始部署错误报告工具-Sentry](https://github.com/qsz/blog/blob/master/前端工程/Sentry部署.md)中的方式先下载安装docker + docker-compose + git , `verdaccio`版本 4.x-next

1.拉取镜像

`docker pull verdaccio/verdaccio`

2.从git上获取配置文件

`git clone https://github.com/verdaccio/docker-examples.git`

选择 `docker-local-storage-volume`版本

3.进入目录

`cd docker-local-storage-volume`

构建并启动项目

`docker-compose up -d`

4.打开 `http://ip:4873`, 看到如下界面, 则说明搭建成功

![verdaccio界面](https://github.com/qsz/blog/blob/master/前端工程/files/verdaccio_1.png)

### 修改配置文件

1.打开配置文件

```
cd /conf
vim conf.yaml
```

2.修改配置

```
# 使用淘宝的cnpmjs镜像，加速
uplinks:
  npmjs:
    url: https://registry.npm.taobao.org/
    
packages:
  '@jota/*':
      access: $all
      publish: $all

  '@*/*':
    # scoped packages
    access: $all  #拉取权限，三个值"$all"(所有用户), "$anonymous", "$authenticated"（以注册用户）
    publish: $authenticated #发布权限 可改为配置的用户名
    proxy: npmjs

  '**':
    access: $all
    publish: $authenticated
    proxy: npmjs
```

### 添加用户

1.打开文件

```
cd /conf
vim  htpasswd
```

2.添加用户

打开[该加密网站](http://www.htaccesstools.com/htpasswd-generator/)，输入用户名和密码，将生成的加密码粘贴到底部

3.重启docker容器

`docker-compose restart `

### 发布私有库

#### 下载nrm, 用于管理npm 源

1.`npm install -g nrm`

2.设置私有库源

```
nrm add verdaccio http://ip:4873 // 给verdaccio源设置别名
nrm use verdaccio // 切换源
```

#### 开始发布

1.新建一个测试库

```
touch npmTest
cd npmTest
npm init
```

2.添加用户`npm adduser`，输入登录名，密码以及邮箱

3.登入`npm login`

4.发布`npm publish`

### 遗留问题

在实践中发现，`npm publish`总是失败，提示

```
npm ERR! code E500
npm ERR! 500 Internal Server Error - PUT http://xxx:4873/npmtest - internal server error
```

待解决。。。

<br/>

## docker + Nexus

因为verdaccio私有库，在`npm publish`总是失败, 转向`Nexus`

### 开始搭建

1.下载配置文件

`git clone https://github.com/sonatype/docker-nexus3.git`

2.构建容器

```
cd /docker-nexus3
docker-compose build
```

3.运行

`docker-compose up -d`

4.打开 `http://ip:8081`, 看到如下界面, 则说明搭建成功

![Nexus界面](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_1.png)

### 创建仓库

默认账号

> 账号：admin
>
> 密码：admin123

选择设置，依次选择`Blob Stores`, `Repositories`

![设置界面](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_2.png)

新建`Blob Stores`,用于存放仓库文件

![Blob Stores](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_3.png)

创建Repositories, npm仓库有以下三种类型

![Blob Stores](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_4.png)

#### 创建代理仓库

当本地没有仓库时，代理仓库会去请求远程仓库，并将远程仓库缓存到本地

![代理仓库](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_5.png)

**Name：**仓库名字

**Remote storage**：远程仓库地址, 可以选择淘宝源`https://registry.npm.taobao.org`

**Blob store**：上一步创建的store

#### 创建私有仓库（hosted）

私有仓库的地址，用于管理私有的npm包

![私有仓库](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_6.png)

**Name**: 仓库名字

**Blob store**：上一步创建的store

#### 创建npm组（group）

npm组可以添加私有和代理仓库，这样通过group可以同时访问私有和代理仓库

![npm组](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_7.png)

<br/>

### 设置Realms

发布私服需要设置权限, 点击Realms，添加` npm Bearer Token Realm `

![npm组](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_8.png)

### 发布一个私有库

点击仓库名的copy选项，可以获取npm register地址

![npm组](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_9.png)

```
npm config set registry http://xxxxx:8081/repository/npm-gruop/   # 设置npm地址
npm init  # 新建项目
npm login --registry=http://xxxx:8081/repository/npm-private/     # 登入到私有仓库，用admin登入
npm publish --registry=http://xxxx:8081/repository/npm-private/   # 发布包
```

可以修改私有库的package.json文件

```
 "publishConfig": {
    "registry": "http://xxxx:8081/repository/npm-private/"
  },
```

这样每次只需要`npm publish`即可

### 添加用户

![添加用户](https://github.com/qsz/blog/blob/master/前端工程/files/Nexus_10.png)

### 与verdaccio，gitlab对比

##### 不足

* 搭建比gitlab, verdaccio稍显复杂

##### 优势
* verdaccio在安装私有库与公共库的时候，需要来回切换npm registry，Nexus则不用

* gitlab方式 在npm 5.x时，`install`会存在私有库被删除的情况，Nexus则不会

* 通过Nexus代理库, 可以缓存远程库，提高下载速度

<br/>
经过实践，我最终采用Nexus搭建公司的前端npm私服。大家则可以根据自身与公司的实际情况，选择一种方式进行搭建

