微服务架构是一种结构模式，它提倡将单一应用程序划分成一组小的服务，服务之间互相协调、互相配合，为用户提供最终价值。每个服务运行在其独立的进程种，服务与服务间采用轻量级的通信机制相互协作（通常是基于HTTP协议的RESTful API）。每个服务都围绕着具体业务进行构建，并且能够被独立的部署到生产环境、类生产环境等。

Spring Cloud

是微服务架构的一种落地实现，能够简单构建地分布式系统，是分布式微服务架构的一站式解决方案，是多种微服务架构落地技术（服务注册与发现、服务负载与调用、服务熔断降级、服务网关、服务分布式配置服务SpringCloud Coonfig、服务开发SpringBoot）的集合体，俗称微服务全家桶，架构图：

![image-20210316105755195](https://gitee.com/Krains/FigureBed/raw/master/img/image-20210316105755195.png)

Cloud技术

![image-20210316112723952](https://gitee.com/Krains/FigureBed/raw/master/img/image-20210316112723952.png)









