---
title: Ghost 更新到了 1.0 正式版
date: 2017-07-30 05:00:05
categories: Note
---

几星期前就看到有个朋友建了一个 Ghost 1.0.0 beta 的站，Casper 主题也更新到了 2.0 版，变化还是很大的，取消了第一屏的设计，首页从列表换成了瀑布流，文章页的布局和排版也有了很多处的调整，更加有了作为一个博客的风格~~文艺范~~，今天看到正式版发布了，打算升级一下，却不料诸事不顺。。。
<!--more-->

从 0.11.x 升级到 1.0.0 官方并不提供一个可以直接平滑升级的脚本，官方的教程也只是让你进行一次全新安装，这就是各种坑。官方如是说：
>Ghost 1.0.0 是一个大版本更新，将会有很多改变并且没有自动化迁移工具。尽管升级的过程会变得无比复杂，但是我们希望你将会发现这是值得的。

反观隔壁 WordPress，这么多版本了都是一键平滑升级，~~可能是 Ghost 团队没有测试部门~~。


之前一直在用 7.10.0 的 NodeJS 来跑，虽然不在官方的支持列表里，但是跑了2个月也没遇到什么问题，以为顺着这次更新官方能提升一下支持的环境，然而并不行，还是要用 Node 7.x 来跑，当然这是我个人的选择，自己搭建我还是推荐使用官方支持的 LTS 版本。

在安装新版本之前，需要先在后台里把数据导出。然后安装 ghost-cli
```html
$ npm install i -g ghost-cli
```
在新文件夹里安装 ghost
```html
$ ghost install local
```

你可能会遇到一个这种错误
<blockquote><p><font color='red'>Command failed: /bin/sh -c sudo -E -u ghost /usr/local/lib/node_modules/ghost-cli/node_modules/.bin/knex-migrator-migrate --init --mgpath /home/wwwroot/ghost<br>
events.js:163
      throw er; // Unhandled 'error' event
</font></p></blockquote>

暂时你需要手动执行数据库初始化的动作，首先需要新建一份配置文件，虽然文档里说 config 可以放在根目录下，但是实测放在根目录里并不能生效，还是要放在 
`versions/1.0.2/core/server/config/config.production.json` 下，格式虽然从 js 改成了 json，但是结构并没有变，把之前的配置稍作修改就可以了。

然后要安装 knex-migrator
```html
$ npm install -g knex-migrator
$ NODE_ENV=production knex-migrator init
```

因为 Ghost 改了文件结构，所以 pm2 的配置也要修改一下。新建一个 json 文件例如 blog.json，写一下 pm2 的 app 配置。
```json
{
  "name"        : "ghost_blog",  // 应用名称
  "script"      : "/home/wwwroot/ghost/current/index.js",  // 实际启动脚本
  "cwd"         : "./",  // 当前工作路径
  "watch": [  // 监控变化的目录，一旦变化，自动重启
    "versions"
  ],
  "ignore_watch" : [  // 从监控目录中排除
    "node_modules", 
    "log",
    "public"
  ],
  "watch_options": {
    "followSymlinks": true
  },
  "error_file" : "./log/err.log",  // 错误日志路径
  "out_file"   : "./log/out.log",  // 普通日志路径
  "env": {
      "NODE_ENV": "production"  // 环境参数，当前指定为生产环境
  }
}
```
然后使用这个 app 配置来启动 ghost
```html
$ pm2 start blog.json
$ pm2 save
```
然后在 Ghost 后台里导入数据，把图片啥的文件拷贝过来就完成了。

如果不习惯新的主题的话，官方也提供了一个兼容的旧版本主题 [Casper 1.4](https://github.com/TryGhost/Casper/tree/1.4)。
如果又想用旧主题又想有顶端进度条的话，我仓库里有个修改过的 [CasperV1](https://github.com/Magentaize/CasperV1)。

如果要更新 ghost 的话，要先 cd 到 ghost 根目录，然后执行
```html
$ NODE_ENV=production ghost update
```
还是因为那个数据库的问题，更新数据库的时候还是会出错，需要手动执行
```html
$ NODE_ENV=production /usr/local/lib/node_modules/ghost-cli/node_modules/.bin/knex-migrator-migrate --init --mgpath /home/wwwroot/ghost
```
*<center>------------ Ghost 1.0.2更新 ------------</center>*
Ghost 更新了一波之后可以正确识别到配置文件了，于是 config.production.json 放在根目录里就可以了。
*<center>------------ 7 月 29 日更新 ------------</center>*
Github 上很多人有那个数据库初始化的问题，今天有人提交了 pull request 并且也已经合并到了主分支 [#395](https://github.com/TryGhost/Ghost-CLI/pull/395)。

<font color='grey'>引用：
[使用PM2管理node进程](http://lizhiqiang.me/pm2.html)
</font>