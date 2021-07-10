b树和b+树

一张Innodb表a，唯一索引userid，主键id，userid = 1时id = 1请问：

`select * from a where userid = 1;`和`select * from a where id = 1`;两者的执行效率一样吗？

不一样，主键不用回表，更快，非主键的索引的叶子结点值为对应的主键，要得到该行完整数据，需要回表。因此使用主键查询的更快。

使用非主键索引进行过滤，只有在`select id from a where userid = 1;`这种情况时不需要回表。