---
title: 自动化部署配置webhooks
date: 2021-07-26 00:31:36
tags: ['GitHub', 'CI/CD']
categories: CI/CD
cover: https://images.pencilfirst.cn/img/89034783_p0.jpg
---
## 创建GitHub项目

首先在github上新建一个Hexo项目用于后续操作，然后再服务器上找一个文件目录clone项目，在服务器下拉代码需要先配置好自己的github ssh

### 服务器配置

#### 配置deploy.sh脚本

```bash
# !/bin/sh
PROJECT_DIR="项目目录"

echo 'start'

cd $PROJECT_DIR

git reset --hard origin/master && git clean -f
git pull && git checkout master

echo 'run npm script'

npm run build

echo 'finished'
```

#### 配置JavaScript文件

```javascript
// deploy.js
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({ path: '/', secret: 'root' })
// 上面的 secret 保持和 GitHub 后台设置的一致
function run_cmd(cmd, args, callback) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var resp = "";
  child.stdout.on('data', function(buffer) { resp += buffer.toString(); });
  child.stdout.on('end', function() { callback (resp) });
}
http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(7777)
handler.on('error', function (err) {
  console.error('Error:', err.message)
})
handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
    run_cmd('sh', ['./deploy.sh',event.payload.repository.name], function(text){ console.log(text) });
})
```

### GitHub配置

通过github中的webhooks来部署项目首先需要在github中新建一个项目在项目的settings中设置webhooks
![settings](https://images.pencilfirst.cn/img/6b189fafe967940b0dbccb9727e05ec.png)

![image-20210730074927669](https://images.pencilfirst.cn/img/image-20210730074927669.png)

点击Add webhook，保存后即可

如果github中的地址配置为**https**还需要在服务器配置反向代理

```nginx.conf
...

server {
	listen 		443 ssl;
	server_name	xxxxxx;
	...
	location / {
		root 	xxxxx/public
		index	index.html	index.htm
	}
	location /deploy {
		proxy_pass	http://127.0.0.1:7777;
	}
}

...
```

root 为hexo文件中的public目录

### 测试

![image-20210730081437136](https://images.pencilfirst.cn/img/image-20210730081437136.png)

根据触发记录的返回的错误信息来排查具体错误

