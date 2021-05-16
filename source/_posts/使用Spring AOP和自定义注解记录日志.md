---
title: 使用Spring AOP和自定义注解记录日志
date: 2019-03-26 21:57:00
tags: [Spring集合]
categories: 编程
---

# 自定义注解

```
//标示该注解用于方法上
@Target(ElementType.METHOD)
//标示该注解可以在运行时通过反射找到
@Retention(RetentionPolicy.RUNTIME)
public @interface Log {
	String value() default "";
}
```
1. @Retention
   作用：标示注解在什么时候可见（运行时可见、仅在.class文件及源代码中可见、仅在源代码中可见），value可用参数有：


| **属性值**     |   **作用**   |
| ---- | ---- |
|RetentionPolicy.RUNTIME   |标示该注解可以再运行时通过反射找到（ORM框架许多注解使用了该参数）      |
|RetentionPolicy.CLASS   |标示该注解保存在.class文件中，但在运行时不能通过反射找到      |
|RetentionPolicy.SOURSE|标示该注解只在源码中可见              |


2. @Target
   作用：标示该注解用于注解什么元素（类、方法、变量等），value可用参数有：


| **属性值**                      | **作用**                             |
| --------------------------- | ------------------------------------ |
| ElementType.PACKAGE         | 标示该注解用于注解包声明             |
| ElementType.ANNOTATION_TYPE | 标示该注解用于注解其他注解           |
| ElementType.CONSTRUCTOR     | 标示该注解用于注解构造函数           |
| ElementType.FIELD           | 标示该注解用于注解成员变量           |
| ElementType.METHOD          | 标示该注解用于注解方法               |
| ElementType.TYPE            | 标示该注解用于注解类，接口，枚举类型 |
| ElementType.PARAMETER       | 标示该注解用于注解参数               |
| ElementType.LOCAL_VARIABLE  | 标示该注解用于注解局部变量           |

3. value为操作信息

# 自定义切面
```
@Component
@Aspect
@Slf4j
public class LogAspect {

    @Autowired
    private LogService logService;

    private long currentTime = 0L;

    /**
     * 配置切入点
     */
    @Pointcut("@annotation(com.**.aop.log.Log)")
    public void logPointcut() {
        // 该方法无方法体,主要为了让同类中其他方法使用此切入点
    }

    /**
     * 配置环绕通知,使用在方法logPointcut()上注册的切入点
     *
     * @param joinPoint join point for advice
     */
    @Around("logPointcut()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = null;
        currentTime = System.currentTimeMillis();
        result = joinPoint.proceed();
        Log log = new Log("INFO",System.currentTimeMillis() - currentTime);
        //保存日志信息到数据库
        logService.save(getUsername(), StringUtils.getIP(RequestHolder.getHttpServletRequest()),joinPoint, log);
        return result;
    }

    /**
     * 配置异常通知
     *
     * @param joinPoint join point for advice
     * @param e exception
     */
    @AfterThrowing(pointcut = "logPointcut()", throwing = "e")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable e) {
        Log log = new Log("ERROR",System.currentTimeMillis() - currentTime);
        log.setExceptionDetail(ThrowableUtil.getStackTrace(e).getBytes());
        logService.save(getUsername(), StringUtils.getIP(RequestHolder.getHttpServletRequest()), (ProceedingJoinPoint)joinPoint, log);
    }

    public String getUsername() {
        try {
            return SecurityUtils.getUsername();
        }catch (Exception e){
            return "";
        }
    }
}
```

# 用法

在Controller方法上使用@Log
如：@Log("查询用户")