---
layout: post
title: SQLAlchemy应用简介
date: 2014-09-06 15:32:50
disqus: y
---

# Introduction of SQLAlchemy

## Introduction
这个wiki将介绍如何将SQLAlchemy这个ORM在项目中应用。

### why
通过使用ORM来，封装了原有的sql语句，并且使得model层与controller层层次更加清晰。分离也使得model层易被复用。


通过ORM，对Service层提供了统一的接口，使得对于数据库中数据的调用更加方便快捷。

## sqlalchemy 使用简介
### 连接数据库
通过SQLAlchemy提供的`create_engine`模块建立与数据库的理解。在Athena中，这个连接在`model`层中`model_base`(`sep/frontend/model/model_base`)中建立。

	# To connect the database
	# The echo flag is a shortcut to setting up SQLAlchemy logging
	engine = create_engine("mysql://" + MYSQL_USER + ":" + MYSQL_PASSWORD +
                       "@"+ MYSQL_HOST + "/" + MYSQL_DB + "?charset=utf8", echo=False)

其中`echo`是用来对于SQLAlchemy进行日志的记载，其实现也是通过调用哪个Python的`loggin`模块实现的。

### 实体类与表的关联
SQLAlchemy通过`declarative_base`模块来将对应的类和表进行关联。一般一个应用只需设置一个`declarative base class`。这个设置也在`model`层中`model_base`(`sep/frontend/model/model_base`)中生成。

	# Declare a mapping
	Base = declarative_base()

此外，对应的**实体类需要继承`Base`**以完成关联。例如：数据库`Athena`中对应的`jobs`表对应的实体类如下

	class Job(Base):
		__tablename__ = 'jobs'
		__table_args__ = (
		    Index('job_id', 'id', 'source_id'),
		)
		
		id = Column(Integer, primary_key=True, nullable=False, index=True)
		job_name = Column(String(255), nullable=False)
		user = Column(String(45), nullable=False)
		job_status_id = Column(ForeignKey(u'jobs_status_map.id'), nullable=False, index=True)
		start_time = Column(Integer, nullable=False)
		last_time = Column(Integer, nullable=False)
		source_id = Column(ForeignKey(u'source.id'), primary_key=True, nullable=False, index=True)
		
		job_status = relationship(u'JobsStatusMap')
		source = relationship(u'Source')

其中一个实体类，至少需要`__table__`属性，为对应的表名， 而且至少需要一个Column为主键。

当数据库中有众多表时，如何从数据库自动生成对应的实体类模块？请见Q&A

### 建立session以与数据库进行交互
通过`sessionmaker`工厂类生成对应的session，从而建立与对应的数据库的联系。通过`sessionmaker`将之前的engine创建出对应的session，e.g.

	# Declare a senssion
	Session = sessionmaker(bind=engine)

此外，也可以通过`sessionmaker`先建立一个空的session，然后之后再将其对与对应的`engine`进行绑定，e.g.

	Session = sessionmaker()
	Session.configure(bind=engine)  # once engine is available

### 添加一个实体对象
例如，对于实体类`User`：

	class User(Base):
	    __tablename__ = 'users'
	    id = Column(Integer, primary_key=True)
	    name = Column(String(50))
	    fullname = Column(String(50))
	    password = Column(String(12))
	
	    def __repr__(self):
	        return "<User(name='%s', fullname='%s', password='%s')>" % (
	                                self.name, self.fullname, self.password)

添加一个实体对象到数据库：

	ed_user = User(name='ed', fullname='Ed Jones', password='edspassword')
	session.add(ed_user)

由于SQLAlchemy采用懒惰模式，所以`add()`后，这个记录并没有马上写到数据库中，而是到之后必要的时候，会被记录到数据库中。

添加多个实体对象到数据库：

	session.add_all([
    	User(name='wendy', fullname='Wendy Williams', password='foobar'),
    	User(name='mary', fullname='Mary Contrary', password='xxg527'),
    	User(name='fred', fullname='Fred Flinstone', password='blah')])

亦可以对对象的属性进行直接修改，如：

	ed_user.password = 'f8s7ccs'

通过调用`sension`的`commit()`方法来将数据写入/更新到数据库中,如：

	session.commit()

### 回滚
通过调用`session.rollback()`将之前的提交进行回滚
例如：

	>>> ed_user.name = 'Edwardo'
	>>> fake_user = User(name='fakeuser', fullname='Invalid', password='12345')
	>>> session.add(fake_user)
	>>> session.query(User).filter(User.name.in_(['Edwardo', 'fakeuser'])).all() 
	[<User(name='Edwardo', fullname='Ed Jones', password='f8s7ccs')>, <User(user='fakeuser', fullname='Invalid', password='12345')>]

之后滚后：
	
	>>> session.rollback()
	>>> ed_user.name 
	u'ed'
	>>> fake_user in session
	False
	>>> session.query(User).filter(User.name.in_(['ed', 'fakeuser'])).all() 
	[<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>]


### 查询
#### 一般查询
通过调用`session`的`query()`方法来进行查询，例如：
	
	>>> for instance in session.query(User).order_by(User.id): 
	...     print instance.name, instance.fullname
	ed Ed Jones
	wendy Wendy Williams
	mary Mary Contrary
	fred Fred Flinstone

或者：

	>>> for name, fullname in session.query(User.name, User.fullname): 
	...     print name, fullname
	ed Ed Jones
	wendy Wendy Williams
	mary Mary Contrary
	fred Fred Flinstone

调用`query`后的返回值类型是`named tuples`，可以看作是一般的python对象。

#### 排序
通过`order_by`进行排序，例如,

	session.query(User).order_by(User.id)

#### 查询过滤
通过调用`filter_by()`或者`filter()`进行过滤，比如：

通过`filter_by()`

	>>> for name, in session.query(User.name).\
	...             filter_by(fullname='Ed Jones'): 
	...    print name
	ed

通过`filter()`，其中`filter()`可以多次调用

	>>> for user in session.query(User).\
	...          filter(User.name=='ed').\
	...          filter(User.fullname=='Ed Jones'): 
	...    print user
	<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>

常见的过滤操作：

- equals

		query.filter(User.name == 'ed')

- not equals

		query.filter(User.name != 'ed')

- LIKE

		query.filter(User.name.like('%ed%'))

- IN

		query.filter(User.name.in_(['ed', 'wendy', 'jack']))

		# works with query objects too:
		query.filter(User.name.in_(
		        session.query(User.name).filter(User.name.like('%ed%'))
		))


- NOT IN

		query.filter(~User.name.in_(['ed', 'wendy', 'jack']))


- IS NULL

		query.filter(User.name == None)

		# alternatively, if pep8/linters are a concern
		query.filter(User.name.is_(None))


- IS NOT NULL

		query.filter(User.name != None)

		# alternatively, if pep8/linters are a concern
		query.filter(User.name.isnot(None))


- AND
		
		# use and_()
		from sqlalchemy import and_
		query.filter(and_(User.name == 'ed', User.fullname == 'Ed Jones'))
		
		# or send multiple expressions to .filter()
		query.filter(User.name == 'ed', User.fullname == 'Ed Jones')
		
		# or chain multiple filter()/filter_by() calls
		query.filter(User.name == 'ed').filter(User.fullname == 'Ed Jones')


- OR
	
		from sqlalchemy import or_
		query.filter(or_(User.name == 'ed', User.name == 'wendy'))


- MATCH

		query.filter(User.name.match('wendy'))

#### 查询返回值类型
- all()返回一个列表
		
		>>> query = session.query(User).filter(User.name.like('%ed')).order_by(User.id)
		>>> query.all() 
		[<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>,
      		<User(name='fred', fullname='Fred Flinstone', password='blah')>]


- first()返回查询结果集的第一个结果
	
		>>> query.first() 
		<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>

- one(), 查询所以记录，当且仅当只有一条符合条件的结果才返回，否则抛出异常

找到多条符合条件的记录

		>>> from sqlalchemy.orm.exc import MultipleResultsFound
		>>> try: 
		...     user = query.one()
		... except MultipleResultsFound, e:
		...     print e
		Multiple rows were found for one()

没有找到对应的记录

		>>> from sqlalchemy.orm.exc import NoResultFound
		>>> try: 
		...     user = query.filter(User.id == 99).one()
		... except NoResultFound, e:
		...     print e
		No row was found for one()

#### 计数
通过调用count()来进行查询到的结果进行计数统计，统计有多少符合条件的记录，例如,

	>>> session.query(User).filter(User.name.like('%ed')).count() 
	2

### 删除
将对应的实体对象删除:

		>>> jack = session.query(User).filter_by(name='jack').one()

		>>> session.delete(jack)
		>>> session.query(User).filter_by(name='jack').count() 
		0



## References
1. [SQLAlchemy 0.9 Documentation](http://docs.sqlalchemy.org/en/rel_0_9/orm/tutorial.html "SQLAlchemy 0.9 Documentation")
2. [sqlacodegen](https://pypi.python.org/pypi/sqlacodegen "sqlacodegen")

## Q&A



### 1. 如何自动生成数据库中表所对应的实体类？
通过调用[`sqlacodegen`](https://pypi.python.org/pypi/sqlacodegen "sqlacodegen"),可以自动生成对应SQLAlchemy所需的对应的实体类。首先需先下载`sqlccodegen`模块：
    
	pip install sqlacodegen
	------or------
	easy_install sqlacodegen

输入以下命令以生成对应的文件：

	sqlacodegen postgresql:///some_local_db
	sqlacodegen mysql+oursql://user:password@localhost/dbname
	sqlacodegen sqlite:///database.db

对于Athena的命令可以是：

	sqlacodegen mysql://MYSQL_USER:MYSQL_PASSWORD@MYSQL_HOST/MYSQL_DB?charset=utf8 > OUTPUT_FILE_NAME


