---
title: 从 Typecho 到 Hugo 的选择与迁移
slug: selection-and-migration-from-typecho-to-hugo
date: 2021-08-20T17:03:40+08:00
tags:
  - Hugo
  - Typecho
  - 博客迁移
categories:
  - 杂七杂八的分享
---

本站最近从 Typecho 迁移到 Hugo，写这篇文章分析一下 Typecho 和 Hugo 各自的优缺点，给读者在这两者之间选择提供参考。最后记录一下我的迁移过程供后来者参考：Typecho 在服务器已经挂掉的情况下如何恢复所有的文章，然后保存为 Hugo 的文件组织方式

<!--more-->

## Hugo 与 Typecho 比较

### Hugo

{{< tip success >}}
优点：
- 静态博客生成器：生成静态 HTML，性能好，部署方便，很多 serverless 服务商支持一键部署 Hugo
- 使用模板语法和 shortcodes 实现灵活的内容组装
- 使用 Hugo pipeline 自动化处理资源文件：SCSS 生成 CSS、minify js、压缩或生成不同尺寸的图片
- 提供一系列现代化功能： 输出 JSON 或 AMP 页面、拓展的 Markdown 语法、接入 Google Analytics、i18n 等
- 文档完善，社区活跃
{{< /tip >}}

{{< tip error >}}
缺点：
- DIY 门槛比较高，学习曲线比较陡峭
- 主题数量比较少，大多都是英文主题，并且主题提供的功能或可以自定义的选项比较少
- 很难实现一些拓展功能，比如博主开发的几款 Typecho 插件就很难在 Hugo 上实现类似的功能
- 没有评论系统。虽然支持集成 disqus，但是由于众所周知的原因 disqus 在中国不能访问
{{< /tip >}}

### Typecho

{{< tip success >}}
优点：
- 主题丰富：有很多漂亮的中文主题，并且提供很多 DIY 选项
- 插件丰富：有第三方插件市场， 提供丰富的拓展功能
- 新手友好：中文社区中教程很多，出问题也有很多大佬可以请教
- 提供管理后台，支持注册用户、发布文章等
{{< /tip >}}

{{< tip error >}}
缺点：
- 早已停止功能更新，只进行维护
- Typecho 是基于 php 的动态博客系统，性能不如静态博客
- 部署比较麻烦，需要部署 web server、php、database，很难使用 serverless 的方式部署
{{< /tip >}}

### Typecho 与 Hugo 选择我之见

博主从 Typecho 迁移到 Hugo 的主要动力是之前白嫖的服务器到期了。。。相比于 Typecho 用的 php 我对 Hugo 的 go 语言更熟悉一些，而且 Hugo 完善的文档更是极大程度上方便于我的 DIY 大业。

Hugo 作为一个静态博客系统最主要的缺点( ~~优点~~ )是没有( ~~不需要~~ )后端，但是这可以通过各家云厂商的函数计算来弥补。函数计算+云上托管的数据库就可以实现一切你想实现的插件功能，本文评论系统的 api 就使用了 Vercel 提供的 function 服务。当然，以上构想需要你具备一定的编程知识，这恐怕不是简单的照猫画虎可以快速掌握的。

博主这里给出一些 Hugo 和 Typecho 的选择建议供读者参考：

- 如果你对静态博客生成器有强需求，那我估计你也不会读到这 qwq
</br>
- 如果你有编程背景，熟悉云服务，崇尚 GitOps，那么选择 Hugo
- 如果你想 DIY 并且原意折腾，反复调试各种错误并乐此不疲，那么选择 Hugo
- 如果你追求极值的访问速度/没有也不想有自己的服务器/有多语言等依赖于 Hugo 特性的需求，那么选择 Hugo
</br>
- 如果你是小白或有一定的 DIY 需求但是并不想折腾，那么选择 Typecho
- 如果你想有一个在线写作/方便的更新文章的平台，那么选择 Typecho
- 如果你想一劳永逸的部署/有一个用户&评论系统/有收费等特殊的插件需求，那么选择 Typecho

## 从 Typecho 到 Hugo

网上有一些 Typecho 迁移插件可以将 Typecho 的文章和附件导出为 Hugo 的文件组织方式，但是我的服务器已经挂掉了没法用这些插件，只能进行手动迁移

### 迁移文章

直接从数据库的 `typecho_contents` 表中提取文章标题和内容等信息，这里给出一个示例 sql 语句：

```sql
SELECT * FROM blog.typecho_contents where type="post";
```

可以写一个脚本来将所有文章输出到以各自文章名命名的文件中

### 迁移图片等附件

我的博客使用了又拍云的对象储存来保存图片等附件，所以只需要把他们下载下来就好。比较麻烦的是修改这些图片在文章中的链接，如果要手动一个个改太耗时间。推荐在 Hugo 的 static 文件夹中新建一个和之前的图片目录结构一样的文件夹，这样就不用修改链接了

### Hugo 目录结构

我的 Hugo 文件夹结构如图所示：

```bash
tree content/post
.
├── solutions-to-certificate-invalidation-after-android-7
│   ├── cal-hash.jpg
│   ├── index.md
│   ├── install-ca-certificate.jpg
│   └── ssl-handshake-failed.jpg
└── web-font-optimization
    ├── blog_title.jpg
    ├── font_type.jpg
    ├── index.md
    ├── sign.jpg
    └── woff2_support.jpg
```

每个文章新建一个以文章的 slug 命名的文件夹，将文章用到的图片包括文章头图都放在这个文件夹下。在主题中可以用 `.Context.Resources.GetMatch` 得到图片的 resource 对象，这样既便于管理，也能将图像交给 Hugo 进行处理。比如 ssl-handshake-failed.jpg 这张图片就会生成以下几个版本：

```bash
ls resources/_gen/images/post/solutions-to-certificate-invalidation-after-android-7
...
ssl-handshake-failed.2a74b1a908dba312f29b1f3f15095116_hu9d2358b8f64836a3db846b7dbcdad0ac_92801_250x150_fill_q75_box_smart1.jpg
ssl-handshake-failed_hu9d2358b8f64836a3db846b7dbcdad0ac_92801_1024x0_resize_q75_box.jpg
ssl-handshake-failed_hu9d2358b8f64836a3db846b7dbcdad0ac_92801_480x0_resize_q75_box.jpg
```
