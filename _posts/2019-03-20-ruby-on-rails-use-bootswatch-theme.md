---
layout: post
title:  "在 Ruby On Rails 中使用 bootswatch flatly 主题"
date:   2019-03-20 10:29:10 +0800
categories: [技术]
tags: [Ruby On Rails, bootswatch]
---

看了不少管理主题，回到本初，使用简单简洁的 bootswatch 主题。来吧，直接进入正题。

- Table of Contents
{:toc .large-only}

## 0、环境 ##

|软件|版本|
|-|-|
|Ruby|2.5.5|
|Rails|5.2.2.1|

## 1、建一个新的 rails 项目 ##

假设你也不用 turbolinks。

``` bash
rails new bootswatchtest --skip-turbolinks
```

## 2、修改 Gemfile ##

增加如下：

``` ruby
gem 'bootstrap', '~> 4.3.1'
gem 'jquery-rails'
gem "bootswatch", github: "thomaspark/bootswatch"
```

执行：bundle install

## 3、初始化路径中加载 bootswatch 资源目录 ##

在 app -> config -> initializers 下，新建文件，比如叫：bootswatch.rb，文件内容：

``` ruby
Rails.application.config.assets.paths += Gem.loaded_specs["bootswatch"].load_paths
```

## 4、修改 application.css 文件 ##

修改 app -> asset -> stylesheets -> application.css 文件，在 "require_self" 行下增加一行：

``` css
*= require bootswatch_flatly
```

在 app -> asset -> stylesheets 目录下，增加文件：bootswatch_flatly.scss，文件名与增加的哪一行相同，内容为：

``` scss
@import "flatly/variables";

// import original bootstrap
@import "bootstrap";

@import "flatly/bootswatch";

```

其中 flatly ，可以根据喜好，修改为 bootswatch 其他主题名称。

## 5、修改 application.js 文件 ##

修改 app -> asset -> javascripts -> application.js 文件，在 "//= require_tree ." 行下增加：

``` js
//= require jquery3
//= require popper
//= require bootstrap-sprockets
```

这一步是为了使用 bootstrap 的 jquery。

## 6、测试主题使用 ##

新建 controller：

``` bash
rails g controller home index
```

打开 app -> views -> home -> index.html.erb 文件，增加测试导航条：

``` html
<nav class="navbar navbar-expand-lg navbar-dark bg-primary">
  <a class="navbar-brand" href="#">Navbar</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarColor01" aria-controls="navbarColor01" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>

  <div class="collapse navbar-collapse" id="navbarColor01">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item active">
        <a class="nav-link" href="#">Home <span class="sr-only">(current)</span></a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Features</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Pricing</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">About</a>
      </li>
    </ul>
  </div>
</nav>

<ul class="nav nav-pills">
  <li class="nav-item">
    <a class="nav-link active" href="#">Active</a>
  </li>
  <li class="nav-item dropdown">
    <a class="nav-link dropdown-toggle" data-toggle="dropdown" href="#" role="button" aria-haspopup="true" aria-expanded="false">Dropdown</a>
    <div class="dropdown-menu">
      <a class="dropdown-item" href="#">Action</a>
      <a class="dropdown-item" href="#">Another action</a>
      <a class="dropdown-item" href="#">Something else here</a>
      <div class="dropdown-divider"></div>
      <a class="dropdown-item" href="#">Separated link</a>
    </div>
  </li>
  <li class="nav-item">
    <a class="nav-link" href="#">Link</a>
  </li>
  <li class="nav-item">
    <a class="nav-link disabled" href="#">Disabled</a>
  </li>
</ul>
```

运行 rails 应用，并在浏览器中查看是否已经使用了你的主题。

## 6、 题外话 ##

如果不幸你也碰到了这个错误：

Puma caught this error: Error loading the 'sqlite3' Active Record adapter. Missing a gem it depends on? can't activate sqlite3 (~> 1.3.6), already activated sqlite3-1.4.0. Make sure all dependencies are added to Gemfile. (LoadError)

可以在 Gemfile 中，将 sqlite 版本降为：1.3.6：

``` ruby
gem 'sqlite3', '~> 1.3.6'
```


bootswatch 主题好耶，bootswatch 主题棒！
