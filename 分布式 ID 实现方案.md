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
