---
layout: post
title: SQL Injection Lab
Description: In-Depth SQLi Lab
date: 2022-08-22 00:59:35  +0300
tags: [Pentesting,sql-injection]
image: '/assets/img/posts/sqlilab/injection.png'
featured: true
---

# SQL Injection Lab
This blog post will examine all kind of SQL injection techniques in detail. 
Sometimes it can be hard to understand how those injections work since there 
is no access to the backend code while conducting a penetration test to an application.
Attacker have to figure out how the backend implementation designed by examining the 
behaviour of the API in order to exploit those vulnerable endpoints.

However, instead of using some prepeared vulnerable training platform,
we will write our own vulnerable backend API. Therefore, we can precisely
see how our injections interacts with the code.

# SQL Injection Types

## In-Band SQLi
In-Band SQLi is the most common and easiest to exploit since the attacker can use the in-band 
network communication channel for both deploy the attack and extract data.
* ### Union-Based SQLi
Union-Based SQLi is an In-Band SQLi technique where attacker concats additional SELECT 
queries to the end of the original query in order to return multiple query results in a single HTTP response.
* ### Error-Based SQLi
Error Based SQLi is an In-Band SQLi technique and occurs when an attacker can see errors thrown by the 
database in the HTTP response. An attacker can exploit these errors and probe the database by manipulating the queries.
## Inferential-Blind SQLi 
Unlike In-Band SQLi, Blind-SQLi takes longer for attacker to gather data from the database since the endpoint does not
return the result of the query within the HTTP response. Therefore, no data being transferred In-Band.
Instead, the attacker extracts data from the differentiating behaviors in the application by sending 
various payloads to the application.
* ### Boolean-Based SQLi
Boolean-Based SQLi is an Blind-SQLi technique where the attacker gathers data by forcing the database to return
different responses depending on whether the query returns a TRUE or FALSE.
* ### Time-Based SQLi
Time-Based SQLi is a Blind SQLi technique where the attacker sends an SQL query to the database which forces it to wait
for specific a amount of time depending on whether the query returns a TRUE or FALSE.
## Out-Of-Band SQLi
Out-Of-Band SQLi is the last resort and needed when the attacker can not infer any differentiation via in-band
responses after trying all those methods mentioned above. It is the most difficult method used to extract data
from the database, as it is variable according to the type of database. 

# HANDS-ON PRACTICE

First of all, lets create a database
and implement some tables in it. We will use PostgreSql for that purpose.

<br>![](/assets/img/posts/sqlilab/1.png){:.normal}

We now have a database called **db1** and two tables named as **users** and **products**.
The users table will mimic as login credentials. We wont be implementing any login page 
for that, we will aim to extract data from that table by using product display pages 
by assuming we already have logged in with some account and have access to those product 
display endpoints.

<br>![](/assets/img/posts/sqlilab/2.png){:.normal}

First thing is to connect the database to the application. Second create endpoint
as **/api/products**. This page displays all the products exist in our database. 
This endpoint does not accept any input from http request.  Therefore it is impossible 
to inject any parameters to that query.

<br>![](/assets/img/posts/sqlilab/3.png){:.normal}

The next endpoint will accept a parameter to scan a product with a specific id in the database. 
This parameter, which will be sent to the endpoint, will be used as the id of the queried product in the 
database.

Send **1** as the parameter.
<br>![](/assets/img/posts/sqlilab/4.png){:.normal}

Send **2** as the parameter.
<br>![](/assets/img/posts/sqlilab/5.png){:.normal}

Send **3** as the parameter.
<br>![](/assets/img/posts/sqlilab/6.png){:.normal}

The problem with that implementation, it can be seen in the code, the application does not perform
any input validation. Therefore an attacker can use any special character in parameter and perform arithmetical
operations in the database. The way to detect whether or not there is an sql injection, you should pick a reference point first.
If you can find a way to go back to that reference point with an unexpected parameter, it means the enpoint is vulnerable to SQLi.

<br>![](/assets/img/posts/sqlilab/8.png){:.normal}

Passing both **1** and **3-2** as parameters bringing the same response. After that point
an attacker can easily pipe that database to himself with right methods.

## UNION BASED SQLi
If the database returns the output of the query within the HTTP response, you can use perform **UNION** based
sql injection. In order for this attack to work, number of the columns comes from **UNION SELECT** must match 
with the original query. Also type of those columns should be same.

<br>![](/assets/img/posts/sqlilab/9.png){:.normal}

Eventually the following query will be performed at server side.
{% highlight sql %}
SELECT * FROM users UNION SELECT 1,'A',3;
{% endhighlight  %}

In every database there are tables to hold all the metadata related to that database. **information_schema.tables**
holds all the table names withing that database.

All databases have a table to hold user credentials. General rule of thumb, those tables named as **users**. It can
have a specific prefix or suffix, such as knucklesec_users for example. In this case **users** is the name of it.

There is an additional row in the HTTP response when
an existing table is queried and no additional rows for the unexisting table.
<br>![](/assets/img/posts/sqlilab/12.png){:.normal}
<br>![](/assets/img/posts/sqlilab/13.png){:.normal}

Now, it is known that there is a table named **users**, however it also needed to know the column names of that table.
<br>![](/assets/img/posts/sqlilab/15.png){:.normal}
Two rows have returned from that query. Column names are **username** and **password** respectively.
<br>
Extraction of the values in these columns:
First **usernames**.
<br>![](/assets/img/posts/sqlilab/16.png){:.normal}

Then **passwords**.
<br>![](/assets/img/posts/sqlilab/17.png){:.normal}

## Inferential-Blind SQLi/Boolean-Based

It is called **Blind SQLi** when the application still can be injected with some SQLi payloads, but does not return
the result of that query within HTTP response. Since it is not possible to see any rows and columns, UNION 
keyword won't help. It is still possible to extract data from the database with different techniques.

Code has been changed to alter the behavior of the application. This time instead of returning the details of the queried product,
application will inform the user whether or not the product exists.
<br>![](/assets/img/posts/sqlilab/18.png){:.normal}
<br>![](/assets/img/posts/sqlilab/20.png){:.normal}
<br>![](/assets/img/posts/sqlilab/21.png){:.normal}

This time the result of the query can not be seen within the HTTP response. Regardless, the backend still vulnerable for injection.
<br>![](/assets/img/posts/sqlilab/22.png){:.normal}

It is possible to extract data with **AND** keyword in this keys.

The following query will find the product with the id of 1 if 1=1. Since
1 equals to 1, product will be found.
{% highlight sql %}
SELECT * FROM users WHERE * id=1 AND 1=1;
{% endhighlight  %}

The following query will find the product with the id of 1 if 1=2. Since
1 is not equal to 2, database won't return any product.

{% highlight sql %}
SELECT * FROM users WHERE id=1 AND 1=2;
{% endhighlight  %}

Launch **BurpSuite**.
<br>![](/assets/img/posts/sqlilab/26.png){:.normal}
<br>![](/assets/img/posts/sqlilab/27.png){:.normal}
<br>![](/assets/img/posts/sqlilab/28.png){:.normal}
<br>![](/assets/img/posts/sqlilab/29.png){:.normal}

This behaviour opens up a door. One can observe that there are two different responses depending on
the validness of the right side of the query.

With the help of the **SUBSTR()** function, a subquery can be written. 

{% highlight sql %}
1 AND SUBSTR((SELECT table_name FROM information_schema.tables WHERE table_name='some table name'),1,50)='some table name'
{% endhighlight  %}

SELECT method will return the table name of the queried table and compare it with our guess.


If there is a table called **some table name**, it will be equal to **some table name**, the condition will be
**true**. However, if there is not any table with that name, query will return empty string, 
since an empty string is not equal to some unempty string value, the condition will be **false**.

TRUE AND TRUE = Product exists<br>
TRUE AND FALSE = Product does not exists.

<br>![](/assets/img/posts/sqlilab/31.png){:.normal}
<br>![](/assets/img/posts/sqlilab/32.png){:.normal}
<br>![](/assets/img/posts/sqlilab/33.png){:.normal}
<br>![](/assets/img/posts/sqlilab/34.png){:.normal}

Next, find column names of the **users** table.
The number of rows which will be returned should be limited to a single row since the SUBSTR() method can only compare a single string 
value with another. Therefore, database have to return only the first row of the query result, which is the first column name of 
the **users** table. If there would not any prior knowledge about the database, all the permutations had to be brute-forced. 
It can be done with the **Cluster Bomb**  utility in **BurpSuite**.
<br>![](/assets/img/posts/sqlilab/41.png){:.normal}
<br>![](/assets/img/posts/sqlilab/42.png){:.normal}
<br>![](/assets/img/posts/sqlilab/39.png){:.normal}
<br>![](/assets/img/posts/sqlilab/40.png){:.normal}
<br>![](/assets/img/posts/sqlilab/43.png){:.normal}
It is validated that there is a row called **username** in the **users** table. 

Let's see if there is an admin user.
<br>![](/assets/img/posts/sqlilab/44.png){:.normal}
<br>![](/assets/img/posts/sqlilab/75.png){:.normal}

It is verified that there is an admin user. However, it is not possible to guess the
password. The characters of the password must be brute-forced one by one.
<br>![](/assets/img/posts/sqlilab/46.png){:.normal}
<br>![](/assets/img/posts/sqlilab/47.png){:.normal}

It is not practical to do it manually. Therefore, it is best to use an automated brute-force attack tool.
In this case a **Sniper** attack would do the job which is an **Intruder** utility exists in BurpSuite.
<br>![](/assets/img/posts/sqlilab/52.png){:.normal}
First character is **c**. 
<br>![](/assets/img/posts/sqlilab/53.png){:.normal}
<br>![](/assets/img/posts/sqlilab/54.png){:.normal}
<br>![](/assets/img/posts/sqlilab/55.png){:.normal}
Second character is **o**. 
<br>![](/assets/img/posts/sqlilab/56.png){:.normal}

By doing the same attack for each character, we can find the password is 'cokgizlisifre' eventually.

## Error-Based SQLi

There are two different **Error-Based SQLi** types. 
* Static Error Message/Boolean-Based: Seeing the same error message each time for all kind of errors.
* Dynamic Error Message/In-Band: Seeing the exact database error log within the HTTP response.

### Static Error Message/Boolean-Based

What if the the endpoint does not behave any differently whether or not the query is successfull or not?
In this case it is not possible extract any data by observing the behaviour of the application.
Let's see the altered code.
<br>![](/assets/img/posts/sqlilab/59.png){:.normal}
As you can see endpoint is embedding the same response in HTTP response. When this is the case,
we use another SQLi techique, which is called **Error Based SQLi**. Practically, we inject an illegitimate 
sql query which will force the application to spawn an error. The point is to create our own conditional response.

First let's see how the API behaves naturally.
<br>![](/assets/img/posts/sqlilab/57.png){:.normal}
<br>![](/assets/img/posts/sqlilab/58.png){:.normal}

We should test whether the application behaves accordingly when it receives an illegitimate parameter to query.
General rule of thumb is to trigger a **division by zero** error. No database can divide an integer to zero (it is not possible in algebra).
<br>![](/assets/img/posts/sqlilab/60.png){:.normal}
<br>![](/assets/img/posts/sqlilab/61.png){:.normal}

In this case the application does not manage the error gracefully, therefore we generated our own conditional response.

{% highlight sql %}
1 AND 1=(SELECT CASE WHEN (3=2) THEN 'illegitimate subquery here' ELSE 1 END);
{% endhighlight  %}

When the **CASE** condition is met, in this case if 3=2, we will trigger an error by injecting an illegitimate subquery, else
**CASE** will select 1. When the **CASE** keyword selects the **ELSE** block, which is 1. Finally, this is how the right side of 
the query will look like;

{% highlight sql %}
1 AND 1=1;
{% endhighlight  %}
No errors.<br>

However when we inject it like this;
{% highlight sql %}
1 AND 1=(SELECT CASE WHEN (3=3) THEN 'illegitimate subquery here' ELSE 1 END);
{% endhighlight  %}
Since 3 is equal to 3, it will execute the **THEN** block, and try to execute an illegitimate subquery.
That query will return the error page.<br>

The illegitimate payload we will use in this example will be;
{% highlight sql %}
cAsT(chr(126)||vErSiOn()||chr(126) aS nUmeRiC)
{% endhighlight  %}

It is not possible for database to cast **CHARACTER** type value as numeric. Therefore, it will generate an error.

**Note:** There are tons of payloads for error-based-sqli and for other types online, the ones I am using for
demonstration purposes.

Instead of testing if 3=3 or 3=2, **CASE WHEN** block will test if a certain named table exists or not by using **SUBSTR()**
function, same as how we did for **Conditional SQLi** above.
<br>![](/assets/img/posts/sqlilab/76.png){:.normal}
<br>![](/assets/img/posts/sqlilab/77.png){:.normal}
Since the condition does not met, no error message have generated by the database.
<br>![](/assets/img/posts/sqlilab/78.png){:.normal}
<br>![](/assets/img/posts/sqlilab/79.png){:.normal}
Since the condition is met, error message have generated by the database.

Let's further extract data from the database by using this technique with altering the payload.
<br>![](/assets/img/posts/sqlilab/68.png){:.normal}
<br>![](/assets/img/posts/sqlilab/69.png){:.normal}
<br>![](/assets/img/posts/sqlilab/70.png){:.normal}
<br>![](/assets/img/posts/sqlilab/71.png){:.normal}
<br>![](/assets/img/posts/sqlilab/72.png){:.normal}
<br>![](/assets/img/posts/sqlilab/74.png){:.normal}


### Dynamic Error Message/In-Band

The application might return the exact databse error in the HTTP response message. It is easier to extract data
from that kind of implementation since it is an in-band vulnerability.
<br>![](/assets/img/posts/sqlilab/89.png){:.normal}
<br>![](/assets/img/posts/sqlilab/80.png){:.normal}
<br>![](/assets/img/posts/sqlilab/81.png){:.normal}
<br>![](/assets/img/posts/sqlilab/82.png){:.normal}
<br>![](/assets/img/posts/sqlilab/83.png){:.normal}
<br>![](/assets/img/posts/sqlilab/84.png){:.normal}
<br>![](/assets/img/posts/sqlilab/85.png){:.normal}
<br>![](/assets/img/posts/sqlilab/86.png){:.normal}
<br>![](/assets/img/posts/sqlilab/87.png){:.normal}

**SUBSTRING()** can be used for further exploitation same as before after this point.

## Inferential-Blind SQLi/Time-Based

What if the endpoint responding the same even if every method explained above had been tried?
<br>This is the implementation:
<br>![](/assets/img/posts/sqlilab/88.png){:.normal}

The application will return the same string for each scenario. However, it is still vulnerable for SQLi.
Data still can be extracted by using **Time-Based SQLi** methods.<br>

The endpoint executes the query and sending the response synchronously [**(See synch/asynch)**](https://www.geeksforgeeks.org/difference-between-synchronous-and-asynchronous-method-of-fs-module/).
Which means that, the response will be sent after the query is completed.<br><br>
**Note:** The client.query() block is asynchron for the rest of the code, which means it is going to be executed in a seperate thread.
However, request-query-response cycle has a  blocking structure, therefore synchronous in itself. 

Expected response:
<br>![](/assets/img/posts/sqlilab/89.png){:.normal}

Conditional triggering using **SELECT CASE** structure. 
{% highlight sql %}
SELECT CASE WHEN (5=0) THEN pg_sleep(3) END;
{% endhighlight  %}
Database will sleep for three seconds after the first query
is completed if the condition of the second one is met. Therefore, it is now possible to extract data from the database by triggering time delays.
Response time will be different accordingly whether the condition is met or not.
<br>![](/assets/img/posts/sqlilab/90.png){:.normal}
<br>![](/assets/img/posts/sqlilab/91.png){:.normal}

Combining time triggering and conditionals, further exploitation can be done same as how it had been done at the examples above.
<br>![](/assets/img/posts/sqlilab/92.png){:.normal}
<br>![](/assets/img/posts/sqlilab/93.png){:.normal}
<br>![](/assets/img/posts/sqlilab/94.png){:.normal}
<br>![](/assets/img/posts/sqlilab/95.png){:.normal}

## Out-Of-Band SQLi 

Application might run the query asynchronously. Or endpoint can pass the parameter to another microservice to run the sql query.
Regardless of which, it is not possible to use time-based technique, since application would not wait for sql query to
complete in order to respond to the client.
<br>![](/assets/img/posts/sqlilab/88.png){:.normal}

As you see, the application will return the same string for each scenario. However, it is still vulnerable for SQLi.
We can still extract data by using **OOB** methods.<br>
Out-of-Band means triggering an interaction from database to an outer network by using some network protocol. DNS is
a good fit for that purpose since services, including all the database services, needs to resolve the IP address 
of a given domain name.
