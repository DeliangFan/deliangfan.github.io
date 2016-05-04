---
layout: post
title:  "踩雷小记 -- 主键非唯一时 ORM 的 query.all()"
categories: Database
---

假定有如下表 user:

~~~ bash
mysql> show columns from user;
+-----------+--------------+------+-----+---------+-------+
| Field     | Type         | Null | Key | Default | Extra |
+-----------+--------------+------+-----+---------+-------+
| name      | varchar(64)  | NO   | PRI | NULL    |       |
| age       | tinyint(1)   | NO   |     | NULL    |       |
+-----------+--------------+------+-----+---------+-------+
~~~

采用 sql 查询 age 为 10 的结果如下：

~~~ bash
mysql> select * from user;
+------+------+
| name | age  | 
+------+------+
| Sam  | 10   |
| Sam  | 12   |
| Tom  | 10   |
+------+------+
~~~

采用 ORM 来查询所有名为 Sam 的 user，代码如下：

~~~ python
class User(...):
	__tablename__ = 'user'
	name = Column(String(64), primary_key=True)
	age = Column(Integer)


engine = sqlalchemy.create_engine(...)
Session = sessionmaker(bind=engine)
session = Session()
query = session.query(User).filter(name='Sam')

print len(query.all())
print query.count()
~~~

所得的结果为：

~~~ bash
1
2
~~~

之所以会出现以上情况，是因为采用 ORM 查询所得的结果转换为 object，并且 SQLAlchemry 的 Identity Map 会记录映射关系，映射依赖 unique primary key。当 query.all() 时，第一个 Sam(age=0) 转换为对象，并且被 Identity Map 所记录，第二个 Sam(age=12) 转换为对象时，ORM 发现 primary key 为 Sam 的对象已经存在，第二个 Sam 被认为和第一个 Sam 是相同的，所以 query.all() 只返回一个对象。

Stackoverflow 中 [Querying for all results in SQLAlchemy 0.7.10](http://stackoverflow.com/questions/19409278/querying-for-all-results-in-sqlalchemy-0-7-10) 也阐述了类似的问题。[SQLalchemy 的解释](http://docs.sqlalchemy.org/en/rel_0_8/orm/session.html#what-does-the-session-do)如下：

> In the most general sense, the Session establishes all conversations with the database and represents a “holding zone” for all the objects which you’ve loaded or associated with it during its lifespan. It provides the entrypoint to acquire a Query object, which sends queries to the database using the Session object’s current database connection, populating result rows into objects that are then stored in the Session, inside a structure called the Identity Map - a data structure that maintains unique copies of each object, where “unique” means “only one object with a particular primary key”.


