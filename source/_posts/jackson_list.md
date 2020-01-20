---
title: jackson转换List格式的Json
categories:
- java
date: 2018-04-12 12:15:00
tags:
- json
---

**jackson处理数组类型json的时候出现问题**：  
**期望:** List<User>  
**实际:** List<LinkedHashMap>  
转List类型需要特殊处理一下  
```
POJO pojo = mapper.convertValue(singleObject, POJO.class);
//处理List，不能直接用ArrayList.class
List<POJO> pojos = mapper.convertValue(listOfObjects, new TypeReference<List<POJO>>() { });
```
<!--more-->
jackson转换：
```
/**
 * json bean转换，基于jackson
 * 无根节点,格式化日期
 * @author zhaoyd
 * @date 2017年5月17日
 */
public class JsonBeanConverUtil {
    private static final String formatStr = "yyyy-MM-dd HH:mm:ss";

    public static <T> String toJson(T bean) throws JsonProcessingException {
        if (bean == null) {
            return null;
        }
        ObjectMapper mapper = new ObjectMapper();
        SimpleDateFormat sdf = new SimpleDateFormat(formatStr);
        mapper.setDateFormat(sdf);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        String json = mapper.writeValueAsString(bean);
        return json;
    }

    public static <T> T fromJson(String json, Class<T> clazz) throws IOException {
        if (StringUtils.isBlank(json)) {
            return null;
        }
        ObjectMapper mapper = new ObjectMapper();
        SimpleDateFormat sdf = new SimpleDateFormat(formatStr);
        mapper.setDateFormat(sdf);
        T t = mapper.readValue(json, clazz);
        return t;
    }
    /**
     * 转list
     */
    public static <T extends List> T fromJsons(String json, TypeReference<T> jsonTypeReference) throws IOException {
        if (StringUtils.isBlank(json)) {
            return null;
        }
        ObjectMapper mapper = new ObjectMapper();
        SimpleDateFormat sdf = new SimpleDateFormat(formatStr);
        mapper.setDateFormat(sdf);
        T t = (T)mapper.readValue(json, jsonTypeReference);
        return t;
    }
}
```  
**一个需要注意的地方：**  
jackson生成对象是调用对象的无参构造方法，如果自定义了构造函数，也要显式定义一个无参构造方法，不然出错。