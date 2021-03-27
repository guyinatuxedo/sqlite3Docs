# Record Overflow

So this is another trick, that can be done via editing the database file to cause "interesting" behavior with the records. This was on sqlite3 version `3.35.1`. 

# Example

So we start off with this database:

```
sqlite> .open example
sqlite> CREATE TABLE x (y varchar, z int);
sqlite> INSERT INTO x (y, z) VALUES ("0000000000", 0);
sqlite> INSERT INTO x (y, z) VALUES ("1111111111", 1);
sqlite> INSERT INTO x (y, z) VALUES ("2222222222", 2);
sqlite> INSERT INTO x (y, z) VALUES ("3333333333", 3);
sqlite> INSERT INTO x (y, z) VALUES ("4444444444", 4);
sqlite> SELECT * FROM x;
0000000000|0
1111111111|1
2222222222|2
3333333333|3
4444444444|4
```

CREATE TABLE x (y varchar, z int);
INSERT INTO x (y, z) VALUES ("0000000000", 0);
INSERT INTO x (y, z) VALUES ("1111111111", 1);
INSERT INTO x (y, z) VALUES ("2222222222", 2);
INSERT INTO x (y, z) VALUES ("3333333333", 3);
INSERT INTO x (y, z) VALUES ("4444444444", 4);


```
0D 	02 	03 	21 	09 	31 	31 	31 	31 	31 	31 	31 	31 	31 	31
```


```
1C 	02 	03 	3F 	09 	31 	31 	31 	31 	31 	31 	31 	31 	31 	31
```


UPDATE x SET y = "999999999999999999999999" WHERE z = 1;

fewafwaefawfawfawf
NOT DONE