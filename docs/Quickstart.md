## Doris Quickstart

```sql
create database demo;

use demo; 

CREATE TABLE IF NOT EXISTS demo.user_data (
    user_id INT,
    name STRING,
    age INT,
    update_time DATETIME
)
UNIQUE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 3
PROPERTIES (
    "replication_num" = "1",
    "enable_unique_key_merge_on_write" = "true"
);

insert into demo.user_data values
(1, "Alice", 30, "2024-01-01 10:00:00"),
(2, "Bob", 25, "2024-01-01 11:00:00");

select * from demo.user_data;

-- Upsert data
insert into demo.user_data values
(1, "Alice", 31, "2024-01-02 12:00:00"),
(3, "Charlie", 22, "2024-01-02 13:00:00");
```