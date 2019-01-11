---
layout:     post
title:    轻松构建微服务之springboot
no-post-nav: true
category: other
tags: [arch]
excerpt: 注册中心的几种模式,路由和负载均衡
---

是一个快速开发spring应用的工具，习惯先于配置，继承了各种常用工具和spring集成，不在需要xml配置，通过java代码的方式来增加springmvc的拦截器等，内置一些默认的配置让应用快速启动，可以集成tomcat和jetty等web容器

好处是更适用于云原生项目，这样启动一个项目只需要一条命令 java - jar包，而不需要先把应用打成一个包让后放在tomcat目录下，设置calsspath等，而现在所有的应用打车一个jar包设置一个启动类，更方便制作docker镜像
