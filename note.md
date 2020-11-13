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

# 对于异常的建议

1. 应该自定义异常来描述具体的异常信息

   ```python
   class RequestException(Exception):
   	"""There was an ambiguous exception that occured while handling your request."""
   
   class AuthenticationError(RequestException):
   	"""The authentication credentials provided were invalid."""
   	
   class URLRequired(RequestException):
   	"""A valid URL is required to make a request."""
   	
   class InvalidMethod(RequestException):
   	"""An inappropriate method was attempted."""
   
   ```

   

# 对于一个可由用户扩展的功能

1. 可以设置一个全局变量，提供一个修改该全局变量的方法

   ```python
   def add_autoauth(url, authobject): # url可以替换成用户需要执行的函数，或者模块
   	"""Registers given AuthObject to given URL domain. for auto-activation.
   	Once a URL is registered with an AuthObject, the configured HTTP
   	Authentication will be used for all requests with URLS containing the given
   	URL string.
   
   	Example: ::
   	    >>> c_auth = requests.AuthObject('kennethreitz', 'xxxxxxx')
   	    >>> requests.add_autoauth('https://convore.com/api/', c_auth)
   	    >>> r = requests.get('https://convore.com/api/account/verify.json')
   	    # Automatically HTTP Authenticated! Wh00t!
   
   	:param url: Base URL for given AuthObject to auto-activate for.
   	:param authobject: AuthObject to auto-activate.
   	"""
   	global AUTOAUTHS
   	
   	AUTOAUTHS.append((url, authobject))
   ```

   

2. 根据目录结构来动态导入包

   ```python
   def async_spider():
       package = "spider"
       modules = get_modules(package) # 所有需要动态导入的模块文件名
       pool = Pool(processes=4)
       for module in modules:
           module = importlib.import_module(module, package) # 采用相对路径导入
           try:
               main_func = getattr(module, "main")
           except AttributeError as e:
               continue
           else:
               # main_func()
               pool.apply_async(main_func, tuple())
       pool.close()
       pool.join()
   ```

   

# 对于Mixin的设计

mixin的类只用于混入别的类中、不应该被实例化、没有自己的\_\_init\_\_方法、不存在实例属性，从而在被混入的类中调用super也就不涉及到调用混乱的情况了

## 原则

1. Mixin 实现的功能需要是通用的，并且是单一的，比如上例中两个 Mixin 类都适用于大部分子类，每个 Mixin 只实现一种功能，可按需继承。
2. Mixin 只用于拓展子类的功能，不能影响子类的主要功能，子类也不能依赖 Mixin。比如上例中 `Person` 继承不同的 Mixin 只是增加了一些功能，并不影响自身的主要功能。如果是依赖关系，则是真正的基类，不应该用 Mixin 命名。
3. Mixin 类自身不能进行实例化，仅用于被子类继承。

```python
In [1]: class MappingMixin:
   ...:     def __getitem__(self, key):
   ...:         return self.__dict__.get(key)
   ...:
   ...:     def __setitem__(self, key, value):
   ...:         return self.__dict__.set(key, value)
   ...:

In [2]: class ReprMixin:
   ...:     def __repr__(self):
   ...:         s = self.__class__.__name__ + '('
   ...:         for k, v in self.__dict__.items():
   ...:             if not k.startswith('_'):
   ...:                 s += '{}={}, '.format(k, v)
   ...:         s = s.rstrip(', ') + ')'  # 将最后一个逗号和空格换成括号
   ...:         return s
   ...:

In [3]: class Person(MappingMixin, ReprMixin):
   ...:     def __init__(self, name, gender, age):
   ...:         self.name = name
   ...:         self.gender = gender
   ...:         self.age = age
   ...:

In [4]: p = Person("小陈", "男", 18)
   ...: print(p['name'])  # "小陈"
   ...: print(p)  # Person(name=小陈, gender=男, age=18)
小陈
Person(name=小陈, gender=男, age=18)
```

