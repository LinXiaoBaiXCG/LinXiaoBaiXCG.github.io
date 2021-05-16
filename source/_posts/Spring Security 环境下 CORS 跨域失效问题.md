---
title: Spring Security环境下CORS跨域失效问题
date: 2019-05-12 20:00:00
tags: [Spring集合]
categories: 编程
---

> 前后端分离项目中，后端已经配置对跨域支持，前端还是报请求跨域问题。

# 基于Spring MVC正常配置跨域问题

```
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    private CorsConfiguration buildConfig() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        corsConfiguration.addExposedHeader("Authorization");
        return corsConfiguration;
    }

    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", buildConfig());
        return new CorsFilter(source);
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")
                .allowedHeaders("*")
                .maxAge(3600)
                .allowCredentials(true);
    }

}
```

配置完成后前端控制他还是显示请求跨域问题，说明上面的配置失效了。

# 跨域失效原因

启动 spring boot 项目，观察控制台打印的过滤器 chain 的顺序 cors filter 是在 spring security filter chain 之后。

```
o.s.b.w.s.FilterRegistrationBean - Mapping filter: 'characterEncodingFilter' to: [/*]
o.s.b.w.s.FilterRegistrationBean - Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
o.s.b.w.s.FilterRegistrationBean - Mapping filter: 'httpPutFormContentFilter' to: [/*]
o.s.b.w.s.FilterRegistrationBean - Mapping filter: 'requestContextFilter' to: [/*]
o.s.b.w.s.DelegatingFilterProxyRegistrationBean - Mapping filter: 'springSecurityFilterChain' to: [/*]
o.s.b.w.s.FilterRegistrationBean - Mapping filter: 'corsFilter' to: [/*]
o.s.b.w.s.ServletRegistrationBean - Servlet dispatcherServlet mapped to [/]
```

 所以，问题很清晰了，问题在于 spring security 的 过滤器链，我们只需要将手动创建的跨域 filter 置于 spring security filter chain 之前即可。 

# 解决

1. 在Spring Security配置文件中configure方法开启跨域

```
@Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .cors().
                and()
                .authorizeRequests()
                ……
                }
```

2. 手动实现跨域过滤器并将过滤器置于spring security filter chain之前

```
/**
 * @Author: lcq
 * @Description: 手动生成过滤器
 * @Date: 2019-05-11 16:55
 */
public class WebSecurityCorsFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletResponse res = (HttpServletResponse) response;
        res.setHeader("Access-Control-Allow-Origin", "*");
        res.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE, PUT");
        res.setHeader("Access-Control-Max-Age", "3600");
        res.setHeader("Access-Control-Allow-Headers", "Authorization, Content-Type, Accept, x-requested-with, Cache-Control");
        chain.doFilter(request, res);
    }
    @Override
    public void destroy() {
    }
}
```

```
.and().addFilterBefore(new WebSecurityCorsFilter(), ChannelProcessingFilter.class) // 添加前置过滤器，保证跨域的过滤器首先触发
```

