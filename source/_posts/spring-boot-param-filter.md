---
title: springboot拦截请求的参数
date: 2018-07-11 09:13:39
tags:
---
前端请求的数据中，有一些数据前后空格、制表符等数据是不期望的，有一些含有非法参数，需要统一过滤这些内容。
<!-- more -->
# springboot拦截请求的参数
## 参数获取方式
**三种方式：**  
1. HttpServletRequest getParameter、getParameterValues方法获取参数
2. Controller方法自动封装的bean参数
3. Controller方法用@RequestBody接收的json格式数据 
   
对于1、2 重写HttpServletRequestWrapper并加入filter  
对于3 重写ObjectMapper在json反序列化时过滤  

## 重写HttpServletRequestWrapper
  
  ```java
  public class ParameterRequestWrapper extends HttpServletRequestWrapper {
    private Logger log = LoggerFactory.getLogger(this.getClass());

    private Map<String , String[]> params = new HashMap<>();

    public ParameterRequestWrapper(HttpServletRequest request) {
        // 将request交给父类，以便于调用对应方法的时候，将其输出，其实父亲类的实现方式和第一种new的方式类似
        super(request);
        //将参数表，赋予给当前的Map以便于持有request中的参数
        Map<String, String[]> requestMap=request.getParameterMap();
        log.trace("转化前参数："+ JSON.toJSONString(requestMap));
        this.params.putAll(requestMap);
        this.handleParameterValues();
        log.trace("转化后参数："+JSON.toJSONString(params));
    }
    /**
     * 过滤parameter
     */
    private void handleParameterValues(){
        Set<String> set =params.keySet();
        Iterator<String> it=set.iterator();
        while(it.hasNext()){
            String key= it.next();
            String[] values = params.get(key);
            for(int i=0; i<values.length; i++) {
            	//过滤参数方法
                values[i] = ParameterUtil.handleParam(values[i]);
            }
            params.put(key, values);
        }
    }

    /**
     * 重写getParameter 参数从当前类中的map获取
     */
    @Override
    public String getParameter(String name) {
        String[]values = params.get(name);
        if(values == null || values.length == 0) {
            return null;
        }
        return values[0];
    }
    /**
     * 重写getParameterValues
     */
    @Override
    public String[] getParameterValues(String name) {//同上
        return params.get(name);
    }

}
  ```
  
  ## filter配置
  filter类
  ```java
  public class ParamsFilter implements Filter {
    private Logger log = LoggerFactory.getLogger(this.getClass());
    @Override
    public void doFilter(ServletRequest arg0, ServletResponse arg1,
                         FilterChain arg2) throws IOException, ServletException {
        ParameterRequestWrapper parmsRequest = new ParameterRequestWrapper(
                (HttpServletRequest) arg0);
        arg2.doFilter(parmsRequest, arg1);
    }

    @Override
    public void init(FilterConfig arg0) throws ServletException {
        log.info("ParamsFilter init");
    }

    @Override
    public void destroy() {
        log.info("ParamsFilter destroy");
    }

}
  ```
  注册filter
  ```java
  @Bean
    public FilterRegistrationBean parmsFilterRegistration() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setDispatcherTypes(DispatcherType.REQUEST);
        registration.setFilter(new ParamsFilter());
        registration.addUrlPatterns("/*");
        registration.setName("paramsFilter");
        registration.setOrder(Integer.MAX_VALUE-1);
        return registration;
    }
  ```
  这样可以处理前两种方式获取参数的情况  
  ## json格式参数
  json格式SrpingMVC通过@RequestBody注解接收，需要配置全局json转换  
  ```java
  @Bean
    @Primary
    public ObjectMapper jsonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        //解析器
        ObjectMapper objectMapper = builder.createXmlMapper(false).build();
        objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        //注册xss解析器
        SimpleModule xssModule = new SimpleModule("ParamStringJsonSerializer");
        //json string反序列化成bean
        xssModule.addDeserializer(String.class, new JsonDeserializer<String>() {
            @Override
            public String deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
                String value = jsonParser.getText();
                //过滤参数方法
                value = ParameterUtil.handleParam(value);
                return value;
            }
        });
        objectMapper.registerModule(xssModule);
        return objectMapper;
    }
  ```
  