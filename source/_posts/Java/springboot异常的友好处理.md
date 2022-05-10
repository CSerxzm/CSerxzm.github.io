---
title: SpringBoot异常的友好处理
date: 2020-11-20 22:13:00
comments: false
categories:
  - java
tags:
  - springboot
  - Exception
---

对于前面一篇 我们在验证不通过的时候，会出现异常的抛出，但是其异常并不是按照我们需要的方式进行抛出(封装成自定义的对象)，对于前端解析不太友好，我们需要进行修改。

```shell
Field error in object 'loginVo' on field 'mobile': rejected value [12345678901]; codes [IsMobile.loginVo.mobile,IsMobile.mobile,IsMobile.java.lang.String,IsMobile]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [loginVo.mobile,mobile]; arguments []; default message [mobile],true]; default message [手机号码格式错误]]
```

由此，我们需要进行封装，我们借助注释`@RestControllerAdvice`和`@ExceptionHandler`。

我们的异常得到期望的返回格式，就需要用到了`@ControllerAdvice`或者`@RestControllerAdvice`。而`ControllerAdvice` 和 `RestControllerAdvice` 区别也就是和 `Controller `和 `RestCOntroller` 一样了。
基于现在 springboot 和 rest 风格的开发，这里就阐述@RestControllerAdvice ，来解决异常返回统一格式

我问在方法上添加`@ExceptionHandler`注释，表示对哪种异常进行处理。如下，当出现异常，就会使用 ExceptionHandler 进行处理，可以对异常返回类型封装为 RespBean 对象。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public RespBean ExceptionHandler(Exception e) {
        if (e instanceof GlobalException) {
            GlobalException ex = (GlobalException) e;
            return RespBean.error(ex.getRespBeanEnum());
        } else if (e instanceof BindException) {
            BindException be = (BindException) e;
            RespBean respBean = RespBean.error(RespBeanEnum.BIND_ERROR);
            respBean.setMessage("参数校验异常" + be.getBindingResult().getAllErrors().get(0).getDefaultMessage());
            return respBean;
        }
        System.out.println("Global     local    " + e.getLocalizedMessage());
        System.out.println("Global     msg    " + e.getMessage());
        System.out.println("Global     cause   " + e.getCause());
        System.out.println("Global     class    " + e.getClass());
        System.out.println("Global     stackTrace     " + e.getStackTrace().getClass());
        System.out.println("***********************************************");
        return RespBean.error(RespBeanEnum.ERROR);
    }
}
```

那么对于后端的异常抛出，比如当验证用户不存在的时候，我们也可以显式的抛出异常。

```java
User user = userMapper.selectById(mobile);
if (null == user) {
    System.out.println("user = null");
    throw new GlobalException(RespBeanEnum.LOGIN_ERROR);
}
```

其中对于`GlobalException`是笔者自己定义的，其类为：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class GlobalException extends RuntimeException{

    private RespBeanEnum respBeanEnum;

}
```

这样也可以达到较友好的返回结果。

那么对于 404 这种 Not Found 的请求，抛出的虽然是异常`NoHandlerFoundException`，但我们依然不能进行处理，我们对 yml 文件进行如下修改。

```yml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false
```

由于上面的代码我们统一的使用 Exception，所以并不能区分何种异常，访问不存在的 URL，会返回如下的信息，需要进行进一步的修改。

```shell
{"code":500,"message":"服务端异常","obj":null}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public RespBean ExceptionHandler(Exception e) {
        if (e instanceof GlobalException) {
            GlobalException ex = (GlobalException) e;
            return RespBean.error(ex.getRespBeanEnum());
        } else if (e instanceof BindException) {
            BindException be = (BindException) e;
            RespBean respBean = RespBean.error(RespBeanEnum.BIND_ERROR);
            respBean.setMessage("参数校验异常" + be.getBindingResult().getAllErrors().get(0).getDefaultMessage());
            return respBean;
        }
        System.out.println("Global     local    " + e.getLocalizedMessage());
        System.out.println("Global     msg    " + e.getMessage());
        System.out.println("Global     cause   " + e.getCause());
        System.out.println("Global     class    " + e.getClass());
        System.out.println("Global     stackTrace     " + e.getStackTrace().getClass());
        System.out.println("***********************************************");
        return RespBean.error(RespBeanEnum.ERROR);
    }

    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ExceptionHandler(NoHandlerFoundException.class)
    public RespBean handle(HttpServletRequest request, NoHandlerFoundException e) {
        return RespBean.error(RespBeanEnum.URL_ERROR,"没有【"+request.getMethod()+"】"+request.getRequestURI()+"方法可以访问");
    }
}
```

通过访问不存在的 url，则会出现如下的结果。

```shell
{"code":404,"message":"URL不存在","obj":"没有【GET】/login/方法可以访问"}
```

对于返回为 json，对移动端可能比较优，但对于网页端，则不是这样，我们同样可以返回渲染后的界面，如下。

```java
@ExceptionHandler(value = Exception.class)
public ModelAndView myErrorHandler(Exception ex) {
    ModelAndView modelAndView = new ModelAndView();
    modelAndView.setViewName("error");
    modelAndView.addObject("code", ex.getCode());
    modelAndView.addObject("msg", ex.getMsg());
    return modelAndView;
}
```

完结。
