# Blog From Hexo To Hugo


最近逛Gayhub看到Hugo，不管框架怎么样，先进官网看主题，颜狗瞬间被皮肤所吸引，回头默默看了下自己的博客，不就是荣耀水晶皮肤和没皮肤的区别么！再浏览文档和上手体验后，竟然还体会到了+10点攻击力的感受，安装、配置、部署、主题、MD即显、Actions配套等处处体现Hugo的简洁、灵活和高效！

随即开始了迁徙之路（主要是想换个皮提高下写Blog的动力）...

<!-- more -->

## 安装

参照官档：

[https://gohugo.io/documentation/](https://gohugo.io/documentation/)

同时Hugo Demo站也提供了详尽文档：

[https://hugoloveit.com](https://hugoloveit.com)

## 主题

个人喜欢黑白极简：

[https://themes.gohugo.io/loveit/](https://themes.gohugo.io/loveit/)

## 配置

着重说下三个小坑：

1. 搜索功能

   未使用开箱即用的lunr（听说对中文不友好），直接用algolia，除了根据[Hugo文档](https://hugoloveit.com/theme-documentation-basics/#5-search)配置`config.toml`对应部分外，为了后续实现Github Actions自动化部署，而使用[atomic-algolia](https://www.npmjs.com/package/atomic-algolia#npm-scripts)，额外需要通过`npm init`生成`package.json`后，在其中添加：

	```json
	...
    "scripts": {
      "test": "echo \"Error: no test specified\" && exit 1",
      "algolia": "atomic-algolia"
    },
    ...
	```
	
	 Actions中workflow配置：

	 ```yml
   - name: Setup Node
     uses: actions/setup-node@v2.1.0
     with:
       node-version: '12.x'
    
   # ... #
    
   - name: Setup Algolia
     env:
       ALGOLIA_APP_ID: 63ROI1DYLO
       ALGOLIA_ADMIN_KEY: ${{ secrets.ALGOLIA_ADMIN_KEY }}
       ALGOLIA_INDEX_NAME: houugen_blog
       ALGOLIA_INDEX_FILE: public/index.json
     run: npm install atomic-algolia && npm run algolia
    ```

{{< admonition note >}}
ALGOLIA_ADMIN_KEY为Algolia API Keys中的`Admin API Key`，将其值添加到Github的`Secrets`中
{{< /admonition >}}

2. 文档图片

	之前使用Hexo时文章和图片文件夹并行在posts中
	
    ```
    ➜  houugen.github.io git:(hugo) ✗ tree posts
    posts/
    ├── Blockchain-Security-Articles.md
    ├── Blockchain-WordCloud
    │   ├── 1.png
    │   ├── 2.png
    │   └── 3.png
    ├── Blockchain-WordCloud.md
    ```
  
	但是迁移到Hugo后图片都无法显示，看了文档明白是路径问题，图片资源可以放在根路径的static中，这样文章图片是能显示，但是需要修改大量文章中的图片引用路径，查阅[资料](https://xiexingchao.ink/posts/post-image-path-problem-in-hugo.html)后发现可以在`config.toml`中插入`uglyurls = true`解决。
	
	但是又引入了新Bug 😭，站头的`Posts`、`Tags`、`Categories`均无法正常解析了，在主题的issues搜索发现有解决方法：[https://github.com/dillonzq/LoveIt/issues/266](https://github.com/dillonzq/LoveIt/issues/266)

3. 自动化发布
   * 使用了 [peaceiris/actions-gh-pages@v3](https://github.com/peaceiris/actions-gh-pages) 发布GitPage，创建并使用SSH Deploy Key后发布成功
   * 如果你有其他解析域名，需要使用参数`cname: your domain`，否则每次发布后会覆盖删除GitPage仓的CNAME文件。

最后附上我的Github Actions 工作流配置文件：

```yml
name: Hugo Build and Deploy

on:
  push:
    branches: [ hugo ]

jobs:
  build_deploy:
    runs-on: ubuntu-18.04

    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: 'hugo'
          submodules: true
          fetch-depth: 0

      - name: Setup Node
        uses: actions/setup-node@v2.1.0
        with:
          node-version: '12.x'

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          publish_dir: ./public
          publish_branch: master
          cname: houugen.fun

      - name: Setup Algolia
        env:
          ALGOLIA_APP_ID: 63ROI1DYLO
          ALGOLIA_ADMIN_KEY: ${{ secrets.ALGOLIA_ADMIN_KEY }}
          ALGOLIA_INDEX_NAME: houugen_blog
          ALGOLIA_INDEX_FILE: public/index.json
        run: npm install atomic-algolia && npm run algolia
```

## 感受

1. 主题真的好看😂，而且又有很多实用功能，比如流程图、嵌入音视频、数学公式和Shortcodes等
2. 之前Hexo用的travis自动部署，而Github Actions体验更加方便，配合各种action能清晰快捷的编写工作流
3. Hugo的毫秒级部署不吹了，而且编写Markdown还能即时网页查看，舒服
