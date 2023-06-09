# Select

> 本节内容来自于
>
> [mybatis – MyBatis 3 | XML 映射器](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#select)
>
> 本文代码
>
> [select](https://github.com/dreamlike-ocean/backend_qingyou_code/tree/master/mybatis/src/main/java/top/dreamlike/qingyou/select)

## 前言

大部分时候我们都是与查询打交道，从各种角度来查询，如何对复杂查询做映射是我们最经常遇到的事情。

## 使用

### 简单的查询

还是回到我们Get-start部分的那个查询

```xml
<mapper namespace="top.dreamlike.qingyou.select.UserMapper">
    <select id="selectById" resultType="top.dreamlike.qingyou.entity.LoginUser">
        select * from login_user where user_id = #{userId}
    </select>
</mapper>
```

```java
public interface UserMapper {
    LoginUser selectById(@Param("userId") Integer userId);
}
```

首先xml通过namespace绑定到一个接口上面，然后在sql语句的标签（这里是select）的id绑定到对应的接口上面，然后通过resultType属性来指定对应的返回值类型（其实不指定也行，能自动解析出来）

注意这个占位符`#{userId}`，这个占位符的名称就是我们在接口方法上用@Param注解的值

note：为了防止有些情况忘记把函数参数名编译进去导致的绑定失败，还是推荐手动标注一下

这个占位符最后会被转换为预编译语句（prepared statement）中的`?`。

~~现在你已经弄明白1+1了，让我们进入微积分的学习~~

现在让我们进一步看看select标签的其他属性

| 属性             | 描述                                                         |
| :--------------- | :----------------------------------------------------------- |
| `id`             | 在命名空间中唯一的标识符，可以被用来引用这条语句。           |
| `parameterType`  | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。 |
| ~~parameterMap~~ | 用于引用外部 parameterMap 的属性，目前已被**废弃**。请使用行内参数映射和 parameterType 属性。 |
| `resultType`     | 期望从这条语句中返回结果的类全限定名或别名。 注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 resultType 和 resultMap 之间只能同时使用一个。 |
| `resultMap`      | 对外部 resultMap 的命名引用。结果映射是 MyBatis 最强大的特性，如果你对其理解透彻，许多复杂的映射问题都能迎刃而解。 resultType 和 resultMap 之间只能同时使用一个。 |
| `flushCache`     | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。 |
| `useCache`       | 将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。 |
| `timeout`        | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。 |
| `fetchSize`      | 这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。 默认值为未设置（unset）（依赖驱动）。 |
| `statementType`  | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| `resultSetType`  | FORWARD_ONLY，SCROLL_SENSITIVE, SCROLL_INSENSITIVE 或 DEFAULT（等价于 unset） 中的一个，默认值为 unset （依赖数据库驱动）。 |
| `databaseId`     | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。 |
| `resultOrdered`  | 这个设置仅针对嵌套结果 select 语句：如果为 true，则假设结果集以正确顺序（排序后）执行映射，当返回新的主结果行时，将不再发生对以前结果行的引用。 这样可以减少内存消耗。默认值：`false`。 |
| `resultSets`     | 这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称之间以逗号分隔。 |

### resultType

期望从这条语句中返回结果的类全限定名或别名。 注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 **resultType 和 resultMap 之间只能同时使用一个**。

这个没什么好讲的，举个例子，比如说我不要映射为java bean而是hashmap呢？就需要这种形式了

```xml
<!--   List<HashMap<String,Object>> selectByIdtToMap(@Param("userId") Integer userId);-->
    <select id="selectByIdtToMap" resultType="java.util.HashMap">
        select * from login_user where user_id = #{userId}
    </select>
```

### resultMap

这个就是本节的重中之重，如果你之前跑过Get-start部分的代码会发现LoginUser的userId字段查出来是空的,因为默认情况下mybatis会假定你的数据库列名和Java Bean字段名是一致的，因此没有映射成功。为了解决这种问题和其他更复杂的结果集映射问题，我们需要它。

ResultMap 的设计思想是，对简单的语句做到零配置，对于复杂一点的语句，只需要描述语句之间的关系就行了。

#### 字段名不同问题

首先先创建resultMap标签

id就是这个映射的唯一识别码就可以了，然后type为要映射的类的全限定名或者别名，autoMapping建议为true这样可以那些我们没有配置的就使用自动配置

读者可以试试改为false的效果

```xml
<resultMap id="UserResultMap" type="top.dreamlike.qingyou.entity.LoginUser" autoMapping="true">
        <id property="userId" column="user_id"></id>
</resultMap>
<!--LoginUser selectByIdUseResultMap(@Param("userId") Integer userId); -->
    <select id="selectByIdUseResultMap" resultMap="UserResultMap">
        select *
        from login_user
        where user_id = #{userId}
    </select>
```

然后指定select标签的resultMap为之前我们给这个映射起的名字即可

#### 一对多问题之类的复杂映射

思考这样一个Java bean

```java
public class UserScoreRecord {
    //省略setter getter
    private Integer userId;
    private List<ScoreRecord> scoreRecords;

    public UserScoreRecord(Integer userId) {
        this.userId = userId;
    }
}
public class ScoreRecord {
    private Integer recordId;
    private Integer userId;
    private Integer count;
    private Long timestamp;
    private Integer type;
}
```

它代表了一个用户与其关联的分数记录集合，这个sql相信大家都会写

我们先一步一来，先写ScoreRecord的映射

```xml
<resultMap id="ScoreRecordResultMap" type="top.dreamlike.qingyou.entity.ScoreRecord" autoMapping="true">
    <id property="recordId" column="record_id"></id>
    <result property="userId" column="user_id"></result>
</resultMap>
```

再来写这个复合类的映射

```xml
 <resultMap id="UserScoreRecordResultMap" type="top.dreamlike.qingyou.select.UserScoreRecord" autoMapping="true">
     <constructor>
         <idArg column="user_id" javaType="Integer" ></idArg>
     </constructor>
     <collection property="scoreRecords" resultMap="ScoreRecordResultMap"></collection>
 </resultMap>
```

然后我们就可以利用这个映射了

```xml
<!-- UserScoreRecord selectUserScoreRecordById(@Param("userId") Integer userId); --> 
<select id="selectUserScoreRecordById" resultMap="UserScoreRecordResultMap">
    select user_id,sr.* from login_user
    join score_record sr using(user_id)
    where user_id = #{userId}
</select>
```

有了这个例子 让我们再来看看resultMap的子标签都是做什么的

- `constructor` - 用于在实例化类时，注入结果到构造方法中,用于指定实例化时使用哪个构造器，默认为空构造器
  - `idArg` - ID 参数；标记出作为 ID 的结果可以帮助提高整体性能
  - `arg` - 将被注入到构造方法的一个普通结果
- `result` – 注入到字段或 JavaBean 属性的普通结果
- `id` – 一个 ID 结果；标记出作为 ID 的结果可以帮助提高整体性能
- `association`- 一个复杂类型的关联；许多结果将包装成这种类型
  - 嵌套结果映射 – 关联可以是 `resultMap` 元素，或是对其它结果映射的引用
- `collection`– 一个复杂类型的集合
  - 嵌套结果映射 – 集合可以是 `resultMap` 元素，或是对其它结果映射的引用

#### id和result

```xml
<id property="id" column="post_id"/>
<result property="subject" column="post_subject"/>
```

这些元素是结果映射的基础。*id* 和 *result* 元素都将一个列的值映射到一个简单数据类型（String, int, double, Date 等）的属性或字段。

这两者之间的唯一不同是，*id* 元素对应的属性会被标记为对象的标识符，在比较对象实例时使用。 这样可以提高整体的性能，尤其是进行缓存和嵌套结果映射（也就是连接映射）的时候。

两个元素都有一些属性：

| 属性          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| `property`    | 映射到列结果的字段或属性。如果 JavaBean 有这个名字的属性（property），会先使用该属性。否则 MyBatis 将会寻找给定名称的字段（field）。 无论是哪一种情形，你都可以使用常见的点式分隔形式进行复杂属性导航。 比如，你可以这样映射一些简单的东西：“username”，或者映射到一些复杂的东西上：“address.street.number”。 |
| `column`      | 数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 |
| `javaType`    | 一个 Java 类的全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以**推断类型**。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。 |
| `jdbcType`    | JDBC 类型，所支持的 JDBC 类型参见这个表格之后的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可以为空值的列指定这个类型。 |
| `typeHandler` | 我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的全限定名，或者是类型别名。 |

#### 支持的 JDBC 类型

为了以后可能的使用场景，MyBatis 通过内置的 jdbcType 枚举类型支持下面的 JDBC 类型。

| `BIT`      | `FLOAT`   | `CHAR`        | `TIMESTAMP`     | `OTHER`   | `UNDEFINED` |
| ---------- | --------- | ------------- | --------------- | --------- | ----------- |
| `TINYINT`  | `REAL`    | `VARCHAR`     | `BINARY`        | `BLOB`    | `NVARCHAR`  |
| `SMALLINT` | `DOUBLE`  | `LONGVARCHAR` | `VARBINARY`     | `CLOB`    | `NCHAR`     |
| `INTEGER`  | `NUMERIC` | `DATE`        | `LONGVARBINARY` | `BOOLEAN` | `NCLOB`     |
| `BIGINT`   | `DECIMAL` | `TIME`        | `NULL`          | `CURSOR`  |             |

#### constructor

之前我们展示了constructor标签，现在让我们单独拿出看看

假设我们有这样一个类的构造器

```java
public User(@Param("userId") Integer userId,@Param("username") String username,@Param("password") String password) {
        this.userId = userId;
        this.username = username;
        this.password = password;
        System.out.println("我被mybatis调用了");
}
```

当你在处理一个带有多个形参的构造方法时，很容易搞乱 arg 元素的顺序。 从 3.4.3 开始，可以在指定参数名称的前提下，以任意顺序编写 arg 元素。 为了通过名称来引用构造方法参数，你可以添加 `@Param` 注解，或者使用 '-parameters' 编译选项并启用 `useActualParamName` 选项（默认开启）来编译项目。下面是一个等价的例子，尽管函数签名中第二和第三个形参的顺序与 constructor 元素中参数声明的顺序不匹配。

```xml
<resultMap id="UserMap" type="top.dreamlike.qingyou.select.User">
    <constructor>
        <idArg column="user_id" name="userId"/>
        <arg column="password" name="password"/>
        <arg column="username" name="username"/>
    </constructor>
</resultMap>
```

#### association

这种属于一对一中常用，比如说我们之前的例子，一个积分记录对应一个用户

```java
public class ScoreRecordUserInfo {
    private Integer recordId;
    private Long timestamp;
    private LoginUser loginUser;
}
```

对应映射就很简单

```xml
<resultMap id="ScoreRecordUserInfoMap" type="top.dreamlike.qingyou.select.ScoreRecordUserInfo" autoMapping="true">
    <id column="record_id" property="recordId"></id>
    <association property="loginUser" resultMap="UserResultMap"></association>
</resultMap>

    <select id="selectScoreUserInfoById" resultMap="ScoreRecordUserInfoMap">
        select record_id,timestamp,u.* from score_record
        join login_user u using(user_id) limit 1
    </select>
```

#### automapping

正如你在前面一节看到的，在简单的场景下，MyBatis 可以为你自动映射查询结果。但如果遇到复杂的场景，你需要构建一个结果映射。 但是在本节中，你将看到，你可以混合使用这两种策略。让我们深入了解一下自动映射是怎样工作的。

当自动映射查询结果时，MyBatis 会获取结果中返回的列名并在 Java 类中查找相同名字的属性（忽略大小写）。 这意味着如果发现了 *ID* 列和 *id* 属性，MyBatis 会将列 *ID* 的值赋给 *id* 属性。

通常数据库列使用大写字母组成的单词命名，单词间用下划线分隔；而 Java 属性一般遵循驼峰命名法约定。为了在这两种命名方式之间启用自动映射，需要将 `mapUnderscoreToCamelCase` 设置为 true。

即在config的文件中添加一个属性

```xml
<configuration>
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="value"/>
    </settings> 
    <!--    省略其他-->
</configuration>
```

甚至在提供了结果映射后，自动映射也能工作。在这种情况下，对于每一个结果映射，在 ResultSet 出现的列，如果没有设置手动映射，将被自动映射。在自动映射处理完毕后，再处理手动映射。 在下面的例子中，*id* 和 *userName* 列将被自动映射，*hashed_password* 列将根据配置进行映射。

```xml
<select id="selectUsers" resultMap="userResultMap">
  select
    user_id             as "id",
    user_name           as "userName",
    hashed_password
  from some_table
  where id = #{id}
</select>
<resultMap id="userResultMap" type="User">
  <result property="password" column="hashed_password"/>
</resultMap>
```

有三种自动映射等级：

- `NONE` - 禁用自动映射。仅对手动映射的属性进行映射。
- `PARTIAL` - 对除在内部定义了嵌套结果映射（也就是连接的属性）以外的属性进行映射
- `FULL` - 自动映射所有属性。

## 其他一些基础配置

以下的所有配置都得写在一个自己定义的配置xml里面，xml开头得带有如下

```xml-dtd
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
```

这里先列举一些配置标签，之后会详细讲解

* 根标签：

  * <configuration>：核心根标签，所有的配置都得在这个标签下面

* 引入连接配置文件：

  * <properties>： 引入数据库连接配置文件标签

    * resource：属性，指定配置文件名

    ```xml-dtd
    <properties resource="jdbc.properties"/>
    ```

* 调整设置

  * <settings>：可以改变 Mybatis 运行时行为

* 起别名：

  * <typeAliases>：为全类名起别名的父标签

    * <typeAlias>：为全类名起别名的子标签
      * type：指定全类名      
      * alias：指定别名

    * <package>：为指定包下所有类起别名的子标签，别名就是类名，首字母小写

    ```xml-dtd
    <!--起别名-->
    <typeAliases>
    	<typeAlias type="com.example.springmvclearn.entity.User" alias="User"/><!--注册单个-->
    	<package name="com.example.springmvclearn.entity"/><!--注册所有-->
    		<!--二选一-->
    </typeAliase>
    ```

  * 自带别名：

    | 别名    | 数据类型          |
    | ------- | ----------------- |
    | string  | java.lang.String  |
    | long    | java.lang.Lang    |
    | int     | java.lang.Integer |
    | double  | java.lang.Double  |
    | boolean | java.lang.Boolean |
    | ....    | ......            |



* 配置环境，可以配置多个标签
  * <environments>：配置数据库环境标签，default 属性指定哪个 environment
  * <environment>：配置数据库环境子标签，id 属性是唯一标识，与 default 对应
  * <transactionManager>：事务管理标签，type 属性默认 JDBC 事务
  * <dataSoure>：数据源标签
    * type 属性：POOLED 使用连接池（MyBatis 内置），UNPOOLED 不使用连接池
  * <property>：数据库连接信息标签。
    * name 属性取值：driver，url，username，password
    * value 属性取值：与 name 对应
* 引入映射配置文件

  * <mappers>：引入映射配置文件标签
  * <mapper>：引入映射配置文件子标签
    * resource：属性指定映射配置文件的名称
    * url：引用网路路径或者磁盘路径下的 sql 映射文件
    * class：指定映射配置类
  * <package>：批量注册

#### 类型处理器（typeHandlers）

MyBatis 在设置预处理语句（PreparedStatement）中的参数或从结果集中取出一个值时， 都会用类型处理器将获取到的值以合适的方式转换成 Java 类型。下表描述了一些默认的类型处理器。

这里摘自官网的一些举例：

| 类型处理器                   | Java 类型                      | JDBC 类型                            |
| :--------------------------- | :----------------------------- | :----------------------------------- |
| `BooleanTypeHandler`         | `java.lang.Boolean`, `boolean` | 数据库兼容的 `BOOLEAN`               |
| `ByteTypeHandler`            | `java.lang.Byte`, `byte`       | 数据库兼容的 `NUMERIC` 或 `BYTE`     |
| `ShortTypeHandler`           | `java.lang.Short`, `short`     | 数据库兼容的 `NUMERIC` 或 `SMALLINT` |
| `IntegerTypeHandler`         | `java.lang.Integer`, `int`     | 数据库兼容的 `NUMERIC` 或 `INTEGER`  |
| `LongTypeHandler`            | `java.lang.Long`, `long`       | 数据库兼容的 `NUMERIC` 或 `BIGINT`   |
| `FloatTypeHandler`           | `java.lang.Float`, `float`     | 数据库兼容的 `NUMERIC` 或 `FLOAT`    |
| `DoubleTypeHandler`          | `java.lang.Double`, `double`   | 数据库兼容的 `NUMERIC` 或 `DOUBLE`   |
| `BigDecimalTypeHandler`      | `java.math.BigDecimal`         | 数据库兼容的 `NUMERIC` 或 `DECIMAL`  |
| `StringTypeHandler`          | `java.lang.String`             | `CHAR`, `VARCHAR`                    |
| `ClobReaderTypeHandler`      | `java.io.Reader`               | -                                    |
| `ClobTypeHandler`            | `java.lang.String`             | `CLOB`, `LONGVARCHAR`                |
| `NStringTypeHandler`         | `java.lang.String`             | `NVARCHAR`, `NCHAR`                  |
| `NClobTypeHandler`           | `java.lang.String`             | `NCLOB`                              |
| `BlobInputStreamTypeHandler` | `java.io.InputStream`          | -                                    |
| `ByteArrayTypeHandler`       | `byte[]`                       | 数据库兼容的字节流类型               |
| `BlobTypeHandler`            | `byte[]`                       | `BLOB`, `LONGVARBINARY`              |
| `DateTypeHandler`            | `java.util.Date`               | `TIMESTAMP`                          |
| `DateOnlyTypeHandler`        | `java.util.Date`               | `DATE`                               |
| `TimeOnlyTypeHandler`        | `java.util.Date`               | `TIME`                               |
| `SqlTimestampTypeHandler`    | `java.sql.Timestamp`           | `TIMESTAMP`                          |
| `SqlDateTypeHandler`         | `java.sql.Date`                | `DATE`                               |
| `SqlTimeTypeHandler`         | `java.sql.Time`                | `TIME`                               |

在mybatis中对于这些基本类型会给我们自动适配对应的处理器，但对于其他一些我们自己定义的类型可能就不好处理了，我们可以自己定义类型处理器。比如我之前遇到的，我需要在数据库里把`varchar`类型查询出来转成我的`List<Integer>`类型（当然用resultmap的collection也可以）

对于自己定义自己的类型处理器，需要继承mybatis的`BaseTypeHandler<T>`里面有如下方法

![image-20221123111952587](C:/Users/ziyi zheng/Desktop/archive/backend_qingyou/mybatis/assets/1.png)

而我们需要实现如下四个方法，这里的方法用于干啥已经在方法名写的很明白了

![image-20221123112418184](C:/Users/ziyi zheng/Desktop/archive/backend_qingyou/mybatis/assets/2.png)

这里借鉴mybatisplus实现的一些架构写一个demo，有兴趣可以去看看mybatisplus里的AbstractJsonTypeHandler

```java
//这里的object可以换成自己想要转换的类型
public class MyTypeHandler extends BaseTypeHandler<Object> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Object parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, toJson(parameter));
    }

    @Override
    public Object getNullableResult(ResultSet rs, String columnName) throws SQLException {
        final String json = rs.getString(columnName);
        return !StringUtils.hasText(json) ? null : parse(json);
    }

    @Override
    public Object getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        final String json = rs.getString(columnIndex);
        return !StringUtils.hasText(json) ? null : parse(json);
    }

    @Override
    public Object getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        final String json = cs.getString(columnIndex);
        return !StringUtils.hasText(json) ? null : parse(json);
    }

    private  Object parse(String json){
        return JSON.parseObject(json, Object.class);
    }
//
    private  String toJson(Object obj){
        return JSON.toJSONString(obj);
    }
}

```

那我们自己定义的TypeHandler得用在哪里呢？来看看下面的简单例子

```xml
<resultMap type="com.example.springmvclearn.entity.User" id="User">

    <id column="id" property="id"/>
    <result column="uername" jdbcType="VARCHAR" property="username" 		                         	                                           typeHandler="com.example.springmvclearn.handler.MyTypeHandler"/>
    <result column="password" property="password"/>
</resultMap>
     <!--    这里只需要把我的typehandler对应的包名写到typeHandler=xxx即可-->
	<!-- 这里只是个举例，对于这个其实mybatis自己已经有了typeHandler-->
```

#### springboot配置mybatis

上面列出了那么多，我们先看看如何让springboot识别到我们的配置xml吧。

可以在我们的yml配置文件那么写，springboot就能自动识别到了

```yml
mybatis:
  mapper-locations: #指明mapper的位置
    - classpath:/mapper/*.xml
  config-location: classpath:/config/mybatisConfig.xml #指明配置文件位置
# 这里注意一下 mapper-locations可以有很多个，但是对于mybatis的配置文件就只有一个，所以不能那么写
#	config-location: 
#	- classpath:/config/mybatisConfig.xml 
#      ❌
```

同时呢，我们也可以直接在yml里写一些配置，聪明的idea也会给我们提示doge，比如我们之前讲到的typeAlias

```yml
mybatis:
  type-aliases-package: com.example.springmvclearn.entity
```

对于只需要简单的配置建议直接写yml里，但是如果自己有复杂的配置需求还是任然建议写个配置文件的xml。

##### classpath补充

先看张图吧，这是我将项目打成jar包之后解压的，这里由于篇幅原因，只讲讲里面的classes目录下的东西，也就是我们自己编写的文件，同时为了方便，读者可以看看自己运行之后产生的target目录里的classes目录

![image-20221129104823003](C:/Users/ziyi zheng/Desktop/archive/backend_qingyou/mybatis/assets/3.png)

classpath其实就是我们的类路径目录，在打包的时候，我们的/src目录以及/resources目录的文件都会被打包到classes下面，这里是不是和我们之前自己编写的代码的目录很像呢？。所以classpath就是告诉我要去哪个classes目录去找，找哪里呢？/config/mybatisConfig.xml

##### 开启日志

你可以选择直接用xml配置

对于现在的springboot的自动配置只需要修改自己的配置文件即可

```yml
logging:
  level:
    com.example.springmvclearn.mapper: debug 
    #这里是具体的包名
```

开启了之后即可看到下面输出的日志

![image-20221123102831975](assets\4.png)

## 结尾

其实还有很多没有介绍到的，这里只是选取了几个常见且易于理解的例子来讲解，真正工作中要有创造力，通过简单的例子来举一反三

总结一下resultmap的核心，首先寻找实体之间的关系（是一对多，还是一对一），然后思考这一组关系的结果集按什么进行区分（主键区分）