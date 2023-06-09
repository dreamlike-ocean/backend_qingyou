# 模板标签
> 本节内容来自于
>
> [mybatis – MyBatis 3 | XML 映射器](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#sql)
>



## sql标签

这个元素可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用。 参数可以静态地（在加载的时候）确定下来，并且可以在不同的 include 元素中定义不同的参数值。

这里很明显就能看出来 SQL标签一般是用于复用一组确定的列名 or 
```xml
<sql id="userColumns">${tablename}.id,${tablename}.username,${tablename}.password</sql>
<select id="getUser" resultType="com.example.springmvclearn.entity.User">
    select
        <include refid="userColumns"><property name="tablename" value="user"/></include> 
        from user where id=1;
</select>
<!--    这里等效于   
    select user.id,user.username,user.password from user where id=1;
-->
```

tablename对应下面include的name属性，用于确定要替换的位置

include中的value用于确定 tablename要替换成啥，这里就是替换成user

## #{}和${}

**#{}：**占位符，传入的内容会作为字符串**加上引号**，以**预编译**的方式传入，将 sql 中的 #{} 替换为 ? 号，调用 PreparedStatement 的 set 方法来赋值，有效的防止 SQL 注入，提高系统安全性

**${}：**拼接符，传入的内容会**直接替换**拼接，不会加上引号，可能存在 sql 注入的安全隐患 

* 能用 #{} 的地方就用 #{}，不用或少用 ${}
* 必须使用 ${} 的情况：
  * 表名作参数时，如：`SELECT * FROM ${tableName}`
  * order by 时，如：`SELECT * FROM t_user ORDER BY ${columnName}`
* sql 语句使用 #{}，properties 文件内容获取使用 ${} 

## 动态sql

动态 SQL 是 MyBatis 强大特性之一，逻辑复杂时，MyBatis 映射配置文件中，SQL 是动态变化的，例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。利用动态 SQL，可以彻底摆脱这种痛苦。使用动态 SQL 并非一件易事，但借助可用于任何 SQL 映射语句中的强大的动态 SQL 语言，MyBatis 显著地提升了这一特性的易用性。

动态SQL 包含的标签：

* if
* choose (when, otherwise)
* trim (where, set)
* foreach

### if

if标签使用是最常见的，和我们正常java语法的if差不多，看个例子就知道了

```xml
<select id="getUser" resultType="User">
    select * from user where 1=1
    <if test="id!=null">
        and id=#{id}
    </if>
</select>
<!--这里其实就是拼sql，必须要有个1=1来让id!=null也能成功
	id==null时：select * from user where 1=1
	id!=null时：select * from user where 1=1 and id=#{id}
-->
```

### choose、when、otherwise

这个类似与java语法中的switch与case和default，

- choose -> switch
- when -> case
- otherwise -> default

但是这里相比于switch，他里面的when和otherwize只会触发一次，类似于`if` `else if ` `else`，即代码块只会拼凑一个

接着看个栗子吧

```xml
<select id="getUser" resultType="User">
    select * from user where 1=1
    <choose>
        <when test="id!=null">
            and id=#{id}
        </when>
        <when test="username!=null and username!=''">
            and username=#{username}
        </when>
        <when test="password!=null and password!=''">
            and password=#{password}
        </when>
        <otherwise>
            and ~~~~自己定义吧
        </otherwise>
    </choose>
</select>
```

### trim、where、set

#### where

先来看*where* 吧，还记得上面我们写的`where 1=1`以及*if* 标签里面的and吗？，再来回顾一下上面可能出现的问题

```sql
-- 如果没有1=1的话且if标签判断的是false
select * from user where --戛然而止了
-- 如果没有1=1的话且if标签里面判断成功，但是没有and
select * from user where 1=1   ???   id=#{id} --缺少and  
```

为了是不是觉得为了解决sql拼凑的错误，要写这个1=1或者and很丑陋，在这里*where* 标签就解决了这个问题，以上面的if的例子来说明吧

```xml
<select id="getUser" resultType="User">
    select * from user
    <where>
        <if test="id!=null">
        and id=#{id}
    	</if>
    </where>
</select>
```

是不是看着舒服多了。看看*where* 标签做了啥呢？

*where* 标签只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 `AND` 或 `OR`，*where* 元素也会将它们去除。

#### trim

如果 *where* 元素与你期望的不太一样，你也可以通过自定义 trim 元素来定制 *where* 元素的功能。比如，和 *where* 元素等价的自定义 *trim* 标签。

*trim* 有如下参数可以选择

- prefix：给拼串后的整个字符串加一个前缀，trim 标签体中是整个字符串拼串后的结果
- prefixOverrides：去掉整个字符串前面多余的字符
- suffix：给拼串后的整个字符串加一个后缀
- suffixOverrides：去掉整个字符串后面多余的字符

我们类似可以这样实现*where* 标签类似的功能

```xml
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

来看个具体点的栗子

```xml
<select id="selectUserByUsernameAndPassword" resultType="User">
    SELECT * FROM user
    <trim prefix="where" prefixOverrides="and | or">
        <if test="username != null">
            AND username=#{username}
        </if>
        <if test="password != null">
            AND password=#{password}
        </if>
    </trim>
</select>
```

#### set

*set* ：进行更新操作的时候，含有 set 关键词，使用该标签

```xml
<!-- 根据 id 更新 user 表的数据 -->
<update id="updateUserById">
    UPDATE user
        <set>
            <if test="username != null and username != ''">
                username = #{username},
            </if>
            <if test="password != null and password != ''">
                password = #{password}
            </if>
        </set>
     WHERE id=#{id}
</update>
```

* 如果第一个条件 username 为空，那么 sql 语句为：update user u set password=? where id=?
* 如果第一个条件不为空，那么 sql 语句为：update user u set u.username = ? ,u.password = ? where id=?

同时*set* 其实也能用*trim* 实现，这里其实主要就是删除掉最后的逗号

```xml
<trim prefix="SET" suffixOverrides=",">
  ...
</trim>
```

### foreach

这个其实也与java语法中的for循环差不多

*foreach*提供了如下参数选择：

- collection 集合参数的名称,看个方法签名, `List<User> selectByIds(@Param("ids") List<Integer> ids);`这里只要填ids即可
- close  结束的 SQL 语句
- index 此时遍历到参数的下标(从0开始)
- item 参数变量名
- open  开始的 SQL 语句
- separator  分隔符

定义那么多，看个栗子吧

```xml
<select id="selectByIds" resultType="user" >
    SELECT * FROM user where id in
        <foreach collection="ids" open="(" close=")" item="id" separator=",">
            #{id}
        </foreach>
<!--这里foreach生成的就是(id1,id2,id3,~~~)这种啦-->
<!--注意collection="list" 这里意味着mapper的中接口的方法签名得是这样的List<User> selectByIds(List<Integer> ids)
也就是说容器类型得是list
-->
</select>
```

