#### 创建文档
```bash
hexo new title --path /path/filesaved --category ca_name
```

#### 发布&部署
```bash
hexo clean && hexo g && hexo d
```

##### github图片地址

```text
https://github.com/xiaokunliu/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/reactor/pattern_title.jpg?raw=true
https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/reactor/pattern_title.jpg
```

#### 写作基本要求

```text
1. 中英文之间需要空格隔开
2. 段落两端对齐
3. 字体15px，微软雅黑
4. 段落与段落之间要用回车键隔开
```

#### 发布部署出错
```text
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Error: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

    at ChildProcess.<anonymous> (/opt/data/xiaokunliu.github.io/websites/node_modules/hexo-util/lib/spawn.js:37:17)
	    at ChildProcess.emit (events.js:315:20)
	    at maybeClose (internal/child_process.js:1021:16)
	    at Socket.<anonymous> (internal/child_process.js:443:11)
	    at Socket.emit (events.js:315:20)
	    at Pipe.<anonymous> (net.js:674:12)
```

#### 解决方案
```text
## 生成ssh的密钥并粘贴到github对应的仓库上
ssh-keygen -o

cat ~/.sshd/id_rsa.pub
```

