# 从类库到服务

:::tip 通过服务来实现组件

Microservice architectures will use libraries, but their primary way of componentizing their own software is by breaking down into services.

微服务架构也会使用到类库，但构成软件系统组件的主要方式是将其拆分为一个个服务。

:::right

—— [Martin Fowler](https://martinfowler.com/) / [James Lewis](https://twitter.com/boicy), [Microservices](https://martinfowler.com/articles/microservices.html), 2014

:::

微服务架构其中一个重要设计原则是“通过服务来实现独立自治的组件”（Componentization via Services），微服务强调通过“服务”（Service）而不是“类库”（Library）来构建组件，两者的差别是类库是在编译期静态链接到程序中的，通过本地调用来提供功能，而服务是进程外组件，通过远程调用来提供功能。基于服务来构建程序，迫使微服务在复杂性与执行性能方面作出了极大的让步，换来的是软件系统“整体”与“部分”的物理层面的真正的隔离。

