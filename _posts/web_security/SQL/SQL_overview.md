### Note:
These are my notes from working through the Port swigger Web Academy 
# SQL 

SQL is a database query language and a SQL injection attack is a common attack that allows the attacker to directly interfere with the quires

This normally allows an attacker to view, modify, & delete data directly to the back end web system.

In some cases this can go straight to compromising the back end system


## Attacks

### Bypassing checks
 These attacks are based around the fact that user input is put into an quire without proper sanitation
 
 For example 
![[Snippets#Snip 1 - Basic where]]
 By giving you're user input as:
 ```SQL
 stuff' --
 ```
 you will get all of the products in category stuff even if they are not released.
 
 This is because the "--" is the comment command for SQL and the interpreter.
 
 This can also be the case when dealing with log in portals where the quire for log in is:
![[Snippets#Snip 2 - User Password check]]
 Because just putting in whatever username'-- shorts out the password check
 
 ### Accessing other tables - UNION attacks
 
 When the request returns your quiries (like when showing shopping items) you can use the union operator to dump more items for you to see,
 
 Take Snip 1 for example
![[Snippets#Snip 1 - Basic where]]
 If you use a payload with UNION you get more things that aren't just in the products table. For example:
 ```SQL
 stuff' UNION SELECT username, password FROM user--
 ```
 Will get the username and password from the site as well as the products. [[UNION attacks]] Holds more.
 
 This can also be very useful for dumping the overall table info, however what works to get database info depends on what database is running
 
 ### Blind attacks
 
 Not all injections give you visible results, however just because you can't see the results of an injection doesn't mean it didn't happen. These are blind attacks can be a little difficult to exploit. 
 
 To figure out if they even exist here are a couple methods:
 
 #### Logic error
  This method revolves around causing some logic error that causes the server state to change in a way you can detect. For example a divide by 0 error my cause your query to fail which tells you it evaluated your input and is vulnerable
  
  #### Time Delay 
   If you can cause a time delay to happen you can judge based on how long it takes your query to resolve if you got your input to be evaluated
   
   
  #### Out of band communication
   If you can cause the server to communicate with you in some other way ( such as making it look up a domain name that you control) you can determine vulnerabilities this way.
   
  This is also useful because it can be used to detect 2nd or 3rd order injection attacks where the first time your input is used it doesn't cause any problems, but later on in the code it does. By keeping track what url you give what website and where and by keeping good DNS logs you can detect an injection that worked months after you gave the input. See [[Payloads#DNS]] for some possible payloads
 
 
 

 ![[Quiries]]
 

 
 ![[Payloads]]
 
 
 
