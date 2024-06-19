---
layout: post
title:  "Gentelella 主题在 Ruby On Rails 中的使用"
date:   2018-04-16 18:03:18 +0800
categories: [技术]
tags: [Ruby On Rails, Gentelella]
---

- Table of Contents
{:toc .large-only}

## 文件复制

将 git 库目录中的 build 目录，拷贝到 rails 工程的 vendor目录下，并将 build 改名为 gentelella。

在 rails 工程里，找到 app -> config -> initializers -> assets.rb 文件，增加如下两行：

```ruby
# 主题目录
Rails.application.config.assets.paths << Rails.root.join('vendor', 'assets', 'gentelella')
```

在 app -> assets -> stylesheets -> application.scss 文件中，增加如下行：

```scss
@import "css/custom.min"
```

表明将 gentelella 主题目录加入css文件扫描列表中，其中 “css/” 在 vendor -> assets -> gentelella 目录下。

在 app -> assets -> javascripts -> application.js 文件中，增加如下行：

```js
//= require 'js/custom.min'
```

表明将 gentelella 主题目录加入css文件扫描列表中，其中 “js/” 在 vendor -> assets -> gentelella 目录下。

## rails 模块

在 Gemfile 中增加 gentelella 依赖的 bootstrap 3：

```ruby
gem 'bootstrap-sass', '~> 3.3.7'
gem 'jquery-rails'
```

## 文件修改

将 app/assets/stylesheets/application.css 更名为 app/assets/stylesheets/application.scss，增加：

```scss
@import "bootstrap-sprockets";
@import "bootstrap";

@import "css/custom.min"
```

删掉文件中的 *= require_self 和 *= require_tree 两行

在 app/assets/stylesheets/application.js 中，增加：

```js
//= require jquery
//= require bootstrap-sprockets
//= require 'js/custom.min'
```

## 增加其他模块

将原始 git 库中对应目录下的文件，放到 vendor -> assets -> gentelella 目录下，然后根据具体情况修改 application.js 和 application.scss

例如增加 bootstrap-daterangepicker 模块，由于此模块还依赖 moment 模块，所以在 gentelella 下建立两个目录，moment 和 bootstrap-daterangepicker，并将各自对应的 js 和 css 拷贝到对应目录，然后修改对应文件：

application.js 增加（注意顺序，被依赖项靠前）：

```js
//= require moment/moment-with-locales.min
//= require bootstrap-daterangepicker/daterangepicker
```


application.scss 增加：

```scss
@import "bootstrap-daterangepicker/daterangepicker"
```

修改对应的 view ，即可测试对应组件的执行情况。
