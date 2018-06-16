---
layout: post
title: 'MyBatis的SQL通配符'
date: 2017-01-13
author: Feihang Han
tags: MyBatis
---

# 案例描述

某平台V1.1版本开发中，开发人员发现常用搜索中，如果输入“%”、“\_”、“\”这三个字符，将搜出意料之外的结果。具体而言，搜“%”、“\_”能查到所有对象；搜“\”会无效。

本平台使用MySQL数据库，下面默认以MySQL为例子做描述。

# 案例分析和解决

## 步骤一

初步看来，这个问题是常见的SQL语句通配符问题。因为SQL语句中以下几个特殊字符有特殊含义。

| 通配符 | 描述 |
| :--- | :--- |
| % | 替代一个或多个字符 |
| \_ | 仅替代一个字符 |
| \ | 转义后面一个字符 |

在Mysql数据库中，如果想解决此问题有两个方案：

1）用ESCAPE关键字，将对应的特殊字符排除掉；

2）使用转义字符。

MyBatis框架提供的动态SQL功能允许用户根据输入条件自行拼接SQL语句，但是每一个小块的SQL语句是固定格式的，可使用替换符\#{}替换用户输入。

比如想根据用户名字、用户类型，搜索匹配的用户列表，其中用户名字是模糊匹配。

```xml
<select id="getPage" resultMap="userResult">
    select * from co_user
    <where>
        <if test="userName != null">
            user_name like CONCAT(#{userName},'%')
        </if>
        <if test="userType != null">
            and user_type = #{userType}
        </if>
    </where>
    order by create_time desc limit #{start},#{pageSize}
</select>
```

如果采用方案1），则程序需要判断过滤条件是否带有特殊字符，然后再改造SQL语句块，使之带ESCAPE或者不带。相比之下方案2）更为简单直观。

## 步骤二

当明确了方案2）后，第二步是确定什么时候去进行特殊字符的转义工作。MyBatis框架DAO层往往只个接口类，而没有任何实现（MyBatis框架会代理这些接口，并自动补全实现）。因此转义工作有两个时机可以做：

1）Service层，将参数传递给Dao层接口的时候

2）MyBatis框架进行SQL替换的时候。

从程序功能划分角度来说，Service层只负责业务，Dao层关注SQL以及与数据库交互，因此采用方案2）来解决这个问题更为妥当。

## 步骤三

MyBatis是支持定制化SQL、存储过程以及高级映射的优秀的持久层框架，避免了几乎所有的 JDBC代码和手动设置参数以及获取结果集，只需要简单配置映射即可使用。因此如果想要在DAO层解决上述问题，则需要深入了解MyBatis机制，并利用框架已有的扩展点，进行扩展，并确保不引入其他问题。

此时可以有下面两个扩展方案：

1）MyBatis拦截器

2）类型处理器

方案1）中的MyBatis拦截器是一个不错的方案，拦截参数并做出替换，本身的逻辑不复杂，但是如何将这个参数需要替换的标志传进去。如果要在Parameter对象里增加一个描述信息，描述这个参数需要进行处理，则会发现需要改动的地方很多，不是一个较为简单的方案，且对MyBatis默认的行为修改较多，容易引入其他问题。

方案2）中的类型处理器（TypeHandler）也是一个不错扩展点。无论是MyBatis在预处理语句（PreparedStatement）中设置一个参数时，还是从结果集中取出一个值时，都会用类型处理器将获取的值以合适的方式转换成JAVA 类型。

MyBatis默认为我们实现了许多TypeHandler , 当我们没有配置指定TypeHandler时，MyBatis会根据参数或者返回结果的不同，默认为我们选择合适的TypeHandler处理。

## 步骤四

1）实现一个专门用于转义的TypeHandler

```java
@MappedJdbcTypes(JdbcType.VARCHAR)
public class WildcardTypeHandler extends BaseTypeHandler<String> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
            throws SQLException {
        parameter = SqlUtil.repalceWildcard(parameter);
        ps.setString(i, parameter);
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return super.getResult(rs, columnName);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return super.getResult(rs, columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return super.getResult(cs, columnIndex);
    }

}
```

```java
public class SqlUtil {

    private static final String ESCAPE_CHAR = "\\";

    /**
     * 替换sql中的模糊特殊字符 _ % \
     * 
     * 该方法只在MyBatis + mysql下测试通过，如更换dao层框架或者数据库，则需重新测试
     * 
     * 请注意，下面这种替换方式，只支持preparedstatement，即对应#{},而无法支持${}
     * 
     * @author hanfeihang 2016年8月3日 上午10:48:50
     * @param str 转义前的字符串
     * @return 转义后的字符串
     */
    public static String repalceWildcard(String str) {
        String result = null;
        if (str != null) {
            result = str.replace(ESCAPE_CHAR, ESCAPE_CHAR + ESCAPE_CHAR);
            result = result.replace("%", ESCAPE_CHAR + "%");
            result = result.replace("_", ESCAPE_CHAR + "_");
        }
        return result;
    }

    public static void main(String[] args) {
        String str = "%3\\2%";
        System.out.println("result: " + repalceWildcard(str));
    }
}
```

2）对需要转义的参数配置TypeHandler

```xml
<if test="userName != null">
    user_name like CONCAT(#{userName, typeHandler=com.hikvision.building.common.MyBatis.WildcardTypeHandler},'%')
</if>
```

# 经验总结、预防措施和对规范的建议等

该文论述的问题在发生在真实项目中，也很容易被遗漏。即使开发人员发现了此问题，由于MyBatis框架代理了与数据库交互的逻辑，处理起来也较为棘手。

在深入了解MyBatis框架后，可以根据业务需求进行一些定制化，来改变其默认的行为。



