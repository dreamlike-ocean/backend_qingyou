
# insert，update，delete

> 本节内容来自于
>
> [mybatis – MyBatis 3 | insert update and delete](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#insert_update_and_delete)
>

数据变更语句 insert，update 和 delete 的实现非常接近，使用起来也差不多

## insert

正常插入一行数据

```xml
<insert id="insertOneUser">
        insert into `user`(`username`,`password`)  values (#{username},#{password});
</insert>
<!--这里如果插入成功会返回插入的条数-->
```

同时你还可以选择插入多条数据(这里属于动态sql,之后会讲到)

```xml
<insert id="insertUser">
    insert into `user`(`username`,`password`)  values
    <foreach item="item" collection="list" separator=",">
        (#{username},#{password})
    </foreach>
</insert>
<!--    这里执行效果大概是这样的，如标签所言，就是foreach的增加sql   
    insert into `user`(`username`,`password`)  values (#{username},#{password}),(#{username},#{password})~~~
-->
```

## update

和正常些insert差不多，看个例子

```xml
<update id="updateUser">
    UPDATE user SET uasername=#{username},password=#{password} WHERE id=#{id}
</update>
    <!--这里如果更新成功会返回更新的条数-->
```

## delete

接着看例子

```xml
<delete id="deleteUser">
    DELETE FROM user WHERE id=#{id}
</delete>
<!--这里如果删除成功会返回删除的条数-->
```

## 注解开发

注解开发适合在写一些比较简单的sql的场景，就没有必要写xml啥的了，很简单，看个例子就知道了

```java
public interface UserMapper {
    //查询全部
    @Select("SELECT * FROM user")
    public abstract List<User> selectAll();

    //新增数据
    @Insert("INSERT INTO user VALUES (#{id},#{username},#{password})")
    public abstract Integer insert(User user);

    //修改操作
    @Update("UPDATE user SET uasername=#{username},password=#{password} WHERE id=#{id}")
    public abstract Integer update(User user);

    //删除操作
    @Delete("DELETE FROM user WHERE id=#{id}")
    public abstract Integer delete(@Param("id")Integer id);

}
```
