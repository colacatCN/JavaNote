# 分布式 ID 实现方案

## 表结构

```bash
CREATE TABLE `distributed_id` (
  `table_name` varchar(128) NOT NULL,
  `max_id` bigint(20) NOT NULL,
  `step` bigint(20) NOT NULL,
  `description` varchar(128) DEFAULT NULL,
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`table_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

##  疑难杂症

**java.sql.SQLException: null,  message from server: "Host 'xxx' is not allowed to connect to this MySQL server"：**
[5.7] GRANT ALL PRIVILEGES ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION; 

**The driver has not received any packets from the server**

1. which mysql

2. /usr/bin/mysql --verbose --help |grep -A 1 'Default options'

[mysqld]
basedir = /usr/local/mysql
datadir = /data/mysql
tmpdir = /tmp
log_error = /data/logs/mysql/error.log
port = 3306
socket = /var/lib/mysql/mysql.sock
character-set-server = utf8

[mysql]
default-character-set = utf8

[client]
port = 3306
socket = /var/lib/mysql/mysql.sock
default-character-set = utf8
