[TOC]

# 1.技术栈
## 1.1 前端
- 前端markdown编辑器：mavon-editor
- 管理后台js框架：vue
- 用户端UI框架：bootstrap

## 1.2 后端
- 服务端页面渲染：thymeleaf
- 数据库：mysql
- 持久层框架：mybatis
- 数据库连接池管理：hikaricp
- 数据库分页插件：github pagehelper
- mvc框架：spring mvc
- 应用层容器：spring boot
- json序列化工具：fastjson

# 2. 功能列表

```$xslt
用户端
    标签
        查看详情
        筛选文章/问答
    文章
        写文章
        编辑
        删除
        评论
        点赞
        查看详情
    问答
        提问题
        编辑
        删除
        查看详情
        评论
        关注
        设置评论为最佳答案
        筛选已解决问题
        筛选未解决问题
    用户
        登录
        查看详情
        编辑个人资料
        更新登录密码
        关注好友
        查看粉丝
    关注
        关注的用户文章/问答
        关注的问答
        评论的问答
        点赞的文章
        评论的文章
    搜索
        根据文章/问答标题/内容模糊搜索
        
```


# 3. 各模块功能
![img.png](readmeImg/img.png)
- starter模块：项目启动入口模块，SpringBootApplication启动类就在这个模块，这个模块也会把一些通过依赖倒置实现的功能infrastructure模块依赖进项目中；
- portal模块：view层，系统门户模块，用户和要访问的页面都在这里，系统提供的http接口也都在这里；
- api模块：系统的对外接口
- facade模块：对api模块接口的实现，以及对接口请求参数的合法性校验，不做任何业务逻辑处理，将校验合法的请求转发给app模块进行业务处理；
- app模块：service层，业务用例处理模块，在domain模块定义了具体服务的接口infrastructure模块实现了这些接口的，用于实现业务。
- common模块：公共枚举的定义、工具类、自定义业务异常等都在这里；
## 3.1 用户
1. 登录
+ (1)forumportal/src/main/resources/templates/include/navbar.html路径下点击登录 弹出id为loginModal的登录表单，前端js代码发出一个post请求
+ (2)在 forum-api的model定义了一个ResultModel包含状态码、msg和泛型数据等定义
+ (3)【api】在api的request里面定义了登录或注册的请求参数，例如登录需要email和password，同时标记@Data生成getter setter方法
+ (4）【facade】在facade模块impl包下对api模块接口实现，请求参数合法性校验,合法性校验是validator包下的UserValidator里面的静态方法，所以可以使用类名.方法名直接调用。而具体检查参数使用的是forum-common 模块support包下的工具类CheckUtil，检查请求路径参数是否为空
+ (5)【facade】 在登录接口的实现类中使用facade模块自己的工具类ResultModelUtil来返回一个包装好的，符合ResultModel格式的结果。默认检查没有问题的话，只需要设置结果泛型data,调用底层返回ResultModel里面的泛型数据是String类型的token
+ (6)【app】 data的获取需要继续调用service层在forum-app模块中，首先进入UserManager，通过在api模块定义的UserEmailLoginRequest获取到传入的，并且经过facade模块检查非空的邮箱。Service层不仅仅是app模块还有domain模块定义了getByEmail接口通过email获取一个用户对象，而这个接口是在infrastructure模块实现的，而这里调用了持久层DAO，并调用infrastructure的工具类将DAO层查询到的结果转换为user类型
+ (7)【infrastructure】dao层也在infrastructure模块，UserDao使用了Mybatis。在xml中首先定义了resultMap，然后写sql语句，但是他这个sql语句是拆分了公共部分放在sql标签内，通过唯一的refid来引用。查询的结果封装为UserDO再转为User类型
+ (8) Service层获取到user对象后继续检查密码是否一样，使用common模块的工具类比较md5加密之后的密码，更新user对象的最后登录时间，最后调用DAO层的update跟新user数据库，最后调用AbstractLoginManagerd的login方法
+ (9)在这个login方法中使用了一个方法UUID.randomUUID().toString().replaceAll("-", "")来生成String类型token，清除之前的缓存，重新保存缓存,然后触发操作日志 最后返回token。
+ (10)portal系统门户(post请求➕@RequestBody将表单数据提交到api定义的UserRegisterRequest形参里面)->api接口->facade接口实现->app(service层)->domain定义获取数据库对象的接口->infrastructure实现上一个接口->dao使用mybatis查询
+ (11)数据流：因为登录服务只是验证。service验证完之后 token存入缓存并返回->ResultModel\<String>封装token的结果集合->portal模块在response请求中加入了一个sid就是token数据然后有返回了这个resultModel->js回调函数装入token
```js
 post('/user-rest/login', {
           'email': email,
                'password': md5(password)
            }, function (data) {
                localStorage.setItem('token', data);
                location.reload();
            });
```
2. 查看详情
- （1）在navbar.html 点击Dropdown下拉框的个人主页超链接，发送get请求通过DispatcherServlet扫描组件找到controller对应的Handler方法
- （2）根据userRequest里面的参数设置前端在个人主页下显示“文章”或者“提问”子页面 默认显示文章子页面
- （3）portal模块的controller调用service层继续获取用户信息，首先是app模块的Usermanager的info方法，该方法的interface定义在api模块，实现在app模块。app模块的info方法调用get方法，这个方法“get”的interface定义在domain模块，实现在infrastructure模块。接下来进入DAO层，首先调用dao的get方法，interface定义在infrastructure的dal.dao包里，mapper的xml定义在infrastructure的resource.mapper里面。
- （4）service层方法实现在infrastructure里面，将dao层查询到的对象转换为User类型。回到service层的app模块的info函数，检查user对象是否为空，再将user对象转换为UserInfoResponse类型。
- （5） service层获取到的UserInfoResponse当做泛型数据放入resultModel中
- （6） 将数据添加到request域中的model中，例如userInfoResponse数据，是否有粉丝（调用service层方法）、粉丝列表、关注列表等数据（通过service层方法）
- （7）返回string类型字符串被视图解析器解析，Thymeleaf对视图进行渲染，最终转发到视图所对应页面

3. 编辑个人资料
- （1）ThreadLocal里面存放着loginUser对象，更新信息首先检查是否登录以及userid是否正确
- （2）弹出来的是更新个人基本信息的弹窗，修改之后点解更新就能够直接更新。旁边还有修改密码的按钮。
- （3）修改个人信息在前端js代码逻辑里面首先初步检查是否为空，长度是否超过设定值，然后提交一个post请求。
- （4）接着调用service层的updateInfo函数，参数是自动装入的UserUpdateInfoRequest类型数据，这个函数interface定义在api模块，实现首先在app模块，同时也调用了domin和infrastructure实现的函数，也更新了缓存中用户登录信息。使用了pub.developers.forum.app.support 的LoginUserContext更新缓存

4. 更新登录密码

   

5. 关注好友

   

6. 查看粉丝
	主要是读取model里面的followerList对象数据，这里需要注意的就是api模块定义了一个PageRequestModel类型，里面有一个泛型数据

# 6. 部分页面展示

## 用户页面展示

- 首页

- 问答页

- 关注页

- 消息列表页

- 文章详情页

- 标签详情页

- 搜索页

- 用户主页

- 写文章页

