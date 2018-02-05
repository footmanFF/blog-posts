---
layout: post
title: MyBatis自定义枚举类型处理
date: 2017-05-17
tags: Java
---

# 直接上代码

需要自己去继承BaseTypeHandler来实现自己的类型转换。注意 @MappedTypes 是需要映射枚举类型，MappedJdbcTypes 是映射成的数据库字段类型。

<!-- more -->

```java
@MappedTypes({StatusEnum.class})
@MappedJdbcTypes({JdbcType.INTEGER, JdbcType.TINYINT})
public class EnumValueHandler<E extends Enum<E> & ValueEnum> extends BaseTypeHandler<E> {

    private final Class<E> type;
    private final Map<Integer, E> codeToEnum;

    public EnumValueHandler(Class<E> type) {
        if (type == null)
            throw new IllegalArgumentException("type不得为null");
        this.type = type;
        E[] enums = this.type.getEnumConstants();
        this.codeToEnum = new HashMap<>(enums.length);
        for (E e : enums) {
            this.codeToEnum.put(e.getCode(), e);
        }
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, parameter.getCode());
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int value = rs.getInt(columnName);
        if (rs.wasNull()) {
            return null;
        }
        return codeToEnum.get(value);
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int i = rs.getInt(columnIndex);
        if (rs.wasNull()) {
            return null;
        } else {
            return codeToEnum.get(i);
        }
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int i = cs.getInt(columnIndex);
        if (cs.wasNull()) {
            return null;
        } else {
            return codeToEnum.get(i);
        }
    }

}
```

将自定义个TypeHandler注册到MyBatis

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
   SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
   // ....
   sqlSessionFactoryBean.setTypeHandlersPackage("cn.com.weidai.rdc.dao.handler");
   return sqlSessionFactoryBean.getObject();
}
```

枚举的接口

```java
public interface ValueEnum {
    int getCode();
}

```

mapper.xml需要注意的地方，ResultMap配置的时候需要指明字段的jdbcType和javaType

```xml
<resultMap id="RuleCoreMap" type="RuleCore">
   <result property="id" column="id"/>
   <result property="ruleName" column="rule_name"/>
   <result property="ruleStatus" column="rule_status" jdbcType="INTEGER" javaType="xx.xxx.enums.StatusEnum" />
</resultMap>
```

查询和写入的时候，同样也要去指定jdbcType

```
insert into xxx values (... #{ruleStatus,jdbcType=INTEGER} ...)
select ... from xxx where xx = #{ruleStatus,jdbcType=INTEGER}
```

其中查询的时候 jdbcType 必须加上。insert也一样。

