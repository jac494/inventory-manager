# Notes

I've had this idea for a few different games I've played, where I'd like to be able to see the entire dependency mapping for a given item to be crafted (combine item-a + item-b to get item-c, combined 3 of item-c + 4 of item-d to get 2 of item-e in the game but be able query the total cost of item-e and see the dependency mapping in some sort of format (json, something similar to the output of `tree` command, idk))

Essentially a record of nested dictionaries. I need a database lol. That's it, I don't need a whole web app.

```sh
docker network create mysql-test-network
```

```sh
docker run \
  --network mysql-test-network \
  --name test-mysql \
  -e MYSQL_ROOT_PASSWORD=test \
  mysql
```

```sh
docker run -it \
  --network mysql-test-network \
  --rm mysql mysql \
  -htest-mysql \
  -uroot -p
```

```txt
[  5:44PM ]  [ jac494@hp-laptop:~/Projects/inventory_manager(main✗) ]
 $ docker run --network mysql-test-network --name test-mysql -e MYSQL_ROOT_PASSWORD=test mysql
2024-07-09 22:44:09+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.0.0-1.el9 started.
2024-07-09 22:44:10+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
2024-07-09 22:44:10+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 9.0.0-1.el9 started.
2024-07-09 22:44:10+00:00 [Note] [Entrypoint]: Initializing database files
2024-07-09T22:44:10.444264Z 0 [System] [MY-015017] [Server] MySQL Server Initialization - start.
...blahblahblah
```

```txt
[  5:47PM ]  [ jac494@hp-laptop:~/Projects/inventory_manager(main✗) ]
 $ docker run -it --network mysql-test-network --rm mysql mysql -htest-mysql -uroot -p        
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 9.0.0 MySQL Community Server - GPL

Copyright (c) 2000, 2024, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use nms
ERROR 1049 (42000): Unknown database 'nms'
mysql> create database nms
    -> ;
Query OK, 1 row affected (0.02 sec)

mysql> use nms;
Database changed
mysql>
```

```sql
DROP DATABASE IF EXISTS nms;
CREATE DATABASE nms;
USE nms;
```

```sql
DROP TABLE IF EXISTS Items;
CREATE TABLE Items (
    ItemID int NOT NULL AUTO_INCREMENT,
    ItemName varchar(255) UNIQUE,
    PRIMARY KEY (ItemID)
);
-- go ahead and create the first item which is None
-- 'base' items with no dependencies will use this in the ItemDependencyMap table to show that they are essentially a 'leaf' node in the dependency graph
INSERT INTO Items (ItemName) VALUES ("None");
```

```sql
DROP TABLE IF EXISTS ItemDependencyMap;
CREATE TABLE ItemDependencyMap (
    DependencyID int NOT NULL AUTO_INCREMENT,
    ParentItemName varchar(255),
    ChildItemName varchar(255),
    Quantity int NOT NULL,
    PRIMARY KEY (DependencyID),
    FOREIGN KEY (ParentItemName) REFERENCES Items(ItemName),
    FOREIGN KEY (ChildItemName) REFERENCES Items(ItemName)
);
```

```sql
DROP TABLE IF EXISTS ItemProductionResultQuantity;
CREATE TABLE ItemProductionResultQuantity (
    ItemProductionResultQuantityID int NOT NULL AUTO_INCREMENT,
    ItemName varchar(255),
    Quantity int NOT NULL,
    PRIMARY KEY (ItemProductionResultQuantityID),
    FOREIGN KEY (ItemName) REFERENCES Items (ItemName)
);
```

```sql
INSERT INTO Items (ItemName)
VALUES
("item_a"),
("item_b"),
("item_c"),
("item_d"),
("item_e");
```

```sql
INSERT INTO ItemDependencyMap (ParentItemName, ChildItemName, Quantity)
VALUES
("item_a", "None", 0),
("item_b", "None", 0),
("item_c", "item_a", 2),
("item_c", "item_b", 3),
("item_d", "item_a", 5),
("item_d", "item_c", 2),
("item_e", "item_b", 1),
("item_e", "item_c", 1),
("item_e", "item_d", 2);
```

```txt
mysql> select * from Items;
+--------+----------+
| ItemID | ItemName |
+--------+----------+
|      2 | item_a   |
|      3 | item_b   |
|      4 | item_c   |
|      5 | item_d   |
|      6 | item_e   |
|      1 | None     |
+--------+----------+
6 rows in set (0.00 sec)

mysql> select * from ItemDependencyMap;
+--------------+----------------+---------------+----------+
| DependencyID | ParentItemName | ChildItemName | Quantity |
+--------------+----------------+---------------+----------+
|            1 | item_a         | None          |        0 |
|            2 | item_b         | None          |        0 |
|            3 | item_c         | item_a        |        2 |
|            4 | item_c         | item_b        |        3 |
|            5 | item_d         | item_a        |        5 |
|            6 | item_d         | item_c        |        2 |
|            7 | item_e         | item_b        |        1 |
|            8 | item_e         | item_c        |        1 |
|            9 | item_e         | item_d        |        2 |
+--------------+----------------+---------------+----------+
9 rows in set (0.00 sec)

mysql>
```

next step is: I'll need to create queries that will show these mappings in some sort of view

The initial easy queries here at least answer the direct-dependency questions, idk how to traverse this graph within SQL though

```txt
mysql> select * from ItemDependencyMap where ParentItemName like "item_d";
+--------------+----------------+---------------+----------+
| DependencyID | ParentItemName | ChildItemName | Quantity |
+--------------+----------------+---------------+----------+
|            5 | item_d         | item_a        |        5 |
|            6 | item_d         | item_c        |        2 |
+--------------+----------------+---------------+----------+
2 rows in set (0.00 sec)

mysql> select * from ItemDependencyMap where ChildItemName like "item_b";
+--------------+----------------+---------------+----------+
| DependencyID | ParentItemName | ChildItemName | Quantity |
+--------------+----------------+---------------+----------+
|            4 | item_c         | item_b        |        3 |
|            7 | item_e         | item_b        |        1 |
+--------------+----------------+---------------+----------+
2 rows in set (0.00 sec)

mysql>
```
