# pymssql 操作 Sql Server 

**注意在使用pymssql连接Sql Server数据库时，
若连接字符集使用utf8(charset="utf8")，那么当sql查询的字段类型为varchar且含有中文的时候需要使用encode("latin1").decode("gbk", "ignore")去处理查询结果，否则会出现乱码情况
该问题因为该字段是varchar而非nvarchar**

```python
connect = connect(charset="utf8")
cursor = connect.cursor()
sql = "INSERT INTO dbo.temp_table VALUES('测试', '测试', 2)"
sql = "SELECT * FROM temp_table" # 对于nvarchar类型结果会出现乱码
cursor.execute(sql)
connect.commit()
```



**若连接字符集使用cp936(charset="cp936")，那么查询的sql语句中含有中文的话，将会导致查询结果和预期不符合，因为cp936无法处理中文，且若查询的字段含有nvarchar会导致连接断开
但是cp936可以很好的处理varchar类型的结果不需要进行encode和decode操作**

```python
connect = connect(charset="cp936")
cursor = connect.cursor()
sql = "SELECT CONVERT(VARCHAR(500), nvarchar_field), varchar_field FROM temp_table"
cursor.execute(sql)
print(cursor.fetchall())
```



**因此推荐的做法应该是字符集采用utf8，同时含有中文的字段类型应该使用nvarchar而非varchar
或者当查询语句里面不含有中文的时候，但是结果集含有nvarchar类型同时含有中文，使用CONVERT来转换成varchar类型**

**在对sql语句中含有中文时使用utf8即可**

# 数据库查询建议

1. 对于update的数据先进行select
2. 可以使用临时表来缓存查询的结果
3. 需要大批量插入或者更新的时候，可以去考虑一次更新数万条记录写在一个sql语句中，然后循环执行该语句，比单独循环每一条sql语句的效率要高得多

# 数据库设计建议

1. 可以设置一个is\_valid字段来逻辑上的删除
2. 对于频繁修改的表，可以设置update_time以及updator来记录修改的数据，或者新增一个表来单独记录修改的数据，以便复原等操作

