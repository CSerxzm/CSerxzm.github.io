---
title: redis页面缓存
date: 2021-5-22 22:13:00
comments: false
categories: 
- 实战
tags:
- redis
- 页面缓存
- 手动模板渲染
---

在以前涉及到的redis过程中，笔者主要是使用redis缓存mapper层查询的相关数据对象，没有接触到界面缓存对象，今天笔者有幸接触到相关tips。

开门见山，用事实说话，在没有使用缓存的时候，笔者用jmeter进行压测，在1000次的请求中，其吞吐量为473每秒，而后使用redis缓存页面的时候，吞吐量为747每秒，提升将近两倍，可见其提升幅度之大。

<!-- more -->

其中主要的思路是将theymeleaf模板的渲染手动来实现，将手动渲染的页面存入redis，有则直接返回，没有则手动渲染存入redis，并返回给用户。

```java
    //用于手动渲染界面
    @Autowired
    private ThymeleafViewResolver thymeleafViewResolver;

     /**
     * 跳转到商品列表页
     *   优化要点：缓存页面，如果redis中存在页面，就返回，没有就手动渲染
     * @param model
     * @param user
     * @return
     */
    @RequestMapping(value="/toList",produces = "text/html;charset=utf-8")
    @ResponseBody
    public String toList(Model model,User user,HttpServletRequest request,HttpServletResponse response){
        ValueOperations valueOperations = redisTemplate.opsForValue();
        String html = (String) valueOperations.get("goodsList");
        if(!StringUtils.isEmpty(html)){
            return html;
        }
        model.addAttribute("user",user);
        model.addAttribute("goodsList",goodsService.findGoodsVo());
        //如果为空，手动渲染存入redis手动返回
        WebContext webContext = new WebContext(request, response, request.getServletContext(), request.getLocale(), model.asMap());
        html = thymeleafViewResolver.getTemplateEngine().process("goodsList", webContext);
        if(!StringUtils.isEmpty(html)){
            valueOperations.set("goodsList",html,60, TimeUnit.SECONDS);
        }
        return html;
    }
```

其代码实现如上，发现redis的操作与之前的操作一致，使用的都是redisTemplate，不同的是涉及到手动的渲染界面。其中`WebContext`和`ThymeleafViewResolver`对象至关重要。为了使页面的变化让用户能够感知，我们也设置了redis的缓存失效时间，如60秒。

还有更多关于并发的优化问题，秒杀的超卖问题，源码见github，网址是[seckill源码](https://github.com/CSerxzm/seckill) 。