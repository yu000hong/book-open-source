# MyBatis Scripting - Dynamic SQL

MyBatis支持动态SQL特性，你可以在settings里配置`defaultScriptingLanguage`属性来定义我们使用的脚本引擎。目前已知的脚本引擎有：

- XMLLanguageDriver
- RawLanguageDriver
- FreeMarkerLanguageDriver
- VelocityLanguageDriver

除了可以设置全局的脚本引擎属性，你还可以针对特定的statement指定特定的脚本引擎属性，如：

**全局属性**

```xml
<typeAliases>
  <typeAlias type="org.sample.MyLanguageDriver" alias="myLanguage"/>
</typeAliases>
<settings>
  <setting name="defaultScriptingLanguage" value="myLanguage"/>
</settings>
```

**特定设置**

```xml
<select id="selectBlog" lang="myLanguage">
  SELECT * FROM BLOG
</select>
```

### XMLLanguageDriver

这是MyBatis默认的脚本引擎，它支持在SQL里面添加如下标签：

- where/set/trim
- choose/when/otherwise
- if
- foreach
- bind

例如：

```xml
<select id="mget" resultType="Area">
    SELECT * FROM area WHERE id IN (
        <foreach collection="areaIds" item="areaId" separator=",">#{areaId}</foreach>
    )
</select>
```

### RawLanguageDriver

它继承自`XMLLanguageDriver`，功能相同，但是它会禁止使用动态SQL，当发现statement里面包含动态SQL的话，就会直接抛出异常：**Dynamic content is not allowed when using RAW language**。

### FreeMarkerLanguageDriver

官方文档：[freemarker-scripting](http://mybatis.org/freemarker-scripting/)

支持我们使用FreeMarker来编写动态SQL，配置如下：

```xml
<typeAliases>
    <typeAlias alias="freemarker" type="org.mybatis.scripting.freemarker.FreeMarkerLanguageDriver"/>
</typeAliases>
<settings>
    <setting name="defaultScriptingLanguage" value="freemarker"/>
</settings>
```

**使用示例**：

```
<!-- This is handled by FreeMarker too, because it is included into select nodes AS IS -->
<sql id="cols">id, ${r"firstName"}, lastName</sql>

<select id="findName" resultType="org.mybatis.scripting.freemarker.Name" lang="freemarker">
    findName.ftl
</select>

<select id="findNamesByIds" resultType="org.mybatis.scripting.freemarker.Name" lang="freemarker">
    select <include refid="cols"/> from names where id in (${ids?join(',')})
</select>

<!-- It is not very convenient - to use CDATA blocks. Better is to create external template
    or use more verbose syntax: ${r"#{id}"}. -->
<select id="find" resultType="org.mybatis.scripting.freemarker.Name" lang="freemarker">
    select * from names where id = <![CDATA[ <@p name='id'/>]]> and id = ${id}
</select>
```

### VelocityLanguageDriver

官方文档：[velocity-scripting](http://mybatis.org/velocity-scripting/)

支持我们使用Velocity来编写动态SQL，配置如下：

```xml
<typeAliases>
    <typeAlias alias="velocity" type="org.mybatis.scripting.velocity.VelocityLanguageDriver"/>
</typeAliases>
<settings>
    <setting name="defaultScriptingLanguage" value="velocity"/>
</settings>
```

**使用示例**：

```xml
<select id="findPerson" lang="velocity">
  #set( $pattern = $_parameter.name + '%' )
  SELECT *
  FROM person
  WHERE name LIKE @{pattern, jdbcType=VARCHAR}
</select>
```
