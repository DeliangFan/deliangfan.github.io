---
layout: post
title:  "复合主键 ORM 的 query.all()"
categories: 踩坑杂记
---

假定有如下表 test:

~~~ bash
mysql> show columns from test;
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
|  id1  | int(11) | NO   | PRI | 0       |       |
|  id2  | int(11) | NO   | PRI | 0       |       |
+-------+---------+------+-----+---------+-------+~~~

采用 sql 查询 id1 为 1 的结果如下：

~~~ bash
mysql> select * from test where id1=1;
+-----+-----+
| id1 | id2 |
+-----+-----+
|   1 |   1 |
|   1 |   2 |
+-----+-----+
~~~

采用 ORM 来查询所有 id1 为 1 的数据，代码如下：

~~~ python
from sqlalchemy import create_engine
from sqlalchemy import Column, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class Test(Base):
    __tablename__ = "test"
    id1 = Column(Integer, primary_key=True)
    id2 = Column(Integer) # My problem!, need to set Column(Interger, primary_key=True)

engine = create_engine("......")
Session = sessionmaker(bind=engine)
session = Session()
query = session.query(Test).filter_by(id1=1)

print length(query.all())
print query.count()
~~~

所得的结果为：

~~~ bash
1
2
~~~

之所以出现以上情况，是因为 ORM 会把查询的结果转换为 object，并且 SQLAlchemry 的 Identity Map 记录映射关系，映射依赖 ORM unique primary key(id1)。当执行 query.all() 时，第一个数据(id1=1, id2=1) 被转换为对象，并被 Identity Map 所记录，第二个 (id1=1, id2=2)) 被转换为对象时，ORM 发现该 key 已经存在，所以 query.all() 只返回一个对象。

Stackoverflow 中 [Querying for all results in SQLAlchemy 0.7.10](http://stackoverflow.com/questions/19409278/querying-for-all-results-in-sqlalchemy-0-7-10) 也阐述了类似的问题。[SQLalchemy 的解释](http://docs.sqlalchemy.org/en/rel_0_8/orm/session.html#what-does-the-session-do)如下：

> In the most general sense, the Session establishes all conversations with the database and represents a “holding zone” for all the objects which you’ve loaded or associated with it during its lifespan. It provides the entrypoint to acquire a Query object, which sends queries to the database using the Session object’s current database connection, populating result rows into objects that are then stored in the Session, inside a structure called the Identity Map - a data structure that maintains unique copies of each object, where “unique” means “only one object with a particular primary key”.

所以，写代码时，千万千万要小心！在处理 composite primary key 时，一定要保证 ORM 与数据库的格式保持一致性。
