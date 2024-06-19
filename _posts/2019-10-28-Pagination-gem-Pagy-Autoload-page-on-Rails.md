---
layout: post
title: "分页组件 Pagy 的页面自动加载"
date: 2019-10-28 12:27:00 +0800
categories: [技术]
tags: [Ruby On Rails, Pagy 分页]
---

Pagy 号称比 WillPaginate 和 Kaminari 还要快，并节省内存。于是，我在新的 Rails 项目中，将分页模块更换为了 Pagy。

- Table of Contents
{:toc .large-only}

本文主要介绍使用 Pagy 分页时的页面自动加载，比如：下拉到页面底部时，自动加载新闻条目。

## 模块准备

### 1、application_controller.rb

``` ruby
class ApplicationController < ActionController::Base
  include Pagy::Backend
end
```

### 2、application_helper.rb

``` ruby
module ApplicationHelper
  include Pagy::Frontend
end
```

### 3、pagy.rb

``` ruby
...

require 'pagy/extras/countless'
require 'pagy/extras/support'

Rails.application.config.assets.paths << Pagy.root.join('javascripts')
Pagy::VARS[:items] = 25

....
```


## 控制器

以 topics_controller 的 index action 为例，这个 action 主要返回新闻条目。

### topics_controller.rb

修改数据记录返回方式，使用 pagy_countless：

``` ruby
  def index
    ...

    @pagy, @topics = pagy_countless(Topic.where(lv1_tag_name: params[:tag]).order(created_at: :desc), link_extra: 'data-remote="true"')
    
    ...
  end
```

## 视图

### 修改 index.html.erb，数据主显示 div：

由：

``` html
<div class="container">
  <div class="list-group col-md-12 col-xs-12">
    <% @topics.each do |topic| %>
    <a href="<%= topic.url %>" target="_blank" class="list-group-item list-group-item-action flex-column align-items-start">
      <div class="d-flex w-100 justify-content-between">
        <h5 class="mb-1"><%= topic.title %></h5>
        <small><%= time_ago_in_words(topic.created_at)%>前，<%= topic.site.title %></small>
      </div>
    </a>
    <% end %>
  </div>
  <div>
    <%== pagy_bootstrap_combo_nav_js(@pagy) %>
  </div>
</div>
```

变更为：

``` html
<div class="container">
  <div class="list-group col-md-12 col-xs-12" id="records_table">
    <%= render partial: 'page_items' %>
  </div>
  <br>
  <div id="div_next_link">
    <%= render partial: 'next_link' %>
  </div>
</div>
```

增加脚本：

``` html
<script>
  var loadNextPage = function(){
    if ($('#next_link').data("loading")){ return }  // prevent multiple loading
    var wBottom  = $(window).scrollTop() + $(window).height();
    var elBottom = $('#records_table').offset().top + $('#records_table').height();
    if (wBottom > elBottom){
      $('#next_link')[0].click();
      $('#next_link').data("loading", true);
    }
  };

  window.addEventListener('resize', loadNextPage);
  window.addEventListener('scroll', loadNextPage);
  window.addEventListener('load',   loadNextPage);
</script>
```

### 增加关联文件

page_items.html.erb（数据条目）：

``` html
<% @topics.each do |topic| %>
<a href="<%= topic.url %>" target="_blank" class="list-group-item list-group-item-action flex-column align-items-start">
  <div class="d-flex w-100 justify-content-between">
    <h5 style="float:left; width:90%"><%= topic.title %></h5>
    <small><%= time_ago_in_words(topic.created_at)%>前，<%= topic.site.title %></small>
  </div>
</a>
<% end %>
```

next_link.html.erb（下一页）：

``` html
<%== pagy_next_link(@pagy, '', 'id="next_link" style="display: none; text-align: center"') %>
```

index.js.html（js 返回数据）：

``` js
$('#records_table').append("<%= j(render 'page_items')%>");
$('#div_next_link').html("<%= j(render 'next_link') %>");
```

至此，重启 Rails 项目，看一下页面是不是已经按照想象中进招了呢？