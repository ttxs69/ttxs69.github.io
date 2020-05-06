---

title: github搭建Hexo博客
date: 2019-07-31 14:29:43
categories: 
 - 教程
---
# github搭建Hexo博客

## 安装相关环境

首先我们需要安装 Git 和 Node.js，还需要有一个Github的账号，已经安装好的同学可以跳过此部分。

[hexo环境配置](https://hexo.io/zh-cn/docs/)

[Git安装以及初始配置](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137396287703354d8c6c01c904c7d9ff056ae23da865a000)

Git配置个人的用户名称和电子邮件地址：

```sh
git config --global user.name "yourname"
git config --global user.email "youremail@example.com"
```

## 安装Hexo

[hexo官方文档](https://hexo.io/zh-cn/)（强烈建议大家自己看文档学习）

不喜欢看文档的同学可以按照一下步骤操作：

打开cmd，输入以下命令。`#`后面的是注释

```sh
npm install hexo-cli -g #下载hexo
hexo init blog #在当前目录下新建一个名叫`blog`的文件夹，并在其中建立hexo相关文件
cd blog #进入`blog`文件夹
npm install #在此文件夹中建立相关文件
hexo server #启动hexo本地服务器
```

打开浏览器，访问`localhost:4000`,可以看到博客已经搭建成功。

## 将博客与Github关联

### 新建仓库

在Github上创建名字为XXX.github.io的项目，XXX为自己的github用户名。具体步骤如下：

1.新建仓库：
在右上角点击加号，选择`New repository`

2.仓库命名：
创建名字为XXX.github.io的项目，XXX为自己的github用户名。勾选下面的`Initialize this repository with a README`,之后点击`Creat repository`。

3.点击右面的`Settings`

4.向下翻，找到`Github Pages`,点击`Choose a Theme`

5.随便选择一个主题，点击`Select theme`.

6.点击`Commit changes`.

7.复制地址：回到`Code`页面，点击右面的`Clone or download`,复制里面的链接。

### 修改博客配置文件

1.打开本地的blog文件夹项目内的_config.yml配置文件,找到`deploy:`
按照下面的填写，请将`repository:`后面的链接替换为你自己的。注意`":"`之后有一个空格。

```yml
deploy:
  type: git  
  repository: https://github.com/test/test.github.io.git  
  branch: master
```

运行下面的命令：

```sh
`npm install hexo-deployer-git –save #安装git插件hexo g #本地生成静态文件hexo d #将本地静态文件推送至Github`
```

此时，打开浏览器，访问[http://XXXX.github.io](http://xxxx.github.io/) ,其中XXXX为你的Github用户名，就可以看到你的博客了。

## 新建博文

### 新建文章

在blog目录下执行：

```sh
hexo new newpost #新建一个名叫newpost的文章
```

会在`source/_posts`文件夹内生成一个.md文件。

编辑该文件（遵循Markdown规则）

[MakeDown语法](https://www.jianshu.com/p/191d1e21f7ed)

### 修改起始字段

```markdown

title: #文章的标题
date: #创建日期 （文件的创建日期 ）
updated: #修改日期 （文件的修改日期）
comments: #是否开启评论 
truecategories: #标签
categories: #分类
permalink: #url中的名字（文件名）
```

### 编写正文内容（遵循Markdown规则）

### 本地调试

在`blog`目录下运行：

```sh
hexo clean #删除本地静态文件（Public目录），可不执行。
hexo g #生成本地静态文件（Public目录）
hexo s #启动本地服务器
```

### 推送到github

```sh
hexo d #将本地静态文件推送至github
```

## 更换主题

我使用的是`next`主题。下面说明更换next主题的过程，其他主题步骤与此类似。

再次建议大家通过官方文档学习：[NexT主题官方文档](http://theme-next.iissnan.com/getting-started.html)

### 下载主题

在blog目录下运行：

```sh
git clone https://github.com/iissnan/hexo-theme-next themes/next

```

### 启用主题

与所有 Hexo 主题启用的模式一样。 当 克隆/下载 完成后，打开blog目录下的`_config.yml`， 找到`theme`字段，并将其值更改为`next`。效果如下：

```yml
theme: next

```

到此，NexT 主题安装完成。下一步我们将验证主题是否正确启用。在切换主题之后、验证之前， 我们最好使用`hexo clean` 来清除 Hexo 的缓存。

### 验证主题

首先启动 Hexo 本地站点，并开启调试模式（即加上 –debug），整个命令是 `hexo s --debug`。 在服务启动的过程，注意观察命令行输出是否有任何异常信息，如果你碰到问题，这些信息将帮助他人更好的定位错误。
当命令行输出中提示出：

```sh
INFO  Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop.

```

此时即可使用浏览器访问 [http://localhost:4000](http://localhost:4000)，检查站点是否正确运行。

现在，你已经成功安装并启用了 NexT 主题。下一步我们将要更改一些主题的设定，包括个性化以及集成第三方服务。

### 主题设定

请参考官方文档：[NexT官方文档](http://theme-next.iissnan.com/getting-started.html)

参考链接：
[Hexo搭建博客教程](https://thief.one/2017/03/03/Hexo搭建博客教程/)

[Hexo-NexT搭建个人博客（一）](https://neveryu.github.io/2016/09/03/hexo-next-one/)

[nexT主题官方文档](http://theme-next.iissnan.com/getting-started.html)

[Hexo NexT 主题SEO优化指南](https://blog.paddings.cn/2016/08/16/blog/Hexo-NexT-SEO/)

[hexo的next主题个性化教程](https://www.jianshu.com/p/f054333ac9e6)

[nexT官网](https://theme-next.org/)（暂未开放）

 

