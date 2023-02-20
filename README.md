## 简介
* 包含开源项目、技术框架、架构设计理念等内容的个人博客
* 基于[Hexo](https://hexo.io/zh-cn/docs/)博客框架，采用***Markdown***解析文章生成静态网页

## 环境要求
* ***Node.js***（版本需不低于 10.13 ，建议试用 12.0 及以上版本）
* ***Git***

## 目录结构
```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```
### 1）_config.yml
网站的 [配置](https://hexo.io/zh-cn/docs/configuration) 信息，可以在此配置大部分的参数。
### 2）package.json
应用程序的信息。EJS, Stylus 和 Markdown renderer 已默认安装，可以自由移除。
```
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "3.9.0"
  },
  "dependencies": {
    "hexo": "^3.9.0",
    "hexo-deployer-git": "^1.0.0",
    "hexo-generator-archive": "^0.1.5",
    "hexo-generator-category": "^0.1.3",
    "hexo-generator-index": "^0.2.1",
    "hexo-generator-tag": "^0.2.0",
    "hexo-renderer-ejs": "^0.3.1",
    "hexo-renderer-marked": "^0.3.2",
    "hexo-renderer-stylus": "^0.3.3",
    "hexo-server": "^0.3.3"
  }
}
```
### 3）scaffolds
[模版](https://hexo.io/zh-cn/docs/writing#%E6%A8%A1%E7%89%88%EF%BC%88Scaffold%EF%BC%89) 文件夹。当新建文章时，Hexo 会根据 scaffold 来建立文件。

Hexo的模板是指在新建的文章文件中默认填充的内容。例如，如果修改scaffold/post.md中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。
### 4）source
资源文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。
Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。
### 5）themes
[主题](https://hexo.io/zh-cn/docs/themes) 文件夹。Hexo 会根据主题来生成静态页面。

## 常用命令
### 1）新建一篇文章
```
hexo new [layout] [-p <PATH>] [-r] <title>
```
* -p 自定义新文章的路径
* -r 如果存在同名文章，将其替换
* 新建一篇文章。如果没有设置 layout 的话，默认使用 [_config.yml](./_config.yml) 中的 default_layout 参数代替。
如果标题包含空格的话，请使用引号括起来。示例：
```
hexo new page --path about/me "About me"
```
### 2）生成静态文件
```
hexo generate  [-d] [-w]
```
* -d 生成后立即部署网站
* -w 监视文件变动
* 该命令可以简写为
```
hexo g [-d] [-w]
```
### 3）发表草稿
```
hexo publish [layout] <filename>
```
### 4）启动服务期
```
hexo server [-p <PORT>]
```
* -p 重设端口
### 5）部署网站
```
hexo deploy [-g]
```
* -g 部署之前预先生成静态文件
* 该命令可以简写为
```
hexo d [-g]
```
### 6）清除缓存文件
```
hexo clean
```
清除缓存文件 (db.json) 和已生成的静态文件 (public)。

在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。
### 7）[更多命令](https://hexo.io/zh-cn/docs/commands)
