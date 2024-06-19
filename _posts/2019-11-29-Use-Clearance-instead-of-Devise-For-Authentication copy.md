---
layout: post
title: "使用 Clearance 代替 Devise 做 Rails 项目的权限认证"
date: 2019-11-29 14:37:00 +0800
categories: [技术]
tags: [Ruby On Rails, Clearance, Devise]
---

Clearance 相对于 Devise 来说很轻很简单，还比 has_secure_password 完善，定制起来也很清晰通透，所以，新项目的权限认证的基础模块，我就开始用 Clearance 了。

- Table of Contents
{:toc .large-only}

我们的目的：登陆、登出、重置密码，使用 Clearance，用户管理；我们重载 UserController 来定制，恰好，Clearance 的三个 generators 都能覆盖我们的需求：

``` ruby
Clearance:
  clearance:install
  clearance:routes
  clearance:views
```

开始吧！

## 模块引入

首先肯定是得在 Gemfile 中增加：

``` ruby
gem 'clearance'
```

执行命令：

``` shell
bundle install
```

## 安装设置

``` shell
rails g clearance:install
```

输出如下：

``` log
Running via Spring preloader in process 1027

      create  config/initializers/clearance.rb
      insert  app/controllers/application_controller.rb
      create  app/models/user.rb
      create  db/migrate/20191129061400_create_users.rb

*******************************************************************************

Next steps:

1. Configure the mailer to create full URLs in emails:

    # config/environments/{development,test}.rb
    config.action_mailer.default_url_options = { host: 'localhost:3000' }

    In production it should be your app's domain name.

2. Display user session and flashes. For example, in your application layout:

    <% if signed_in? %>
      Signed in as: <%= current_user.email %>
      <%= button_to 'Sign out', sign_out_path, method: :delete %>
    <% else %>
      <%= link_to 'Sign in', sign_in_path %>
    <% end %>

    <div id="flash">
      <% flash.each do |key, value| %>
        <div class="flash <%= key %>"><%= value %></div>
      <% end %>
    </div>

3. Migrate:

    rails db:migrate

*******************************************************************************
```

按图索骥，依例修改。

在有需要验证身份的 controller 里增加：

``` ruby
before_action :require_login
```
以上为常规使用。


## 定制


比如增加用户名称字段。


我们首先要生成视图：

``` shell
rails g clearance:views
```

修改视图 app\views\sessions\\_form.html.erb，增加：

``` html
<div class="text-field">
  <%= form.label :username %>
  <%= form.text_field :username %>
</div>
```

修改模型：

``` ruby
rails g migration AddUsernameToUsers username:string:index
```

修改对应的 migration，增加用户名唯一限制：unique: true

修改后大体如下：

``` ruby
class AddUsernameToUsers < ActiveRecord::Migration[6.0]
  def change
    add_column :users, :username, :string
    add_index :users, :username, unique: true
  end
end
```

执行命令：

``` shell
rails db:migrate
```

在 User Model 中，增加用户名唯一限制，大体如下：

``` ruby
class User < ApplicationRecord
  include Clearance::User

  validates :username, presence: true, uniqueness: true
end
```

这时打开 /sign_in，已经可以看到用户名登陆的输入框了，但是没法注册新用户（/sign_up），因为对应的 controller 我们还没有做修改。

在修改 controller 之前，我们先获取 Clearance 的路由，以作定制：

``` shell
rails g clearance:routes
```

然后可以发现 route.rb 中增加了如下内容：

``` ruby
  resources :passwords, controller: "clearance/passwords", only: [:create, :new]
  resource :session, controller: "clearance/sessions", only: [:create]

  resources :users, controller: "clearance/users", only: [:create] do
    resource :password,
      controller: "clearance/passwords",
      only: [:create, :edit, :update]
  end

  get "/sign_in" => "clearance/sessions#new", as: "sign_in"
  delete "/sign_out" => "clearance/sessions#destroy", as: "sign_out"
  get "/sign_up" => "clearance/users#new", as: "sign_up"
```

将：

``` ruby
  resources :users, controller: "clearance/users", only: [:create] do
    resource :password,
      controller: "clearance/passwords",
      only: [:create, :edit, :update]
  end
```

更改为：

``` ruby
  resources :users, only: [:create] do
    resource :password,
      controller: "clearance/passwords",
      only: [:create, :edit, :update]
  end
```

也就是去掉：controller: "clearance/users" 部分。

然后，直接在 controllers 目录中 创建文件：users_controller.rb，框架如下：

``` ruby
class UsersController < Clearance::UsersController

  private

  def user_params
    params.require(:user).permit(:username, :email, :password)
  end
end
```

编辑 app\views\users\\_form.html.erb 文件，增加：

``` html
<div class="text-field">
  <%= form.label :username %>
  <%= form.text_field :username %>
</div>
```

此时，访问 /sign_up，已经可以添加 username 了。

至于用户管理的其他 CURD 功能，都可以以常规方式在 users_controller.rb 里实现，比如增加用户列表什么的：

``` ruby
def index
  @logged_in_users = User.where(blah) # whatever logic you need to retrieve the list of users 
end
```

## 整合 pundit 做权限管理

在 Gemfile 中增加：

``` ruby
gem 'pundit'
```

执行命令

``` shell
bundle install
```

在 application controller 中增加：

``` ruby
class ApplicationController < ActionController::Base
  include Pundit
  protect_from_forgery

  after_action :verify_policy_scoped, only: :index

  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  def user_not_authorized
    flash[:alert] = "您没有权限执行此操作。"
    redirect_to(request.referrer || root_path)
  end
end
```
 默认会对所有 controller 鉴权，可以只指定某些 controller ，在 application controller 中增加：

``` ruby
class ApplicationController < ActionController::Base
  ......
  
  after_action :verify_authorized, only: [:posts]
  after_action :verify_policy_scoped, only: :index

  ......
end
```


执行命令：

``` shell
rails g pundit:install
```

给 User model 增加 role 字段：

``` shell
rails g migration AddRoleToUsers role:integer
rails db:migrate
```

在 User model 增加：

``` ruby
  # 用户权限
  enum role: [:admin, :editor, :user]
  after_initialize :set_default_role, :if => :new_record?

  def set_default_role
    self.role ||= :user
  end

  # 至少确保有超级管理员
  def ensure_admin_exists
    unless User.admin.exists?
      errors.add(:base, '只剩一个管理员了')
      throw :abort
    end
  end

  # 超级管理员不能被删掉
  def ensure_not_admin
    if (self.admin? and (User.admin.count <= 1))
      errors.add(:base, '最后一个管理员不能被删除。')
      throw :abort
    end
  end

  def manager?
    self.admin? or self.editor?
  end
```

修改 users_controller.rb，增加 role 参数：

``` ruby
class UsersController < Clearance::UsersController

  private

  def user_params
    params.require(:user).permit(:username, :email, :password, :role)
  end
end
```

给 Post 增加权限认证

``` ruby
rails g pundit:policy post
```

修改 app\policies\post_policy.rb 文件，比如权限设为：只有管理员才能创建、修改、删除，普通用户智能浏览。增加：

``` ruby
  def index?
    true
  end

  def show?
    true
  end

  def new?
    user.present? and user.admin?
  end

  def edit?
    user.present? and user.admin?
  end

  def create?
    user.present? and user.admin?
  end

  def update?
    user.present? and user.admin?
  end

  def destroy?
    user.present? and user.admin?
  end
```

修改 app\controllers\posts_controller.rb 文件，给每个 action 加上 authorize。

重启一下服务，刷新一下页面，可以了。
