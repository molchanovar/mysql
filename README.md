# mysql
### AutoDeploy Master-Slave Percona configuration 

`cat ~/.mysql_history` - просмотр истории запросов Mysql
`SELECT user FROM mysql.user;` - просмотр созданных пользователей
`show variables;` - вся настроичная таблица (все настройки)
`show status;` - # метрики (для анализа - что происходит ?)

- FUI cmd:
```
show databases; 
use <databases_name>;
show tables; 
select * from <table_name>;
```

### ACID ?  
### MVCC ?

#### Настройка реплики (GTID - как минимум одна реплика должна записать транзакцию в журнал, прежде чем та будет признана успешной)
Установка Percona (CentOS 7): 
```
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
yum install Percona-Server-server-57 -y
```
Основной конфиг - `/etc/my.cnf` 
Директория с конфигами - `/etc/my.cnf.d/`
Дата файлы - `/var/lib/mysql`

#### MASTER

Запускаем службу MySQL, находим автосгенерованный пароль, заходим в DB и меняем его: 
```
systemctl start mysql
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
mysql -uroot -p'{_PASSWORD_FROM_/var/log/mysqld.log_}'

mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';
```

Проверяем server-id на Мастере (Должен отличаться от Слэйва)
```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```

Проверяем GTID: 
```
mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.00 sec)

```

Проверяем таблицы для репликации: 
```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| bet                |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> USE bet;

mysql> show tables;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)
```

Создаем пользователя с правами на репликацию: 
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '123PassWeryStrong()';
mysql> SELECT user,host FROM mysql.user where user='repl';
+------+-----------+
| user | host |
+------+-----------+
| repl | % |
+------+-----------+
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '123PassWeryStrong()';
```

Снимаем дамп: 
```
[root@master etc]#  mysqldump --all-databases --triggers --routines --events --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master2.sql
```


#### SLAVE 
Прaвим в `/etc/my.cnf.d/01-basics.cnf` - `server-id = 2`
```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)
```

Добавление таблиц которые будут игнорироваться при бэкапе: 
/etc/my.cnf.d/05-binlog.cnf  `replicate-ignore-table=bet.events_on_demand`



