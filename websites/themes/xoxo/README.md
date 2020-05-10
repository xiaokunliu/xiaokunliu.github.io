#### hexo-theme-xoxo

##### _config.yml的配置
```yml
# Custom Config
menu:
  home: /
  archives: /archives
  tags: /tags
  search: /search
  about: https://0xff000000.com
  rss: /atom.xml

search:
  path: search.xml
  field: post

#RSS feed https://github.com/hexojs/hexo-generator-feed
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content_limit: 140

sitemap:
    path: sitemap.xml

disqus_shortname: XXX

google_analytics: XXX
baidu_analytics: FuckUBaidu

#footer
index_page: https://blog.0xff000000.com
main_site: https://0xff000000.com
github: https://github.com/KevinOfNeu
```

##### _config.yml的配置
```bash
## 安装依赖
## package.json目录下安装
node install

npm install hexo-generator-feed --save
```

##### 修改头部以及尾部的图片链接
```text
## 修改图片
## layout/partials/head.js
## layout/partials/footer.js

## 修改分享
## layout/partials/share.js
```

