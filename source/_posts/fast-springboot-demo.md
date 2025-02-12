---
title: 单体springboot小应用基础配置项
date: 2025-02-11 10:00:00
tags: [java, Spring, SpringBoot]
---

# 快速搭建 springboot 单体应用的步骤

## 第一步：创建一个简单的项目

1. 使用 idea 创建，并选择相关的依赖，这里只引入了最简单的部分

   <img src="/images/image-20250211094135244.png" alt="image-20250211094135244" style="zoom: 50%;" />

2. 编写一个简单的接口，验证项目是否创建成功

   ```java
   @RestController
   @RequestMapping("/")
   public class HomeController {

       @GetMapping
       public String helloWorld() {
           return "Hello World!";
       }
   }
   ```

## 第二步：创建拦截器和过滤器做一些想要的动作

这里以 token 校验和请求日志打印为例

```java
//过滤器
public class BaseFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
      	//需要将默认的request对象转为可重复读取的对象
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(httpServletRequest);
        filterChain.doFilter(wrappedRequest, servletResponse);
    }
}
//拦截器
public class BaseInterceptor implements HandlerInterceptor {

    Logger log = LoggerFactory.getLogger(BaseInterceptor.class);
    private static final String REQ_BEGIN = "req_begin";
    private static final String TOKEN_KEY = "Authorization";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        log.info("请求开始：{}", request.getRequestURI());
        request.setAttribute(REQ_BEGIN, System.currentTimeMillis());
        //自定义token校验，存入上下文中
        String token = request.getHeader(TOKEN_KEY);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
        // 在请求处理完成后读取请求体
        if (request instanceof ContentCachingRequestWrapper) {
            ContentCachingRequestWrapper requestWrapper = (ContentCachingRequestWrapper) request;
            String requestBody = new String(requestWrapper.getContentAsByteArray(),
                    requestWrapper.getCharacterEncoding());
            log.info("请求体内容: {}", requestBody);
        }

        // 计算请求耗时
        Long reqBegin = (Long) request.getAttribute(REQ_BEGIN);
        long reqEnd = System.currentTimeMillis();
        log.info("请求结束，耗时：{} ms", reqEnd - reqBegin);
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
//配置到spring容器中
@Configuration
public class SpringWebConfig implements WebMvcConfigurer {

  	//注册过滤器
    @Bean
    public FilterRegistrationBean<BaseFilter> cachingRequestBodyFilter() {
        FilterRegistrationBean<BaseFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new BaseFilter());
        registrationBean.addUrlPatterns("/*"); // 根据需要设置过滤的 URL 模式
        return registrationBean;
    }
		//注册拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new BaseInterceptor());
    }
}

```

## 第三步：统一返回格式和异常处理

```java
//统一返回数据结构
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Result<T> {
    private String message;
    private Integer code;
    private T data;

    public static <T> Result<T> ok(T data) {
        return new Result<>("success", 0, data);
    }

    public static <T> Result<T> failed(String message) {
        return new Result<>(message, 1001, null);
    }

    public Result(String message, Integer code, T data) {
        this.message = message;
        this.code = code;
        this.data = data;
    }
}
//通过result返回数据，以及统一异常
@ControllerAdvice
public class GlobalResponse implements ResponseBodyAdvice<Object> {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // 处理所有的结果
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
                                  ServerHttpResponse response) {
        if (body instanceof String) {
            try {
                return objectMapper.writeValueAsString(Result.ok(body));
            } catch (Exception e) {
                return Result.failed("序列化异常");
            }
        }
        // 返回结果使用Result进行包装
        if (body instanceof Result) {
            return body;
        }
        return Result.ok(body);
    }

    @ExceptionHandler(RuntimeException.class)
    @ResponseBody
    public final Result<String> handleUnknownExceptions(RuntimeException e) {
        // 异常报错使用Result进行包装
        return Result.failed(e.getMessage());
    }
}

```

## 附：测试代码

```java
@Slf4j
@RestController
@RequestMapping
public class HomeController {
    @GetMapping("/str")
    public String helloWorld() {
        return "Hello World!";
    }

    @PostMapping("/req")
    public HelloReq postHelloWorld(@RequestBody HelloReq helloReq) {
        log.info("收到请求：{}", helloReq.toString());
        HelloReq changeGender = new HelloReq();
        changeGender.setUsername(helloReq.getUsername());
        changeGender.setUserGender("女");
        changeGender.setUserAge(helloReq.getUserAge());
        return changeGender;
    }

    @PostMapping("/ex")
    public void deleteHelloWorld() {
        throw new RuntimeException("模拟异常");
    }

    @Data
    public static class HelloReq {
        private String username;
        private String userGender;
        private Integer userAge;
    }
}
```
