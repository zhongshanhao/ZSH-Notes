### SpringMVC

web服务器三层架构

 - 表现层、业务层、数据访问层

MVC（与服务器三层架构不同，MVC都处在表现层）

- Controller：控制层
- Model：模型层
- View：视图层

核心组件： 前端控制器（DispatcherServlet）(基于spring容器)，MVC这三层都是由前端控制器(DispatchServlet)去调用的, DispatcherServlet是满足 javaee 的规范的，javaee提出了Servlet接口，DispatcherServlet是这个接口的实现类。

![springmvc请求处理流程](https://gitee.com/Krains/FigureBed/raw/master/img/springmvc%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)

springmvc处理客户端请求的流程

浏览器访问服务器，前端控制器接受到请求，根据浏览器所访问的地址与方法上@RequestMapping("\path")所写路径匹配，找到对应处理该请求的controller，把请求交给controller，controller调用业务组件处理请求，业务层调用数据访问层得到数据，将数据返回给controller，controller得到数据并且封装数据到model里，返回给前端控制器，前端控制器将model传给view层，视图层利用Model数据生成html页面放回给前端控制器，由前端控制器将html页面返回给浏览器。

视图层如何利用model数据生成html页面呢？

我们可以使用模板引擎，它能够利用model数据和模板文件生成动态的html页面。

目前较流行的模板引擎：Thymeleaf，以html文件为模板。

### Thymeleaf

<html lang="en" xmlns:th="http://www.thymeleaf.org">

声明该html文件是一个模板，模板语法来自thymeleaf官网

${变量名}

官网

https://www.thymeleaf.org

### springmvc统一处理异常

由于请求调用的顺序是从表现层到业务层到数据访问层，因此这三层的抛出的异常最终都会汇总到表现层，我们在表现层可以统一处理异常，springmvc为我们提供了解决方案。

定义一个ExceptionAdvice处理异常的类，在类上加上注解@ControllerAdvice，在加上处理异常的方法就好。

> ​        @ControllerAdvice
>
> - 用于修饰类,表示该类是Controller的全局配置类。
> - 在此类中,可以对Controller进行如下三种全局配置:
>   异常处理方案、绑定数据方案、绑定参数方案。
>   @ExceptionHandler
> - 用于修饰方法,该方法会在Controller出现异常后被调用,用于处理捕获到的异常。
>   @ModelAttribute
> - 用于修饰方法,该方法会在Controller方法执行前被调用,用于为Model对象绑定参数。
>   @DataBinder
> - 用于修饰方法,该方法会在Controller方法执行前被调用,用于绑定参数的转换器。

```java
// springmvc统一异常处理配置类
// 所有带controller注解的类都能被扫描到
@ControllerAdvice(annotations = Controller.class)
public class ExceptionAdvice {
    private static final Logger logger = LoggerFactory.getLogger(ExceptionAdvice.class);

    @ExceptionHandler({Exception.class})
    public void handleException(Exception e, HttpServletRequest request, HttpServletResponse response) throws IOException {
        logger.error("服务器发生异常: " + e.getMessage());
        for (StackTraceElement element : e.getStackTrace()) {
            logger.error(element.toString());
        }

        String xRequestedWith = request.getHeader("x-requested-with");
        if ("XMLHttpRequest".equals(xRequestedWith)) { // 异步请求出现异常时
            response.setContentType("application/plain;charset=utf-8");
            PrintWriter writer = response.getWriter();
            writer.write(CommunitiesUtil.getJSONString(1, "服务器异常!"));
        } else {
            response.sendRedirect(request.getContextPath() + "/error");
        }
    }
}
```

注解

```java
// 写在方法上
@RequestMapping(path = "/students" ,method = RequestMethod.GET)    // 将请求路径与方法绑定，
@ResponseBody　　//　声明返回的不是模板文件的位置，即直接返回数据给浏览器

// 写在参数列表上
@RequestParam(name = "current", required = false, defaultValue = "1") int current
    
// 获取get请求路径中的参数，有两种方法

    // /students?current=1&limit=20，
     // DispatcherServlet会将路径中的参数自动注入到参数列表中，参数名与变量名相同，
    // 可以使用注解＠RequestParam做默认值的处理
    @RequestMapping(path = "/students" ,method = RequestMethod.GET)
    @ResponseBody
    public String getStudents(
            @RequestParam(name = "current", required = false, defaultValue = "1") int current,
            @RequestParam(name = "limit", required = false, defaultValue = "1") int limit){
        System.out.println(current);
        System.out.println(limit);
        return "some students";
    }

	// /student/123
    @RequestMapping(path = "/student/{id}", method = RequestMethod.GET)
    @ResponseBody
    public String getStudent(@PathVariable("id") int id){　　//@PathVariable能够获取路径中的id，即123
        System.out.println(id);
        return "a student";
    }
```