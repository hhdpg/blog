## 封装自动构建MongoDB查询条件的方法

#### 在对mongoDb进行查询的时候，繁琐的查询条件写一堆，还得一个个去判空，实在是麻烦， 本着是偷懒的想法，所以写了一套封装方法，可以自动去构建查询条件，目前只加了一个常用的查询方法，读者想加其他方法可以自己加下。详情如下：

#### 1. 先创建MongoDbField和一些DTO类

```java
package com.sf.retail.common.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @Auther xiechuandong
 * @Date 2022/11/8
 * @Description:
 */
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MongoDbField {

    Model model() default Model.DEFAULT;

    //字段属于区间值时，指定该字段，且model只能是两个值且不相同
    String appointField() default "";

    enum Model {
        //默认值
        DEFAULT,
        IS,
        IN,
        NE,
        NIN,
        REGEX,
        GT,
        LT,
        GTE,
        LTE
    }
}

```

MongodbFieldDTO类

```java
package com.sf.retail.common.dto;

import com.sf.retail.common.annotation.MongoDbField;
import lombok.Data;

/**
 * @Auther xiechuandong
 * @Date 2022/11/8
 * @Description:
 */
@Data
public class MongodbFieldDTO {

    private String key;
    private Object value;
    private Class<?> fieldType;
    private MongoDbField.Model model;
    private String appointField;
}

```

AppointFieldTableDTO类

```java
package com.sf.retail.common.dto;

import com.sf.retail.common.annotation.MongoDbField;
import lombok.Data;

/**
 * @Auther xiechuandong
 * @Date 2022/11/8
 * @Description:
 */

@Data
public class AppointFieldTableDTO {

    private String appointField;
    private MongoDbField.Model startModel;
    private Object startValue;
    private MongoDbField.Model endModel;
    private Object endValue;
}

```

#### 2. 核心方法MongoBbQuerySqlBuildService类

```java
package com.sf.retail.order.repository;

import com.sf.retail.common.annotation.MongoDbField;
import com.sf.retail.common.dto.AppointFieldTableDTO;
import com.sf.retail.common.dto.MongodbFieldDTO;
import com.sf.retail.common.enums.ErrorCode;
import com.sf.retail.common.exception.BizException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.stereotype.Service;

import java.beans.BeanInfo;
import java.beans.IntrospectionException;
import java.beans.Introspector;
import java.beans.PropertyDescriptor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.*;
import java.util.regex.Pattern;

/**
 * @Auther xiechuandong
 * @Date 2022/11/8
 * @Description: 封装MongoDB自动构建查询criteria方法
 */
@Slf4j
@Service
public class MongoBbQuerySqlBuildService {

    /**
     * ------------------------------------使用方法 --------------------------------------------
     * 新建一个DTO,如果不是通用的，也可以不新建。字段和查询类request一模一样，并使用BeanUtils.copyProperties方法将复制过来
     * 然后在需要查询的类上添加@MongoDbField注解，出list类型外，默认转化为criteria语句就是is，list是in，也可以在注解上添加model属性值
     * 填入要执行的操作，如ne，nin等
     * 对于属于区间值的字段，在@MongoDbField注解上添加appointField属性值，指定要查询的字段名，并添加model，如gt，lt等
     * 然后调用buildQueryCriteria方法即可
     * ----------------------------------------------------------------------------------------
     */


    public Criteria buildQueryCriteria(Object obj) {
        if (obj == null) {
            throw new RuntimeException("model is null");
        }
        Map<String, Object> beanMap = null;
        try {
            beanMap = getBeanToMap(obj);
        } catch (Exception e) {
            log.error("parse model is failed");
        }
        if (beanMap.isEmpty()) {
            throw new RuntimeException("model param all is null");
        }

        MongoDbField mongoDbField;
        Criteria criteria = new Criteria();
        Map<String, AppointFieldTableDTO> appointFieldMap = new HashMap<>();
        for (Field declaredField : obj.getClass().getDeclaredFields()) {
            if (declaredField.getAnnotation(MongoDbField.class) != null) {
                //如果该字段没值，直接跳过
                if (!beanMap.containsKey(declaredField.getName())) continue;

                mongoDbField = declaredField.getAnnotation(MongoDbField.class);
                MongodbFieldDTO mongodbFieldDTO = new MongodbFieldDTO();
                //对大于小于的字段指定属性值，错误的不进行拼接
                String appointField = mongoDbField.appointField();
                //若存在指定属性值，则已该指定属性值作为key，不存在则已字段名作为key
                if (!Objects.equals(appointField, "")) {
                    mongodbFieldDTO.setKey(appointField);
                } else {
                    mongodbFieldDTO.setKey(declaredField.getName());
                }
                mongodbFieldDTO.setAppointField(appointField);
                mongodbFieldDTO.setValue(beanMap.get(declaredField.getName()));
                mongodbFieldDTO.setModel(mongoDbField.model());
                mongodbFieldDTO.setFieldType(declaredField.getType());

                //拼接criteria参数
                splicingCriteria(criteria, mongodbFieldDTO, appointFieldMap);
            }
        }
        //对过滤下来的区间值字段进行拼接
        if (!appointFieldMap.isEmpty()) {
            appointFieldMap.forEach((k, v) -> {
                if (Objects.nonNull(v)) {
                    buildGTE(k, criteria, v);
                }
            });
        }
        log.info("buildQueryCriteria result: {}", criteria.getCriteriaObject());
        return criteria;
    }

    /**
     * 获取的属性名和属性值
     * @param obj
     * @return
     * @throws IntrospectionException
     * @throws InvocationTargetException
     * @throws IllegalAccessException
     */
    private Map<String, Object> getBeanToMap(Object obj) throws IntrospectionException, InvocationTargetException, IllegalAccessException{
        Map<String, Object> map = new HashMap<>();
        BeanInfo beanInfo = Introspector.getBeanInfo(obj.getClass());
        PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();
        for (PropertyDescriptor propertyDescriptor : propertyDescriptors) {
            String key = propertyDescriptor.getName();
            if (key.compareToIgnoreCase("class") == 0) continue;
            Method getter = propertyDescriptor.getReadMethod();
            Object value = getter != null ? getter.invoke(obj) : null;
            if (value != null) {
                map.put(key, value);
            }
        }
        return map;
    }

    private void splicingCriteria(Criteria criteria, MongodbFieldDTO mongodbFieldDTO, Map<String, AppointFieldTableDTO> appointFieldMap) {
        String key = mongodbFieldDTO.getKey();
        Object value = mongodbFieldDTO.getValue();
        Class<?> fieldType = mongodbFieldDTO.getFieldType();
        MongoDbField.Model model = mongodbFieldDTO.getModel();
        String appointField = mongodbFieldDTO.getAppointField();
        //不填值，除了list类型默认是in，其余都是默认is
        if (Objects.equals(model, MongoDbField.Model.DEFAULT)) {
            if (fieldType != List.class) {
                criteria.and(key).is(value);
            } else {
                criteria.and(key).in(value);
            }
        } else {
            //指定属性值存在的大于等于单独进行拼接
            if (!Objects.equals(appointField, "")) {
                statisticAppointFieldCriteria(value, model, appointField, appointFieldMap);
            } else {
                splicingAppointModelCriteria(criteria, key, value, model);
            }
        }
    }

    private void statisticAppointFieldCriteria(Object value, MongoDbField.Model model, String appointField, Map<String, AppointFieldTableDTO> appointFieldMap) {

        boolean gtOrGte = Objects.equals(model, MongoDbField.Model.GT) || Objects.equals(model, MongoDbField.Model.GTE);
        boolean ltOrLte = Objects.equals(model, MongoDbField.Model.LT) || Objects.equals(model, MongoDbField.Model.LTE);
        //校验appointFieldMap是否已经存在该指定值
        if (appointFieldMap.containsKey(appointField)) {
            AppointFieldTableDTO tableDTO = appointFieldMap.get(appointField);

            if (tableDTO != null) {
                //当map存在该指定属性值时，新的model如果不能和map中的model对应则认为输入错误，并置null
                if (Objects.nonNull(tableDTO.getStartModel())  && Objects.isNull(tableDTO.getEndModel()) && (ltOrLte)) {
                    tableDTO.setEndModel(model);
                    tableDTO.setEndValue(value);
                } else if (Objects.nonNull(tableDTO.getEndModel()) && Objects.isNull(tableDTO.getStartModel()) && (gtOrGte)) {
                    tableDTO.setStartModel(model);
                    tableDTO.setStartValue(value);
                } else {
                    log.info("appointField input is failed");
                    appointFieldMap.put(appointField, null);
                }
            }
        } else {
            AppointFieldTableDTO tableDTO = new AppointFieldTableDTO();
            tableDTO.setAppointField(appointField);
            if (gtOrGte) {
                tableDTO.setStartModel(model);
                tableDTO.setStartValue(value);
                appointFieldMap.put(appointField, tableDTO);
            } else if (ltOrLte) {
                tableDTO.setEndValue(model);
                tableDTO.setEndValue(value);
                appointFieldMap.put(appointField, tableDTO);
            }
        }
    }

    private void splicingAppointModelCriteria(Criteria criteria, String key, Object value, MongoDbField.Model model) {
        switch (model) {
            case IS:
                criteria.and(key).is(value);
            case NE:
                criteria.and(key).ne(value);
            case IN:
                criteria.and(key).in(value);
            case NIN:
                criteria.and(key).nin(value);
            case REGEX:
                criteria.and(key).regex((Pattern) value);
            default:
        }
    }


    private void buildGTE(String key, Criteria criteria, AppointFieldTableDTO appointFieldTableDTO) {
        MongoDbField.Model startModel = appointFieldTableDTO.getStartModel();
        Object startValue = appointFieldTableDTO.getStartValue();
        MongoDbField.Model endModel = appointFieldTableDTO.getEndModel();
        Object endValue = appointFieldTableDTO.getEndValue();

        if (Objects.equals(startModel, MongoDbField.Model.GT) && Objects.equals(endModel, MongoDbField.Model.LT)) {
            criteria.and(key).gt(startValue).lt(endValue);
        } else if (Objects.equals(startModel, MongoDbField.Model.GT) && Objects.equals(endModel, MongoDbField.Model.LTE)) {
            criteria.and(key).gt(startValue).lte(endValue);
        } else if (Objects.equals(startValue, MongoDbField.Model.GTE) && Objects.equals(endModel, MongoDbField.Model.LT)) {
            criteria.and(key).gte(startValue).lt(endValue);
        } else if (Objects.equals(startModel, MongoDbField.Model.GTE) && Objects.equals(endModel, MongoDbField.Model.LTE)) {
            criteria.and(key).gte(startValue).lte(endValue);
        }
    }
}

```

#### 3. 使用方法

① 先创建一个需要查询的字段的一个DTO类，在查询字段上面加上想对应的注解

```java
package com.sf.retail.order.dto;

import com.sf.retail.common.annotation.MongoDbField;
import lombok.Data;

/**
 * @Auther xiechuandong
 * @Date 2022/11/9
 * @Description:
 */
@Data
public class MongoQueryDTO {

    @MongoDbField
    private String orderSn;

    @MongoDbField(appointField = "orgId", model = MongoDbField.Model.GT)
    private Long oneOrgIdStart;

    @MongoDbField(appointField = "orgId", model = MongoDbField.Model.LT)
    private Long oneOrgIdEnd;

//    @MongoDbField
    private Long orgId;
}

```

② 添加测试接口

```java
@Autowired
    private MongoBbQuerySqlBuildService mongoBbQuerySqlBuildService;

    @Test
    public void splicingTest() {
        MongoQueryDTO mongoQueryDTO = new MongoQueryDTO();
        mongoQueryDTO.setOrderSn("SO1587743601739173889");
        mongoQueryDTO.setOneOrgIdStart(14444l);
        mongoQueryDTO.setOneOrgIdEnd(172222l);
        mongoQueryDTO.setOrgId(154254l);
        Criteria criteria = mongoBbQuerySqlBuildService.buildQueryCriteria(mongoQueryDTO);
        ShopOrderDoc shopOrderDoc = shopOrderDao.findOne(Query.query(criteria));
        log.info("test:{}", JSON.toJSONString(shopOrderDoc));
    }
```

