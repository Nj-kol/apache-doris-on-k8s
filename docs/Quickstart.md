**Test**

To test

```sql
create database demo;

use demo; 

create table mytable
(
    k1 TINYINT,
    k2 DECIMAL(10, 2) DEFAULT "10.05",    
    k3 CHAR(10) COMMENT "string column",    
    k4 INT NOT NULL DEFAULT "1" COMMENT "int column"
) 
COMMENT "my first table"
DISTRIBUTED BY HASH(k1) BUCKETS 1;

insert into mytable values
(1,0.14,'a1',20),
(2,1.04,'b2',21),
(3,3.14,'c3',22),
(4,4.35,'d4',23);

select * from mytable;
```