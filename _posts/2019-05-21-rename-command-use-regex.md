---
layout: post
title:  "使用带正则表达式功能的 rename 命令来给你的美剧字幕文件重命名！"
date:   2019-05-21 13:50:10 +0800
categories: [技术]
tags: [正则表达式, rename 命令]
---

生活大爆炸剧终了，权利的游戏剧终并完美烂尾，到了整理全剧及相关字幕文件的时候了。但是……，但是……，字幕文件名字和剧文件名并不一定完全匹配，这就需要将他们的名字变成一致，才能在 Kodi 中自动加载，一个个的改，肯定不会做的，用十个小时找个工具去批量改名，也不会花一分钟去一个个的改，因为太 Low 了！写 bash sed 也不是不可能了，但是谁记得住？幸好有支持正则表达式的 rename 命令。

当然大概率系统可能没有安装，不同的系统安装命令方式不一样。

比如 Debian：

``` bash
sudo apt install rename
```

安装好了，开始。

我有这些文件：

```
Game.of.Thrones.S02E01.The.North.Remembers.ass         Game.of.Thrones.S02E06.The.Old.Gods.and.the.New.ass
Game.of.Thrones.S02E02.The.Night.Lands.ass             Game.of.Thrones.S02E07.A.Man.Without.Honor.ass
Game.of.Thrones.S02E03.What.Is.Dead.May.Never.Die.ass  Game.of.Thrones.S02E08.The.Prince.of.Winterfell.ass
Game.of.Thrones.S02E04.Garden.of.Bones.ass             Game.of.Thrones.S02E09.Black.Water.ass
Game.of.Thrones.S02E05.The.Ghost.of.Harrenhal.ass      Game.of.Thrones.S02E10.Valar.Morghulis.ass

```

我要把他们变成这样式儿的：

```
game.of.thrones.s02e01.720p.bluray.x264-demand.ass  Game.of.Thrones.S02E06.720p.bluray.x264-demand.ass
Game.of.Thrones.S02E02.720p.bluray.x264-demand.ass  Game.of.Thrones.S02E07.720p.bluray.x264-demand.ass
Game.of.Thrones.S02E03.720p.bluray.x264-demand.ass  Game.of.Thrones.S02E08.720p.bluray.x264-demand.ass
Game.of.Thrones.S02E04.720p.bluray.x264-demand.ass  Game.of.Thrones.S02E09.720p.bluray.x264-demand.ass
Game.of.Thrones.S02E05.720p.bluray.x264-demand.ass  Game.of.Thrones.S02E10.720p.bluray.x264-demand.ass
```

我用如下命令：

``` bash
rename -v "s/Game\.of\.Thrones\.S02E(\d*)\.(.*)\.ass/Game\.of\.Thrones\.S02E\1\.720p\.bluray\.x264-demand\.ass/" *.ass
```

最主要的是把 perl 正则表达式中的 $1 变成 \1 来代表第一组匹配的内容。

都是普通的 perl 正则表达式啦，但是真的很方便啊。