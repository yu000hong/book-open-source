# 参数相关

- ParameterHandler
- ParameterMapping

### ParameterMapping

ParameterMapping包含了参数所有的元数据信息，有：

- property: 参数名称
- mode: 参数类型(IN/OUT/INOUT)
- javaType: Java类型
- jdbcType: JDBC类型
- typeHandler: 对应的类型处理器
- resultMapId:
- expression: 
- numericScale:

### ParameterHandler

ParameterHandler的作用就是将参数设置到`PreparedStatement`对象里面去，只有唯一实现`DefaultParameterHandler`。

ParameterHandler接口源码：

```java
public interface ParameterHandler {

  Object getParameterObject();

  void setParameters(PreparedStatement ps) throws SQLException;

}
```