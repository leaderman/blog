# 前言

GitBook是一个基于Node.js的命令行工具，可使用Git和Markdown来编写文档，赞誉太多，不再赘述。

# Node.js

1. 下载安装包

```text
cd /tmp

wget https://nodejs.org/dist/v12.16.1/node-v12.16.1-linux-x64.tar.xz
```

2. 解压安装包

```
tar xvf node-v12.16.1-linux-x64.tar.xz
```

3. 安装

安装过程分为3步：移动安装包解压目录至/user/local、为node、npm建立软链接，以及删除安装包。

```text
mv node-v12.16.1-linux-x64 /usr/local/

ln -s /usr/local/node-v12.16.1-linux-x64/bin/node /usr/bin/node
ln -s /usr/local/node-v12.16.1-linux-x64/bin/npm /usr/bin/npm

rm -rf rm -rf node-v12.16.1-linux-x64.tar.xz
```

# GitBook

**参考链接**：https://github.com/GitbookIO/gitbook/blob/master/docs/setup.md

## 安装

```text
npm install gitbook-cli -g

ln -s /usr/local/node-v12.16.1-linux-x64/bin/gitbook /usr/bin/gitbook

gitbook -V
```

**gitbook-cli** 是用于安装、使用多个不同版本GitBook的工具。使用GitBook时会自动安装需要的版本，比如：“gitbook -V”。

## 初始化

1. GitLab创建项目，命名为“wiki”，内容为空，克隆至本地；

```text
git clone ssh://git@git.intra.weibo.com:2222/dip/wiki.git
```

GitLab创建项目的目的仅仅为Markdown文件的版本控制，不是必须选项，本地直接建立目录也是可以的。

2. 初始化示例

```text
gitbook init wiki
```

3. 预览

执行以下命令：

```text
cd wiki

gitbook serve
```

等待，看到如下信息：

```text
Starting server ...
Serving book on http://localhost:4000
```

即可以通过浏览器访问预览效果，如下：

![avatar](https://yurun-blog.oss-cn-beijing.aliyuncs.com/GitBook%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2%E5%AE%9E%E6%93%8D%E6%89%8B%E5%86%8C/preview.png)

4. 后台启动

```text
mkdir -p /var/log/gitbook

gitbook serve >> /var/log/gitbook/serve.log 2>&1 &

gitbook serve --no-live >> /var/log/gitbook/serve.log 2>&1 &

gitbook serve --no-live --log=debug >> /var/log/gitbook/serve.log 2>&1 &
```

## 目录结构

**链接参考**：https://github.com/GitbookIO/gitbook/blob/master/docs/structure.md

&nbsp;
基本的目录结构，如下图：

![avatar](https://yurun-blog.oss-cn-beijing.aliyuncs.com/GitBook%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2%E5%AE%9E%E6%93%8D%E6%89%8B%E5%86%8C/structure.png)

**book.json**

用于存储配置信息（可选），简单可以理解为配置文件，后续会涉及。

**README.md**

用于描述前言/说明信息（必须），简单可以理解为主页，按照Markdown格式编写即可。

**SUMMARY.md**

用于描述章节列表（可选，建议必须），简单可以理解为导航栏，接下来会介绍。

## SUMMARY.md

**链接参数**：https://github.com/GitbookIO/gitbook/blob/master/docs/pages.md

&nbsp;
SUMMARY.md格式实际是一个链接列表。链接的名称就是章节的名称，链接的目标就是章节文件路径，如下图：

![avatar](https://yurun-blog.oss-cn-beijing.aliyuncs.com/GitBook%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2%E5%AE%9E%E6%93%8D%E6%89%8B%E5%86%8C/summary.png)

“Part I”表示“章节1”，“part1/README.md”表示“章节1对应的文件路径”；“Writing is nice”是“Part I”的子章节，“part1/writing.md”是相对应的文件路径；“GitBook is nice”与“Writing is nice”相同。可以按照上述描述的层级格式继续向下延展。我们可以使用目录 + 子目录的方式对章节文件进行归档。

## 插件

GitBook使用的插件及相应的配置需要通过 **book.json**指定，如下：

```text
{
    "plugins": [
        "expandable-chapters-small",
        "-lunr",
        "-search",
        "search-plus",
        "-sharing",
        "splitter",
        "anchor-navigation-ex-toc",
        "hide-element",
        "insert-logo",
        "code"
    ],
    "pluginsConfig": {
        "hide-element": {
            "elements": [".gitbook-link"]
        },
        "insert-logo": {
            "url": "/images/dip.png",
            "style": "background: none; max-height: 120px; min-height: 120px"
        }
    }
}
```

配置文件的变更可能会导致GitBook进程重启或异常终止，如上述插件配置调整，如果相应的插件没有安装完成，就会导致进程终止，需要安装完成之后，再重新启动。

插件安装命令：

```text
gitbook install
```

1. expandable-chapters-small

    章节导航支持多层目录，并配置箭头图标，点击箭头才能实现收放目录。
    
2. search-plus

    高级搜索，支持中文，使用此插件，需要将默认的 **lunr** 和 **search**禁用掉，即“-lunr”和“-search”。
    
3. sharing

    分享插件，默认开启，禁用。
    
4. splitter

    扩展导航侧边栏，支持宽度可调节。
    
5. anchor-navigation-ex-toc

    为文章增加锚点目录栏及回到顶部功能。
    
6. hide-element

    隐藏元素，如：“Published with GitBook”。
    
7. insert-logo

    左侧导航栏上方插入Logo。

# 预览

![avatar](https://yurun-blog.oss-cn-beijing.aliyuncs.com/GitBook%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2%E5%AE%9E%E6%93%8D%E6%89%8B%E5%86%8C/result.png)

# 团队协作

目前对GitBook了解有限，大致谈下自己的想法：团队成员可以通过GitLab将“wiki”克隆至本地，创建自己各自的写作分支；编写完成且本地启动服务测试正常之后，可以提交并合并至Master。部署GitBook服务的服务器，部署Cron任务，定时Pull Master，保持同步更新。
