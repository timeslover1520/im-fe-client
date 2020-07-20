<img width="464" src="https://i.loli.net/2020/07/01/AmsnawZ29RbUqk8.png">

即时通讯应用, 包含[服务端](https://github.com/hezhongfeng/im-server)、[管理端](https://github.com/hezhongfeng/im-fe-admin)和[客户端](https://github.com/hezhongfeng/im-fe-client)

现已部署上线，欢迎体验[客户端](https://im-client.hezf.online/)和[管理端](https://im-admin.hezf.online/)

## 介绍

使用 @vue/cli 搭建，IM 服务的客户端，UI 部分使用了 Vant

<img width="300" src="https://i.loli.net/2020/07/15/fO2naUmPluYRsBd.png">

<img width="300" src="https://i.loli.net/2020/07/15/5lvZtFBsPrUnHgN.png">

## 功能简介

1. 注册，登录，个人、群组聊天，个人信息编辑等基础功能
2. 申请添加好友和申请入群
3. 表情，图片，视频，定位信息支持
4. 聊天会话列表记录
5. 消息记录（微信的消息记录真实一言难尽）
6. 支持多点同时登录
7. 百度 UNIT 机器人自动聊天
8. 支持 github 一键登录
9. 管理端，进行角色和权限的管理，群状态管理（我也当一回马化腾）

## 项目介绍

这个项目主要是演示一个 client（类似微信）的基础应用，具有注册、登录、编辑个人信息、添加好友、申请入群、聊天等功能，需要配合后端使用

## 项目整体

Vue 全家桶的项目大家很熟悉的了（这点我觉得比 React 好一些，没有了那么多的选择困难），这里说下扩展的几个地方：

1. 封装了一个 http 插件，主要目的是统一参数的传递和对后端反馈的统一响应
2. 自定义全局 scss 变量，不需要在使用的时候手动引用
3. 将常用的$urls,$http 和 \$checkLogin 注入到 Vue 原型上，方便使用
4. 使用 vue-page-stack 做页面的栈管理工具，使用黄老师的[better-scroll](https://github.com/ustbhuangyi/better-scroll)做滚动处理（这个很重要）
5. 在路由的全局守卫 beforeEach 上面添加了对目前登录状态的确认，这点做法和原生 App 不一致，需要注意

## IM 功能简介

首先引入 socket.io-client 这个包，通过名字可以知道是 socket.io 的客户端部分，下面说一下用法：

### 链接服务端

想要和服务端进行收发信息，先要建立链接

```
import io from 'socket.io-client';
// 首先需要链接上后端的 socket.io，query是链接的参数
this.socket = io('http://127.0.0.1:7001', {
  query: {
    scene: 'im',
    userId: '1'
  }
});

// socket.on 就是监听事件，connect就是链接上了服务端
this.socket.on('connect', () => {
  console.log('socket连接成功！');
  // ....
});
```

在本项目中链接成功后我开始请求会话列表和消息记录，这样可以保证在获取消息的时候链接已经成功

### 收发信息

在和服务端进行链接后就可以进行收发信息了，很简单：

```
// 发送消息，message是自己定义的消息体
this.socket.emit('/v1/im/new-message', message);


// 有新消息收到
this.socket.on('/v1/im/new-message', message => {
  // 自己进行一系列的处理
});
```

收发消息这块可以和后端结合在一起看，前端即使是发送自己的消息也是在收到后台的群发后才会显示，也就是说如果断开了链接，那么自己的消息也发不出来

## 表情消息

输入的时候是`[酷]`这种的纯字符串，保存的时候也是字符串，在显示的时候替换为对应的图标，这里说下为什么不想办法在输入框里面也替换为对应的图标。老板原先要求过这么做，我也尝试了一下，结果就是这玩意是个大坑。现如今我接触过的好像只有微博的输入框显示的是图标而不是字符串，也是引出了很多的问题：

1. 假的输入框操作不灵活，总有卡顿
2. IOS 在使用光标快速移动的时候，会很卡，因为我怀疑他们也是隐藏了一个原生输入框，然后插入
3. 在一些安卓手机中会一直提示复制粘贴的权限，原因也是想造个假的输入框

但是使用字符串的话就很好了（微信和钉钉都是），所有原生的特性都能够用，前端也好处理（重点）

## 会话

以上可以看出来 socket.io 的使用多么的方便了，首先建立链接，然后就可以进行收发消息了，结合我们的 IM 场景，做了如下的安排：

1. 通过 vuex 维护一个会话列表，即 conversationList
2. 和服务端建立链接成功后向后端请求会话列表，成功后请求每一个会话的聊天记录
3. 点击会话，会进入到会话详情，激活这个会话，输入信息发送，将信息发送到服务器
4. 由于已经对消息有了监听，所以服务器将上一步发送的消息广播后自己也可以收到，然后 push 进会话的消息列表，通过对比发送用户知道是不是自己的消息
5. 服务端主动发来的消息也一样，都由 `/v1/im/new-message` 统一监听

## 滚动

说下怎么存储的前一页的滚动条，在 github 上也经常会有人询问如何保持滚动条的位置和状态，当我们从某一可滚动页面（例如这里的会话列表页面）进入到会话详情再回退的时候，可以发现滚动条状态是被保存了的。

页面使用了我自己的 UI 栈管理插件 vue-page-stack，回退的时候恢复了 Vue 的虚拟 dom，但是并不能记住原生 dom 的滚动条状态，这时候需要辅助工具出马了。滚动处理使用了黄老师的 better-scroll 2.0，bs 在内部记录了滚动的状态，所以在恢复虚拟 dom 的时候 bs 也一并被恢复了，滚动自然也恢复了。

### 会话消息记录的滚动

这个滚动耗费了我不少心思，因为绝大多数滚动都是上拉加载，但是咱们这个场景下是下拉加载，这就导致了一个很严重的问题：上拉加载的时候，加载的内容在滚动区域的下方出现，加载之后，我们将数据添加到列表，由 Vue 等负责渲染新加载的内容，我们继续上拉就可以继续查看

但是下拉加载不一样，下拉加载后出现的内容在滚动区域的上方，不做任何处理的话会直接跳到新加载内容的最上方，这就出问题了。

我的处理办法就是：

1. 通过接口获取到加载信息后首先标记（shouldScroll）第一条信息，也就是最终我们的视角要看到的内容
2. messageList 更新后，Vue 会更新数据和视图
3. MessageItem 组件 mounted 后，这时候已经完成了视图的渲染，通过检查标记（shouldScroll）,通知父容器滚动到刚才标记的位置，也就是加载的第一条信息处，这样也就把渲染和滚动做到一起了

效果还可以，如下：

![](https://i.loli.net/2020/07/09/SXu9vUTA6rtEIJj.gif)

## 优化 build 体积

由于我的服务器带宽太弱，所以想着尽可能的使用 CDN，使用了一些扩展，这样对服务器压力小了很多，这部分内容属于 webpack 部分

```
// index.html
<script src="https://api.map.baidu.com/getscript?v=3.0&ak=ZHjk59sSOpM1eNWgNWyj9zpyAFTHdL5z"></script>
<script src="https://cdn.jsdelivr.net/npm/vue@2.6.11/dist/vue.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vue-router@3.3.4/dist/vue-router.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vuex@3.5.1/dist/vuex.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vant@2.9/lib/vant.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/xgplayer@2.9.8/browser/index.js"></script>

// vue.config.js
externals: {
  BMap: 'BMap',
  vue: 'Vue',
  'vue-router': 'VueRouter',
  vuex: 'Vuex',
  vant: 'vant',
  xgplayer: 'Player'
}
```

这也导致了我的 https 显示是不安全的，因为 SameSite ，但是也没别的办法了

## 部署

前端资源使用 nginx 管理，由 nginx 做反向代理，index.html 在前端不做缓存，js 和 css 等静态文件做了一个月的强缓存和 gzip 压缩，剩下的/api、/public 和/socket.io 是需要转发到服务器的，我的服务器就运行在 http://127.0.0.1:7001 ，这里需要注意设置 http 可升级为 websocket

https 的证书是在 https://freessl.cn/ 上面免费申请的证书，在 nginx 上进行了配置

```
server {
  listen 80;
  server_name im-client.hezf.online;

  # Load configuration files for the default server block.
  include /etc/nginx/default.d/*.conf;

  location / {
    root /data/static/im-client;
    index index.html;
    try_files $uri $uri/ /index.html;
  }

  location ~* \.(html)$ {
    root /data/static/im-client;
    access_log off;
    add_header Cache-Control no-store;
  }

  location /static {
    access_log off;
    root /data/static/im-client;
    gzip on;
    gzip_buffers 32 8K;
    gzip_comp_level 6;
    gzip_min_length 100;
    gzip_types application/javascript text/css text/xml;
    gzip_disable "MSIE [1-6]\."; #配置禁用gzip条件，支持正则。此处表示ie6及以下不启用gzip（因为ie低版本不支持）
    gzip_vary on;
    add_header Cache-Control max-age=2592000;
  }

  location /api {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto http;
    proxy_set_header X-NginX-Proxy true;
    client_max_body_size	100m;
  }

  location /public {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size	100m;
  }

  location /socket.io {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size	100m;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html {
  }
}

server {
  listen 443 ssl;
  server_name im-client.hezf.online;
  ssl_certificate /etc/nginx/conf.d/hezf-online/im-client.hezf.online_chain.crt;
  ssl_certificate_key /etc/nginx/conf.d/hezf-online/im-client.hezf.online_key.key;
  ssl_session_timeout 5m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
  ssl_prefer_server_ciphers on;

  # Load configuration files for the default server block.
  include /etc/nginx/default.d/*.conf;

  location / {
    root /data/static/im-client;
    index index.html;
    try_files $uri $uri/ /index.html;
  }

  location ~* \.(html)$ {
    root /data/static/im-client;
    access_log off;
    add_header Cache-Control no-store;
  }

  location /static {
    access_log off;
    root /data/static/im-client;
    gzip on;
    gzip_buffers 32 8K;
    gzip_comp_level 6;
    gzip_min_length 100;
    gzip_types application/javascript text/css text/xml;
    gzip_disable "MSIE [1-6]\."; #配置禁用gzip条件，支持正则。此处表示ie6及以下不启用gzip（因为ie低版本不支持）
    gzip_vary on;
    add_header Cache-Control max-age=2592000;
  }

  location /api {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-NginX-Proxy true;
    proxy_redirect off;
    client_max_body_size	100m;
  }

  location /public {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size	100m;
  }

  location /socket.io {
    proxy_pass http://127.0.0.1:7001;
    proxy_connect_timeout	3;
    proxy_send_timeout 30;
    proxy_read_timeout 30;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size	100m;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  error_page 500 502 503 504 /50x.html;
  location = /50x.html {
  }
}
```
