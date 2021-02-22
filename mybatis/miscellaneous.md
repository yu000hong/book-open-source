### 所有的数据库connection都来自于Transaction对象

Transaction接口方法：
- Connection getConnection()
- void commit()
- void rollback
- void close()
- Integer getTimeout()

