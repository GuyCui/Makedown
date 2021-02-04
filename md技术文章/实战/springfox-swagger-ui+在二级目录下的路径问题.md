---
name: swagger2-basepath
title: springfox-swagger-ui 在二级目录下的路径问题
date: 2021-02-04 14:14:51
tags: swagger2-basepath
categories: 
---
本文解决 springfox-swagger-ui 在二级目录下的使用问题。如同一个域名的 `/user` 和 `/post` 用 Nginx 分别反向代理指向不同的应用，我们希望在每个应用中都可以正常使用 Swagger。

下面，我们假设要配置 /user 指向 user 应用。

很多人爱折腾，会打起修改源码的主意，希望本文能帮你节省点时间。

**注意：本文使用的 springfox-swagger2 版本是 2.6.0**

## 修改全局 context path

Spring Boot 环境中只要配置以下环境变量即可：

```
server.contextPath=/user
```

那么你的所有的接口，默认就都是在 `/user` 下面了，自然 swagger-ui 也就能正常使用了，访问 `/user/swagger-ui.html` 即可。

这是最最简单的方法，不过在有些特定的环境中会有问题，比如我司：

由于设置了 contextPath，那么健康检测接口 `/health` 也会被自动换为 `/user/health`，而我们的发布系统一根筋地要找 `/health` 接口，也就导致我们的应用会发布不了。

**只有在碰到这种方法解决不了的时候，我们才要考虑使用下面介绍的方法。**

## 使用 Controller 做 forward

首先，将 **/user/swagger-ui.html** forward 到 **/swagger-ui.html**。

```java
@GetMapping("/swagger-ui.html")
public String index() {
    return "forward:/swagger-ui.html";
}
```

剩下的其实很简单，大家打开 your-domain/user/swagger-ui.html 页面，发现可以看到 swagger-ui 页面，但是接口列表没出来。

然后打开浏览器控制台，就会发现，它有很多发向 /user/webjars/… 和 /user/swagger… 的请求都是 404，我们只要一一把这些请求搞定就 OK 了。

还是少说些废话，直接看下代码，大家就清楚了：

```java
@Controller
// 看这里
@RequestMapping("user")
public class SwaggerController extends BaseController {

    @GetMapping("/swagger-ui.html")
    public String index() {
        return "forward:/swagger-ui.html";
    }

    @GetMapping("/webjars/springfox-swagger-ui/css/{s:.+}")
    public String css(@PathVariable String s) {
        return "forward:/webjars/springfox-swagger-ui/css/" + s;
    }

    @GetMapping("/webjars/springfox-swagger-ui/{s:.+}")
    public String baseJs(@PathVariable String s) {
        return "forward:/webjars/springfox-swagger-ui/" + s;
    }

    @GetMapping("/webjars/springfox-swagger-ui/lib/{s:.+}")
    public String js(@PathVariable String s) {
        return "forward:/webjars/springfox-swagger-ui/lib/" + s;
    }

    @GetMapping("/webjars/springfox-swagger-ui/images/{s:.+}")
    public String images(@PathVariable String s) {
        return "forward:/webjars/springfox-swagger-ui/images/" + s;
    }

    @GetMapping("/swagger-resources/configuration/ui")
    public String ui() {
        return "forward:/swagger-resources/configuration/ui";
    }

    @GetMapping("/swagger-resources")
    public String resources() {
        return "forward:/swagger-resources";
    }

    @GetMapping("/v2/api-docs")
    public String docs() {
        return "forward:/v2/api-docs";
    }

    @GetMapping("/swagger-resources/configuration/security")
    public String security() {
        return "forward:/swagger-resources/configuration/security";
    }
}
```

## 使用 ViewControllerRegistry

很多人会使用下面的方法来写，我们也来看一下行不行：

```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/user/**").addResourceLocations("classpath:/META-INF/resources/");
    }
    
	@Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addRedirectViewController("/user/v2/api-docs", "/v2/api-docs").setKeepQueryParams(true);
        registry.addRedirectViewController("/user/swagger-resources/configuration/ui","/swagger-resources/configuration/ui");
        registry.addRedirectViewController("/user/swagger-resources/configuration/security","/swagger-resources/configuration/security");
        registry.addRedirectViewController("/user/swagger-resources", "/swagger-resources");
    }
}
```

这种写法，访问静态资源的时候是完全没有问题的，但是 swagger-ui.html 页面在使用 ajax 调用接口的时候，这种配置做的是**跳转**，如 "/user/v2/api-docs" 自动跳转到 "/v2/api-docs" 其实是不满足我们需求的，因为 `/v2/api-docs` 这个路径根本就不会跳到我们的 user 应用。

（全文完）