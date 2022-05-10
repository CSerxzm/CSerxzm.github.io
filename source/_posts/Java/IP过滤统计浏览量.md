---
title: IP过滤统计浏览量
date: 2021-7-19 10:20:00
comments: false
categories:
  - 实战
tags:
  - redis
  - IP过滤
---

### 前言

在昨天发布的关于注解结合切面编程时候，在大约 1 天不到的时间，发现该 blog 的浏览量到达 300 多，由于这是个人博客，也只有几个好友知道它的网址，由于 blog 的质量问题，能够访问的也是少之又少，那么 300 多的浏览量是怎么贡献的喃？除了从好友处获得的流量，就是从 github 和 gitee 处获得的，以及少之又少的从简历上知道链接的面试官。

### 问题发现

作者通过对该网页访问，发现一次点击中，假如你退出又进入就会统计两次，在发布之后，我也会是不是访问该博客是否有错别字和排版错误等问题，浏览量自然而然就会增加。

### 问题解决

通过思考，我们可以对 IP 进行检测，当同一个 IP 在一段时间内访问同一博客，就不会增加 redis 中的浏览量，这样统计的浏览量也会较真实。由于这种数据后期不需要查看，所以不需要进行数据表的设计，直接将 IP 和博客标识进行存储，如果在一小时之内，redis 中含有该键值，说明用户在这一个小时内访问过该博客，否则没有访问过，更新浏览量。

为了让业务逻辑代码中尽量减少代码的冗余、杂乱，我采用的是 spring aop 来实现，其中具体代码如下；

```java
@Aspect
@Component
public class BlogAspect {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Pointcut("execution(* com.xzm.blog.service.BlogService.selectAndConvert(..))")
    public void log() {
    }

    @Around(value="log()")
    public Object aroundMethod(ProceedingJoinPoint joinPoint) throws NotFoundException,Throwable {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        String ip = request.getRemoteAddr();
        Integer id = ((Integer) joinPoint.getArgs()[0]);
        Blog blog = null;
        if(RedisUtils.isEmpty(BlogConstant.ONEBLOG + id)){
            blog =  (Blog) joinPoint.proceed();
            if(blog!=null){
                RedisUtils.hPut(BlogConstant.ONEBLOG + id,"blog", JSON.toJSONString(blog));
                RedisUtils.hPut(BlogConstant.ONEBLOG + id,"views",blog.getViews().toString());
            }
        }else{
            //存在该blog
            blog = JSON.parseObject(RedisUtils.hGet(BlogConstant.ONEBLOG + id,"blog").toString(),Blog.class);
            String views = (String)RedisUtils.hGet(BlogConstant.ONEBLOG + id,"views").toString();
            blog.setViews(Integer.valueOf(views));
        }
        //校验IP
        String key = BlogConstant.BLOGANDIMAP + id +":"+ ip;
        if(blog!=null && RedisUtils.isEmpty(key)){
            //不存在,则放入redis，一个xia时过期
            RedisUtils.set(key,"1");
            blog.setViews(blog.getViews()+1);
            RedisUtils.hIncrement(BlogConstant.ONEBLOG + id,"views");
        }
        logger.info("ip:{} blog:{}",ip,blog.toString());
        return blog;
    }

}
```

其中切点为`* com.xzm.blog.service.BlogService.selectAndConvert(..)`,为前端访问 blog 时调用的方法，为了让 redis 的操作集中于一处，对 blog 的相关缓存也用切面实现，在进入原有方法之前，首先判断 redis 中是否有该 blog 的内容。没有的话进入原方法，否则直接将 redis 中数据取回，以便后面返回该内容。

```java
if(RedisUtils.isEmpty(BlogConstant.ONEBLOG + id)){
    blog =  (Blog) joinPoint.proceed();
    //存入redis
}else{
    //存在该blog，从redis中取出，赋值给blog
}
//校验IP,此处代码省略
return blog;
```

上面的代码为 blog 的逻辑部分，在取出 blog 后，我们需要对 IP 和博客标识组成的键进行判断，已决定是否要更新浏览量。

```java
//校验IP
String key = BlogConstant.BLOGANDIMAP + id +":"+ ip;
if(blog!=null && RedisUtils.isEmpty(key)){
    //不存在,则放入redis，一个小时过期
    RedisUtils.set(key,"1");
    blog.setViews(blog.getViews()+1);
    RedisUtils.hIncrement(BlogConstant.ONEBLOG + id,"views");
}
```

### 结果

以上问题就这样得到解决，通过切面的方式，让浏览量的更新更简单。演示结果可见个人博客主页，发现在一小时之内，您的重复访问不会增加浏览量。
