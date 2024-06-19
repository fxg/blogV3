---
layout: post
title: "Ruby On Rails 6 中使用 Webpacker 管理 Javascipt 模块和相关的 CSS"
date: 2019-11-22 09:50:00 +0800
categories: [技术]
tags: [Ruby On Rails, Webpacker]
---

> 本文成文过久，请谨慎参考。
{: .prompt-info }

Ruby On Rails 6 相对 5 极好的一个变化就是默认使用了 Webpacker 模块做 Javascript 库管理，package.json 就像 Gemfile 一样，存储了相关的模块和依赖关系。使用 yarn 安装模块可是容易多了。

- Table of Contents
{:toc .large-only}

新建一个 Rails 6 工程，开始本文的 Webpacker 之旅。

### 1、新建一个资源，使用脚手架：

``` shell
rails g scaffold Post title publish_date:date --no-scaffold-stylesheet

rails db:migrate
```

### 2、更改根路由：

``` ruby
root to: "posts#index"
```

这时执行 rails s 命令，启动应用后，是一个默认的帖子空列表页面，点击 “New Post” 按钮，表单的日期输入还是默认的日期选择框，我们将更改为更好看的更人性化的组件：flatpickr。

### 3、化腐朽为神奇

执行命令：

``` shell
yarn add flatpickr
```


编辑 app\javascript\packs\application.js 文件，增加如下内容：

``` js
import flatpickr from "flatpickr"
require("flatpickr/dist/flatpickr.css")

document.addEventListener("turbolinks:load", () => {
  flatpickr("[data-behavior='flatpickr']", {
    altInput: true,
    altFormat: "F j, Y",
    dateFormat: "Y-m-d",
  })
})
```

Webpack 不光集成 Javascript，还可以集成相关的样式表，比如：require("flatpickr/dist/flatpickr.css")，增加了对应的 helper 函数：stylesheet_pack_tag 和 javascript_pack_tag，我们在 application.html.erb 文件中会用到。

Post 的 _form.html.erb 文件，更改 publist_date 的输入方式：

``` ruby
<%= form.text_field :publish_date, data: { behavior: "flatpickr" } %>
```

application.html.erb 增加：

``` ruby
<%= stylesheet_pack_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
```

完整的：

``` html
<%= stylesheet_link_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
<%= stylesheet_pack_tag 'application', media: 'all', 'data-turbolinks-track': 'reload' %>
<%= javascript_pack_tag 'application', 'data-turbolinks-track': 'reload' %>
```

此时，刷新你的首页，看看是否达到预期。

更多例子：

### 4、使用 Bootstrap 及相关套件

执行命令：

``` shell
yarn add bootstrap jquery popper.js
```

设置 Webpack，编辑 config/webpack/environment.js，增加如下内容：

``` js
const webpack = require('webpack')
environment.plugins.prepend('Provide',
  new webpack.ProvidePlugin({
    $: 'jquery/src/jquery',
    jQuery: 'jquery/src/jquery',
    Popper: ['popper.js', 'default']
  })
)
```

完整内容如下：

``` js
const { environment } = require('@rails/webpacker')

const webpack = require('webpack')
environment.plugins.prepend('Provide',
  new webpack.ProvidePlugin({
    $: 'jquery/src/jquery',
    jQuery: 'jquery/src/jquery',
    Popper: ['popper.js', 'default']
  })
)

module.exports = environment
```

编辑 app/javascript/packs/application.js，增加：

``` js
require("bootstrap")
require("bootstrap/scss/bootstrap")
```

如果兼容旧版 stylesheet_link_tag 的话，也可以不写 require("bootstrap/scss/bootstrap")，而替换为在文件 app/assets/stylesheets/application.css 中增加：

``` css
@import 'bootstrap/scss/bootstrap'
```

### 番外：Procfile.dev 的 foreman

foreman 是管理多条启动命令的工具，可以同时启动你的数据库，缓存，sidekiq，还有 webpack-dev-server。

Procfile.dev 典型命令行：

```
rails: bin/rails s -p 3000
webpack-dev: bin/webpack-dev-server
worker: sidekiq
```

根目录新建 .foreman：

```
procfile: Procfile.dev
```

运行命令：

``` shell
foreman start
```