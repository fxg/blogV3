---
layout: post
title: "Beego 最简实践"
date: 2023-05-18 13:36:18 +0800
categories: [技术]
tags: [Beego]
---


- Table of Contents
{:toc .large-only}

## Go 环境

从这里下载：https://go.dev/dl/

在 Debian 上：

``` bash
wget https://go.dev/dl/go1.20.4.linux-amd64.tar.gz --inet4-only

tar -zxvf go1.20.4.linux-amd64.tar.gz

sudo mv go /usr/local/
```
建立GOPATH

``` bash 
mkdir ~/go
```

设置环境变量，我的 shell 是 zsh，

``` bash
vi .zshrc
```

添加环境设置，增加如下内容：

```
export PATH=$PATH:/usr/local/go/bin:/home/digg/go/bin
```

## Beego 脚手架工具

下载安装：

``` bash
go install github.com/beego/bee/v2@latest
```

其中脚手架工具 bee 就在 /home/digg/go/bin 下面，前边的步骤已经将其加入环境变量，现在可以执行一下：

``` bash
bee version
```

查看是否正常运行。


## 生成 API 类型项目框架

使用 bee 工具，在 GOPATH 目录的 src 目录下，生成名叫 notepad 的项目框架：

``` bash
bee api notepad
```

然后进入 notepad 目录，执行：

```
go get notepad
```

以安装涉及到的模块。安装完毕后，可以在此目录下运行此命令：

```
bee run
```

如果运行无误，可以打开浏览器查看运行效果。

提示：也可以直接运行 main.go 文件， 例如： go run main.go，这种方式每次修改文件，需要手动编译。


## 最简实践

如果不是需要全功能 CRUD 代码，建议手工在对应目录建立 controller, routers, test 文件，这样可以保证代码的简洁。

### 新建控制器

手工在 controller 目录下，建立文件 hello.go （此文件也算是最简控制器模板，甚至没有数据库模块）:

``` go
package controllers

import (
	beego "github.com/beego/beego/v2/server/web"
)

// Operations about Hello
type HelloController struct {
	beego.Controller
}

func (c *HelloController) URLMapping() {
	c.Mapping("Hello", c.Hello)
}

// @Title Hello
// @Description say hello
// @Success 200 {String} "hello"
// @router / [get]
func (c *HelloController) Hello() {
	c.Ctx.WriteString("hello")
}

```

> 注意这一行，
> ```
> // @router / [get]
> ```
> 虽然是注解行，但是从 beego 1.3 版本开始支持了注解路由，用户无需在 router 中注册路由，只需要 Include 相应地 controller，然后在 controller 的 method 方法上面写上 router 注释（// @router）就可以了，而且在 Beego 2.0.4 版本，如果没有这行注释，即便在 routes.go 和 commentsRouter_controllers.go 还有 commentsRouter.go 中增加了对应的路由描述，也是 404 ，找不到对应路由，此事待了解。

### 增加路由

在项目目录执行：

``` bash
bee generate routers
```

执行后，会将路由描述写入到 commentsRouter_controllers.go 还有 commentsRouter.go 文件中。

在 routers/router.go 中增加：

```go
beego.NSNamespace("/hello",
	beego.NSInclude(
		&controllers.HelloController{},
	),
),
```

### 最简获参和输出 JSON 控制器

``` go
package controllers

import (
	beego "github.com/beego/beego/v2/server/web"
)

// Operations about Bye
type ByeController struct {
	beego.Controller
}

func (c *ByeController) URLMapping() {
	c.Mapping("Bye", c.Bye)
}

// @router / [get]
func (c *ByeController) Bye() {
	username := c.GetString("name")
	c.Data["json"] = map[string]string{"message": "Bye " + username}
	c.ServeJSON()
}
```

### 测试

手工在 test 目录下，建立文件 hello_test.go （此文件也算是最简测试模板），判断接口状态和返回值是否符合预期:

``` go
package test

import (
	"net/http"
	"net/http/httptest"
	_ "notepad/routers"
	"path/filepath"
	"runtime"
	"testing"

	"github.com/beego/beego/v2/core/logs"
	beego "github.com/beego/beego/v2/server/web"
	. "github.com/smartystreets/goconvey/convey"
)

func init() {
	_, file, _, _ := runtime.Caller(0)
	apppath, _ := filepath.Abs(filepath.Dir(filepath.Join(file, ".."+string(filepath.Separator))))
	beego.TestBeegoInit(apppath)
}

func TestHello(t *testing.T) {
	r, _ := http.NewRequest("GET", "/v1/hello", nil)
	w := httptest.NewRecorder()
	beego.BeeApp.Handlers.ServeHTTP(w, r)

	logs.Info("testing", "TestGet", "Code[%d]\n%s", w.Code, w.Body.String())

	Convey("Subject: Test Station Endpoint\n", t, func() {
		Convey("Status Code Should Be 200", func() {
			So(w.Code, ShouldEqual, 200)
		})
		Convey("The Result Should Not Be Empty", func() {
			So(w.Body.String(), ShouldEqual, "hello")
		})
	})
}
```

### 完成记事本功能开发

不使用 generate scaffold 实现一个记事本表的增删改查功能，对应表：note。

本节中所有动作对应的实例名都是：note，不使用复数形式。

#### 生成数据库迁移文件

``` bash
bee generate migration CreateTableNote
```

#### 修改产生的文件

比如我的文件名：20230512_163151_CreateTableNote，修改 up 和 down 两个函数如下：

``` go
// Run the migrations
func (m *CreateTableNote_20230512_163151) Up() {
	// use m.SQL("CREATE TABLE ...") to make schema update
	m.SQL(`
	CREATE TABLE note
	(
		ID int NOT NULL AUTO_INCREMENT,
		NOTETEXT varchar(255) NOT NULL,
		PRIMARY KEY (ID)
	);
	`)
}

// Reverse the migrations
func (m *CreateTableNote_20230512_163151) Down() {
	// use m.SQL("DROP TABLE ...") to reverse schema update
	m.SQL("DROP TABLE if exists note")
}
```

#### 进行数据库迁移

回到项目目录，执行：

``` bash
bee migrate --conn="notepad:notepad123@tcp(127.0.0.1:3306)/notepad?charset=utf8"
```

如果发现数据库设置有误，可以回退这次迁移：

``` bash
bee migrate rollback --conn="notepad:notepad123@tcp(127.0.0.1:3306)/notepad?charset=utf8"
```

修改完成后，可以继续迁移。

> 如果迁移失败，有可能是数据库中表：migrations 中的迁移记录没有删除，只能手工删除后在继续迁移。

#### 配置数据库链接

修改配置文件中的数据库链接信息，conf/app.conf：

``` ini
sqlconn = "notepad:notepad123@tcp(127.0.0.1:3306)/notepad?charset=utf8"
```

#### 引入包

在 main.go 中，引入如下两个包：

``` go
// 导入orm包
"github.com/beego/beego/v2/client/orm"

// 导入mysql驱动
_ "github.com/go-sql-driver/mysql"
```

执行 go mod 安装关联模块：

``` bash
go mod tidy
```

> 此处有坑：main.go 增加 orm 的代码后，go 自动填充到源码中的包如果不是：github.com/beego/beego/v2/client/orm，要修改成它。如果没碰到，忽视此行。

#### 生成 model

``` bash
bee generate model note -fields="id:int64,notetext:string"
```

#### 生成 controller

``` bash
bee generate controller note
```

#### 设置 routers

在 routers/router.go 文件中，追加：

``` go
ns := beego.NewNamespace("/v1",
...
	beego.NSNamespace("/note",
	  beego.NSInclude(
		  &controllers.NoteController{},
	  ),
  ),
...

```

#### 查看结果

查看所有记事：

``` bash
curl -s "http://127.0.0.1:8080/v1/note"
```

### 记事项分类管理功能

#### 一步完成 

使用 bee generate scaffold 生成框架，因为是 API 框架，不生成 view，不迁移数据库。

``` bash
bee generate scaffold catelog -fields="id:int64,name:string"
```

再手工迁移数据库，如果有问题，此时还可以修改，比如自动生成的建表代码中，id 不是自增的，我们可以修改文件 database/migrations/20230512_181612_catelog.go 的 up 函数，从：

``` sql
func (m *Catelog_20230512_181612) Up() {
	// use m.SQL("CREATE TABLE ...") to make schema update
	m.SQL("CREATE TABLE catelog(`id` int(11) DEFAULT NULL,`name` varchar(128) NOT NULL)")
}
```
到

``` sql
// Run the migrations
func (m *Catelog_20230512_181612) Up() {
	// use m.SQL("CREATE TABLE ...") to make schema update
	m.SQL("CREATE TABLE catelog(`id` int(11) NOT NULL AUTO_INCREMENT,`name` varchar(128) NOT NULL, PRIMARY KEY (ID))")
}
```

迁移数据

``` bash
bee migrate --conn="notepad:notepad123@tcp(127.0.0.1:3306)/notepad?charset=utf8"
```

#### 设置 routers

在 routers/router.go 文件中，追加：

``` go
ns := beego.NewNamespace("/v1",
...
	beego.NSNamespace("/catelog",
	  beego.NSInclude(
		  &controllers.NoteController{},
	  ),
  ),
...

```

#### 查看结果

查看所有分类：

``` bash
curl -s "http://127.0.0.1:8080/v1/catelog"
```

## 数据库 SQL 查询最简实践

前文已经引入数据库相关的包，并链接数据成功，本节在其基础上，实现数据库的 SQL 语句查询。

### 返回单行数据到结构体

#### 新建 model

在 models 目录下，新增 hello.go 文件：

``` go
package models

import (
	"github.com/beego/beego/logs"
	"github.com/beego/beego/v2/client/orm"
)

// 定义 Hello 结构体，用以接受数据库查询结果，我们只查询返回字段是 one 的，结果是 1 的记录，所以结构体定义如下：
type Hello struct {
	One int
}

// init 中注册结构体
func init() {
	orm.RegisterModel(new(Hello))
}

// 获取一条字段是 one 的，结果是 1 的记录，并将其返回。
func GetHelloOne() (v *Hello, err error) {
	// 创建数据库链接实例
	o := orm.NewOrm()

	// 定义 hello 的变量 h
	var h Hello

	// 查询返回字段名为 one 的，结果是 1 的记录到变量 h
	// 此处有需要注意的，SQL 语句中的字段名要和对应结构体
	// 的命名一致，但都小写。
	error := o.Raw("select 1 as one").QueryRow(&h)

	// 打印查询结果
	logs.Info(h)

	// 如果查询有错，返回空结构体和错误信息，并打印到 logs
	if error != nil {
		logs.Error(error)
		return nil, error
	}

	// 返回查询结果
	return &h, nil
}

```

#### 修改控制器

修改控制器中的 hello.go 文件，将 hello 函数修改如下：

``` go
// @router / [get]
func (c *HelloController) Hello() {
	// 注释或删除原内容
	// c.Ctx.WriteString("hello")

	// 调用数据模块的函数，获取数据库返回
	h, err := models.GetHelloOne()
	
	// 如果有错误则直接返回错误原因
	if err != nil {
		c.Data["json"] = err.Error()
	}
	
	// 如果查询没问题，将返回值输出成 json
	c.Data["json"] = h
	c.ServeJSON()
}
```

#### 测试输出

``` bash
curl -s "http://localhost:8080/v1/hello"
```

返回值：

``` json
{
  "One": 1
}
```

### 返回多行数据到结构体

#### 修改 model

在 models 下的 hello.go 文件中增加结构体：

``` go
type VMInfo struct {
	Id         int64
	Name       string `orm:"size(128)"`
	Cluster    string `orm:"size(128)"`
	CreateTime *time.Time
}
```

`init()` 函数中，增加注册 model 的过程：

``` go
orm.RegisterModel(new(VMInfo))
```

增加数据获取方法：

``` go
// 获取多条 vm 信息，并将其返回。
func GetVMInfos() (VMs *[]VMInfo, err error) {
	// 创建数据库链接实例
	o := orm.NewOrm()

	// 测试起见，仅取 10 条数据
	sql := `
		select 
			id, 
			JSON_UNQUOTE(insdetail->'$.metadata.name') as name, 
			cluster, 
			STR_TO_DATE(JSON_UNQUOTE(insdetail->'$.metadata.creationTimestamp'), '%Y-%m-%dT%H:%i:%s') as create_time
		from vminstance 
		where insdetail <> '' 
			and insdetail->'$.status.ready' = true
		order by insdetail->'$.metadata.creationTimestamp' desc
		limit 10;
	`
	var lVMs []VMInfo
	_, error := o.Raw(sql).QueryRows(&lVMs)

	// 打印查询结果
	logs.Info(lVMs)

	// 如果查询有错，返回空结构体和错误信息，并打印到 logs
	if error != nil {
		logs.Error(error)
		return nil, error
	}

	// 返回查询结果
	return &lVMs, nil
}
```

#### 修改控制器

在 controllers 下的 hello.go 文件中，增加函数：

``` go
// @router /getvmsinfo [get]
func (c *HelloController) GetVMsInfo() {
	
	VMs, err := models.GetVMInfos()

	if err != nil {
		c.Data["json"] = err.Error()
	}

	c.Data["json"] = VMs
	c.ServeJSON()
}
```

并修改 URLMapping ：

``` go
func (c *HelloController) URLMapping() {
	c.Mapping("Hello", c.Hello)
	c.Mapping("GetVMsInfo", c.GetVMsInfo)
}
```

#### 测试输出

``` bash
curl -s "http://localhost:8080/v1/hello/getvmsinfo"
```

返回值：

``` json
[
  {
    "Id": 3626,
    "Name": "i-a5**********3-*****02",
    "Cluster": "*****02",
    "CreateTime": "2019-11-18T03:16:37+08:00"
  },
  ...
  {
    "Id": 3517,
    "Name": "i-3d**********1-c*****01",
    "Cluster": "******01",
    "CreateTime": "2020-09-13T06:07:06+08:00"
  }
]
```

### 一点缺陷

在上边的返回多行记录的例子中，控制器是这么调用函数的：

``` go
models.GetVMInfos()
```

package models 下（可能）有很多文件，每个文件又有很多函数，显然如果将一类函数关联到一个类型上，在被外部调用的时候会更明晰一些，比如这样的调用：

``` go
models.HelloModel.GetVMInfos()
```

在Go语言中，可以将一个方法关联到一个类型上，使得该方法只能通过该类型的实例来调用。这种关联关系就是方法和类型之间的绑定。在这段代码中，`GetVMInfos()` 方法被绑定到 `helloModel` 类型上，所以只能通过 `HelloModel` 变量来调用该方法。

如需实现这种效果，需要修改 models 下的 hello.go 文件。

在 import 下面，追加如下两行：

``` go
// `helloModel`是一个无字段的结构体类型，它的目的是作为接收者类型，
// 即方法所属的类型。
type helloModel struct{}

// `HelloModel`是一个全局变量，它是一个指向`helloModel`类型的指针。
// 由于Go语言中允许通过指针来调用方法，所以可以使用
// `models.HelloModel.GetVMInfos()`这样的方式来调用`GetVMInfos()`方法。
var HelloModel *helloModel
```

修改 `GetVMInfos()` 函数，从：

``` go
func GetVMInfos() (VMs *[]VMInfo, err error)
```

修改为：

``` go
func (*helloModel) GetVMInfos() (VMs *[]VMInfo, err error)
```

加了函数头：
``` go
(*helloModel)
```
表明 `GetVMInfos()` 是一个方法，它属于 `helloModel` 类型。