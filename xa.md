# MySQL XA Transactions


MySQL enables XA transactions by default. However, they are only supported by the InnoDB engine. XA transactions are by design, optimistic: the manager assumes that if a transaction can prepare completely, it can commit completely. This is not always the case.



[A Guide to Atomikos](https://www.baeldung.com/java-atomikos)

[MySQL XA Transactions](https://www.percona.com/blog/2018/05/16/mysql-xa-transactions)
