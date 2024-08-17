### SpringBoot

核心作用

- 起步依赖、自动配置、端点控制

### 起步依赖

只需要写几个依赖，就能引入一堆构建项目所需要的jar包

  ![springboot起步依赖](https://gitee.com/Krains/FigureBed/raw/master/img/springboot%E8%B5%B7%E6%AD%A5%E4%BE%9D%E8%B5%96.png)

按住ctrl鼠标左键点击spring-boot-starter-web进去，也是能看见引用了一堆jar包，springboot帮助我们快速构建web项目所依赖的jar包，这就是起步依赖的作用。

在com.springboot.demo路径下新建一个controller包，新建一个类，在类上写上@Controller，配置好访问路径，返回hello spring boot ，然后启动服务，在浏览器上输入对应路径就可以看到hello spring boot，虽然简单，但是这说明我们的项目可以处理客户端的请求了。

### 自动配置

使用springboot可以统一管理配置，能够自动配置，几乎不用做配置就可以启动服务。

在springboot中，统一用一个配置文件管理项目中的配置，这个文件就是application.properties，我们查阅官方文档，查看配置的方法。

![springboot配置文件](https://gitee.com/Krains/FigureBed/raw/master/img/springboot%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.png)

向下翻阅，可以看到很多工具的配置方法，如thymeleaf：

![thymeleaf配置](https://gitee.com/Krains/FigureBed/raw/master/img/thymeleaf%E9%85%8D%E7%BD%AE.png)

我们在applicationi.properties文件中配置了spring.thymeleaf.cache=false,就能配置thymeleaf关闭缓存功能，它是如何进行配置的呢，我们发现上面的写了一些注释，括号中写了一个类，这个类又使用了ThymeleafProperties作为配置类：

![ThymeleafPropertiespes配置类](https://gitee.com/Krains/FigureBed/raw/master/img/ThymeleafPropertiespes%E9%85%8D%E7%BD%AE%E7%B1%BB.png)

我们写的spring.thymeleaf.cache=false实际上就是给类ThymeleafProperties中的cache赋值，其默认值为true，我们配置为false。 

自动配置还能够在程序开始时将一些starter包内的对象加入容器进行管理，从而我们可以在程序中直接注入就可以使用了。

###　端点监控

项目上线后可以用springboot进行项目的监控。

###　快速构建项目

![使用springboot构建项目](https://gitee.com/Krains/FigureBed/raw/master/img/springboot%E6%9E%84%E5%BB%BA%E9%A1%B9%E7%9B%AE.png)

填写项目信息，注意选择对应自己的java版本，在右边可以导入项目依赖，刚开始做项目的时候不知道使用什么jar包可以在创建项目之后在pom文件中导入，最后在下方点击generate下载创建好的初始项目，解压导入IDEA即可完成项目创建。

导入项目到IDEA后，依赖下载完成后，我们初始的web项目就创建好了，项目的结构如下：

![springboot初始项目结构](https://gitee.com/Krains/FigureBed/raw/master/img/springboot%E5%88%9D%E5%A7%8B%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png)

web项目结构已经被构建好了，可以看到右边Dependencies中是我们开始导入的依赖，我们导入依赖时仅仅导入了3个依赖，依赖中包含着一堆jar包，细心的我们发现，右边的依赖中包含tomcat，也就是说springboot将tomcat也集成到了jar包里，我们不用自己在去下载工具tomcat了，这省下了我们配置tomcat，测试项目时启动tomcat的时间了，真是非常省力呢。

写一个demo实现处理的客户端请求的案例

![springbootDemo](https://gitee.com/Krains/FigureBed/raw/master/img/springbootDemo.png)

在com.springboot.demo路径下新建一个controller包，新建一个类，写好以上几行代码，然后启动服务，在浏览器上输入localhost:8080/alpha/hello就可以看到hello spring boot，虽然简单，但是这说明我们的项目可以处理客户端的请求了。

![hello](https://gitee.com/Krains/FigureBed/raw/master/img/hello.png)

以上就是使用springboot快速构建一个web项目，非常方便快捷。

https://spring.io
