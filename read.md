##### 阅读文档

> hexo 官网

```text
https://hexo.io/zh-cn/docs/
```

> hexo 主题

```text
https://hexo.io/themes/

## 使用主题
https://github.com/litten/hexo-theme-yilia
https://github.com/iissnan/hexo-theme-next
https://github.com/liuyib/hexo-theme-stun
https://github.com/KevinOfNeu/hexo-theme-xoxo
```

> 后续使用Travis CI自动构建（TODO）

```text
## 参考： https://juejin.im/post/5a1fa30c6fb9a045263b5d2a

## 先配置GitHub Token
https://github.com/settings/tokens

## 配置Travis-CI
## 1. 将github关联的项目绑定上去
## 2. 将配置github的token到ci的环境配置中

## 配置并编写脚本到.travis.yml文件
## 部署文件单独存储到_config.yml文件的deploy节点

##上述的两个文件需要在项目的根目录下
## 一旦配置好之后就推送到分支上,ci会自动触发构建任务执行

## 需要github的rsa权限
使用docker的dev的密钥
cat ~/.ssh/id_rsa.pub

## 安装ruby to 2.6
## apt-get install gcc libffi-dev make ruby-dev
## 安装travis的客户端
https://github.com/travis-ci/travis.rb#installation

## use command 
gem install travis --no-document

## 加密rsa密钥
travis version
travis login --org
## finish login with github

## 生成密钥
travis encrypt-file ~/.ssh/id_rsa

## 输出结果
encrypting /root/.ssh/id_rsa for xiaokunliu/xiaokunliu.github.io
storing result as id_rsa.enc
DANGER ZONE: Override existing id_rsa.enc? |no| yes
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

	openssl aes-256-cbc -K $encrypted_9b2d7e19d83c_key -iv $encrypted_9b2d7e19d83c_iv -in id_rsa.enc -out ~\/.ssh/id_rsa -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add id_rsa.enc to the git repository		
Make sure not to add /root/.ssh/id_rsa to the git repository.
Commit all changes to your .travis.yml.


## 将输出密钥添加到.travis.yml中


## 在.travis.yml中添加密钥脚本
```

> 重新安装需要注意事项

```bash
## 先安装node 以及 npm

## 然后安装hexo-cli
npm install -g hexo-cli
hexo version

## 删除之前的node_modules
rm -rf websites/node_modules

## 重新安装依赖
npm install 
npm install hexo-deployer-git --save

## 如果没有git需要安装git
apt-get install -y git

## 配置
git config --global user.name "xiaokunliu"
git config --global user.emial "liuxkmail@gmail.com"

## 生产密钥
ssh-keygen -t rsa

## 获取密钥
cat ~/.ssh/id_rsa.pub

## 拷贝到项目的deploy keys
https://github.com/xiaokunliu/xiaokunliu.github.io/settings/keys
```
