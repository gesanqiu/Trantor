# Gitbook Tutorial

## Gitbook

gitbook是一个基于Node.js的命令行工具。

### Install

```shell
# install node.js 10.23.1 and npm
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt-get install -y nodejs

# check the node.js version
nodejs -v

# install gitbook
sudo npm install -g gitbook-cli

# check the gitbook version
gitbook -V
```

### Usage

#### Basic command

- 新建书籍，在目录中执行：`gitbook init`
- 构建预览html文件：`gitbook build`
- 开启服务在浏览其中预览：`gitbook serve`

#### Directory structure

After you init, you should get two `.md` file include `README.md` and `SUMMARY.md`, and in genaral, one gitbook will format like this: 

```shell
.
├── book.json
├── README.md
├── SUMMARY.md
├── chapter-1/
|   ├── README.md
|   └── something.md
└── chapter-2/
    ├── README.md
    └── something.md
```

- `book.json`：主要用来放置配置信息，包括页面设置，插件等。
- `README.md`：通常是gitbook的说明信息
- `SUMMARY.md`：决定 GitBook 的章节目录，它通过 Markdown 中的列表语法来表示文件的父子关系，下面是一个简单的示例。

```shell
# SUMMARY.md
* [Introduction](README.md)
* [Part I](part1/README.md)
    * [Writing is nice](part1/writing.md)
    * [GitBook is nice](part1/gitbook.md)
* [Part II](part2/README.md)
    * [We love feedback](part2/feedback_please.md)
    * [Better tools for authors](part2/better_tools.md)
```

这个配置对应的目录结构如下所示:

![](gitbook_dir.jpg)

### Gitbook-plugin

GitBook 有 [插件官网](https://links.jianshu.com/go?to=https%3A%2F%2Fplugins.gitbook.com%2F)，默认带有 5 个插件，highlight、search、sharing、font-settings、livereload。修改gitbook-plugin只需要修改项目目录下的`book.json`即可，例如：

```json
{
    "plugins": [
        "toc",
        "hide-element",
        "page-treeview",
        "simple-page-toc"
    ],

    "pluginsConfig":{
        "hide-element": {
            "elements": [".gitbook-link"]
        },
        "page-treeview": {
            "copyright": "Copyright &#169; aleen42",
            "minHeaderCount": "2",
            "minHeaderDeep": "6"
        }
    }

}
```

- `toc`为目录插件

- `hide-element`为隐藏组件插件，用于隐藏`gitbook Copyright`  
- `page-treeview`目录插件
- `simple-page-toc`简易导航插件
- `gitbook install`安装新插件

## Markdown

`Gitbook`文档使用`.md`格式文档，关于`Markdown`的语法可以查看[Markdown教程](https://www.runoob.com/markdown/md-tutorial.html)。

`Markdown`编辑器可以使用[Typora](https://www.typora.io/).

## FAQ

> 禁用page-treeview plugin copyright

Copyright信息为plugin作者内嵌信息，需要修改plugin脚本源码，删除关于copyright相关定义与显示。

```shell
# 当前项目目录
vim ./node_modules/gitbook-plugin-page-treeview/lib/index.js

# 查找Copyright相关定义，删去即可
# 注意：每次执行gitbook install之后都要修改
```




>  `gitbook build`生成的`.html`单击不跳转：

点击事件被js代码禁用导致新版本的gitbook不支持单击事件，所以点击没有反应，但是如果右键，在新窗口/新标签页打开的话是可以跳转的，解决办法如下：

```shell
# 当前项目目录
vim ./_book/gitbok/theme.js

# 查找 “if(m)for(n.handler&&“
# 将判断条件m改为false即可
# 注意：每次gitbook build之后都需要修改
```

> `gitbook`部署
```shell
# 将远端仓库克隆到本地
# 然后将本地修改上传，并使用Github Pages来解析
git push -u origin master
git subtree push --prefix=_book origin gh-pages
```
