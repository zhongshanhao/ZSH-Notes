REST 是一种编码风格，这种风格就是URL定位资源，用HTTP动词（GET,POST,DELETE,DETC）描述操作。

满足这些约束条件和原则的应用程序或设计就是 RESTful。

RESTful API就是有REST风格的API。

RESTful API就是满足一种设计原则和规范接口，这种原则和规范就是URL定位资源，用HTTP动词（GET,POST,DELETE,DETC）描述操作。

RESTful API的设计原则和规范

使用URI定位资源

用HTTP方法（GET、POST、PUT、DELETE）描述操作，GET（读取）、POST（新建）、PUT（更新）、DELETE（）

无状态：定位资源时与其他资源无关



安全性：安全的方法不会修改资源状态，尽管多次调用的返回值可能不一样(被其他非安全方法修改过)。

幂等性：是指该方法多次调用返回的效果(形式)一致，客户端可以重复调用并且期望同样的结果。

HTTP方法的安全性和幂等性见下表：

| 方法名  | 安全性 | 幂等性 |
| ------- | ------ | ------ |
| GET     | 是     | 是     |
| HEAD    | 是     | 是     |
| OPTIONS | 是     | 是     |
| DELETE  | 否     | 是     |
| PUT     | 否     | 是     |
| POST    | 否     | 否     |



比如，我们有一个friends接口，对于“朋友”我们有增删改查四种操作，怎么定义REST接口？

增加一个朋友，uri: generalcode.cn/v1/friends 接口类型：POST

删除一个朋友，uri: generalcode.cn/va/friends 接口类型：DELETE

修改一个朋友，uri: generalcode.cn/va/friends 接口类型：PUT

查找朋友，uri: generalcode.cn/va/friends 接口类型：GET

上面我们定义的四个接口就是符合REST协议的，请注意，这几个接口都没有动词，只有名词friends，都是通过Http请求的接口类型来判断是什么业务操作。

举个反例：generalcode.cn/va/deleteFriends 该接口用来表示删除朋友，这就是不符合REST协议的接口。

一般接口的返回值是JSON或者XML类型的，笔者在互联网的工作经验一般都是JSON类型的。

用HTTP Status Code传递Server的状态信息。比如最常用的 200 表示成功，500 表示Server内部错误，403表示Bad Request等。（反例：传统web开发返回的状态码一律都是200，其实不可取。）

那这种风格的接口有什么好处呢？前后端分离。前端拿到数据只负责展示和渲染，不对数据做任何处理。后端处理数据并以JSON格式传输出去，定义这样一套统一的接口，在web，ios，android三端都可以用相同的接口，是不是很爽？！