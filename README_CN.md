# ec-router [![License](https://img.shields.io/badge/license-MIT-blue.svg)](http://opensource.org/licenses/MIT)

一个简单易用的koa2路由中间件，提供规则路由功能，不再需要复杂无趣的路由文件，路由影射表等。

同时提供无代码自动实现RESTful服务的功能（已支持使用mysql或者mongodb作为存储）


## 安装

```
npm install ec-router --save
npm test

```

也可从 [git仓库](https://github.com/tim1020/ec-router) 中下载源码，放到node_modules目录

## URI格式说明

``` http://domain[:port]/[prefix]/[apiver]/path```

**[prefix]** : 表示路径前缀（可为空），一般用来在同一服务器部署时区分不同的应用，可在config中设置
**[apiver]** : 表示api的版本，在config中设置 apiVer 为true时生效，版本规则由apiVeRegex定义，默认规则为两位版本号，比如v1.0,v11,v2,11,12.22
**path** : 为具体资源路径，根椐不同的路由类型有不同

## 路由类型(使用type参数设置)

### type=1, RESTful方式

本方式使用RESTful访问，根椐请求方法和请求的资源名称、ID来处理，比如：

```
GET /res/12
POST /user
PUT /user/11
```

其中的"res"和"user"表示资源名称，中间件自动路由到以资源名称命名的controller文件中，并执行与请求方法对应的控制器方法(控制器方法名使用全小写).

如果对应的方法找不到，则继续查找名为“all”的控制器方法。

当请求同名方法和all方法都找不到，如果开启了自动RESTful服务，会进入到自动处理逻辑（下文进行说明），否则，响应404。

> 自动RESTful只支持POST/DELETE/PUT/GET方法，分别对应数据的增删改查


### type=2,Path方式

本方式根椐路径匹配控制器和控制器方法, path的格式为： /controller/action

当收到请求后，中间件会查找并执行对应的controller及其action，如果没找到，会尝试在控制器中查找名叫"all"的方法

当action和all都没找到，则响应404。

如:

**/res/list** 路由到 controller=res,method=list

**/user/add** 路由到 controller=user,method=add

**/user**    路由到 controller=user,method=all


该方法不区分请求方法，可在实现控制器时自动判断


### type=3,QueryString方式

本方式使用请求字符串进行路由判断，比如：**/apiName?c=controller&a=action**,查找controller及action的方式同type=2

(其中c,a为参数名称，可在config中修改)

如：

**[/?c=User&a=list]**  路由到 controller=User.js,method=list 

**[/?c=User&a=add]**  路由到 controller=User.js,method=add

**[/?c=User&a=]**  路由到 controller=User.js,method=all

此方法同样不区分请求方法，在实现控制器时由开发都进行判断处理

## 使用


### koa app main

```
//index.js
const Koa = require('koa')
const app = new Koa()
const ecRouter = require('ec-router')

process.env.NODE_ENV = 'dev' //开启debug log

//加载其它中间件
//如果需要自动RESTful服务，需要使用bodyParser之类的请求内容解释中间件来预处理请求参数
app.use(bodyParser())

//修改ec-router的默认配置
ecRouter.loadConfig('/home/code/koa/ec-config.js')
app.use(ecRouter.dispatcher())

//use other middleware

app.listen(3000)

```

### 热加载

当在配置文件中设置了hotLoad=true(缺省值)时，ec-router支持配置文件及controller的热加载(hotLoad配置的修改不支持热更新)

如果需要使用热加载，请将配置独立成模块，再使用 ```ecRouter.loadConfig('./config.js')``` 代替 ```ecRouter.setConfig(conf)```

### controller

> 默认地，需要将控制器文件放置在APP根目录下的controllers目录，可在配置中修改

> 如果开启了apiVer，则需在controllers目录下创建版本目录，并将控制文件放到相应的版本目录，如 : /v1/res，会路由到 controllers/v1/res.js。

> 如果新版本继承了旧版本的大量方法，可以用require导入原版本，再重新某个特定方法。

> 控制器文件名、控制器方法、资源名称等大小写敏感

> 控制器方法的函数原型是 (ctx) =>{} , ctx是koa2本身的ctx，如果type=1，ec-router会在ctx.req上面绑定resource和resrouceId

> type=1时,使用get,post,put,delete,all来命名对应的控制器方法，type非1时，可以自行定义（对应path或querystring中的action命称）


```

// controllers/user.js
module.exports = {
    get : (ctx) => {
		//ctx.req.resourceId  //effective when type=1,
        //ctx.req.resource
        ctx.body = "get User"
    },
    post: (ctx) => {
        ctx.body = "post user"
    },
    //可以在控制器方法中使用ctx.go()来进行内部重定向
    go:(ctx) => {
        //ctx.go('user','list') //ctx.go('controller','action')
        ctx.go('list') //ctx.go('action')

        //后面代码会继续执行（如果有的话）
    },
    //当action无法匹配以上方法时，会自动匹配为all方法
    all: (ctx) => {
        //other method
    }
}

```

### controller钩子

如果需要在每个控制器方法执行之前或之行都执行一些逻辑，可以使用钩子，方法是：

1. 在 controllers目录下放置控制器钩子,默认文件名为 _hook.js (名字可以通过在配置controllerHook来修改)
2. 在_hook.js中实现并导出before或after方法(可同时或单独前后添加钩子)

注：钩子是全局的，使用版本时，所有版本共用外层的钩子定义。

```

module.exports = {
    //do before all controller action
    before : (ctx) => {
        console.log('controller start')
    },

    //do after all controller action
    after: (ctx) => {
        console.log('controller finish')
    },
}

```


### config

> 可以在调用ec-router.dispatcher之前使用loadConfig来修改默认配置（config文件只需定义不使用默认值的项）

```
{
    type            : 1,                //路由方式
    uriApiName      : 'index',          //使用querystring方式时，指定API文件名，即/apiName?c=xx&m=xx
    uriCParam       : 'c',              //使用querystring方式时，指定控制器的参数名
    uriAParam       : 'a',              //使用querystring方式时，指定控制器方法的参数名
    uriPrefix       : '',               //API路径前缀，如: /prefix/controller/action
    uriDefault      : '/index',         //默认uri
    apiVer          : false,            //是否支持版本声明
    apiVeRegex      : /^v?(\d){1,2}(\.[\d]{1,2})?$/, //版本规则,
    controllerPath  : 'controllers',    //控制器文件所在目录，相对于app根目录
    controllerHook  : '_hook',			//控制器钩子名称
	allowMethod     : ['get','post','put','delete'] //允许的请求方法
    tbPrefix        : 'res_',           //使用自动RESTful时，数据表名称前缀,该前缀与资源名共同组成mysql的表名或mongodb的collection
    dbConf          : {                 //使用自动RESTful时所用的数据连接配置
        driver: 'mysql',				//使用的数据驱动，支持mysql 或者 mongodb
        connectionLimit : ,
        host            : '',
        port            : ,
        user            : '',
        password        : '',
        database        : ''
    }
    //mysql详细配置请参见：(https://github.com/mysqljs/mysql#pool-options)
}


```

## 自动RESTful服务

如果需要开启自动RESTful服务，需要设置以下参数：

```
type:1
dbConf:{
    
}
```

自动RESTful服务的原理是:

根椐请求的路径，解释出资源名称（对应数据表名）和资源项ID，并结合不同的请求方法，构建中相应的SQL语句，最后执行构建的SQL语句获得结果返回，本中间件的自动RESTful服务只支持简单的数据处理，复杂的逻辑可以自定义控制器和方法来覆盖自动逻辑。

整个RESTful请求的处理步骤如下：

1. 根椐请求路径解析出资源名称，查找以资源名称命名的控制器，并在里面查找以请求方法命名的控制器方法，如果能找到，则执行它并返回。
2. 如果找不到对应的方法，尝试查找名为"all"的控制器方法，如果找到，则执行该方法并返回。
3. 找不到控制器方法时，如果开启了自动RESTful服务（设置了有效的dbConf），中间件根椐请求方法、请求参数，自动构建相应的SQL语句，执行并返回结果

### 请求例子和生成的SQL （for mysql）

**[GET /task]**  

SELECT * FROM res_task

**[GET /task/12]** 

SELECT * FROM res_task WHERE id=12

**[POST /task]**

INSERT INTO res_task SET k=v,k=v (k,v 是post参数的键和值)

**[PUT /task/12]**

UPDATE res_task SET k=v,k=v WHERE id=12

**[DELETE /task/12]**

DELETE res_task WHERE id=12

### 补充说明

1. 自定义的控制器方法将会优于内置数据处理逻辑,换句话说，如果开启了自动RESTful服务，又想对某一资源的某一操作进行定制，则可以针对此实现一个控制器和方法来特定处理。这在常规处理无法满足，需要复杂的逻辑实现的资源请求很有用。

2. GET方法对应sql的select,PUT对应update,POST对应insert,DELET对应delete

3. PUT/DELETE必须设置资源ID,即 PUT /resurce/[:resourceId]

4. PUT/POST所插入或更新的数据来自于请求内容，ec-router并不解释请求body,所以你需要在use ec-router之前使用bodyParse之类的能解释请求body的中间件。

5. 自动RESTful返回格式约定为json，当发生错误，返回 {code:'0xxxx',error:'msg'},如果成功，则返回 {code:0,data:{xxx}} ,如返回不符合需求，可使用其它中间件进行转换。 


### select的参数

当使用get请求时，可使用更多的参数来定制生成的select语句，包括: where,order,limit，例如：

```
GET /task/?where=xxx&order=xxx&limit=xxx

```

> fields=a,b alias_b,c as alias_c //mongodb不支持别名方式

> order=field1,field2 desc

> limit=[offset,]nums  //当未设置limit，且没有where条件(有指定resourceId仅使用where id=:resourceId)，系统默认设置limit值,避免错误的返回了大量数据

> where条件待完善

> where=cond1 and cond2 [or] cond3 [and] cond4 or cond5 // (cond1 and cond2) or cond3 and (cond4 or cond5)

> where=cond1,cond2,cond3 //mongodb暂时只支持and条件

> 其中，cond表达式为 字段=值，目前支持的运算符包括： >=,>,<=,<,=, !=,<>, in

## License

MIT is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT).
