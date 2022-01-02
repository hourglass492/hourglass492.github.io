---
layout: post
title: Injection attacks
status: in progress
type: post
published: false
comments: true
date: 2022-01-01
---
### SQL injection

```sql
');  SELECT colum names FROM messages;--
```

In the above the '); ends the previous command

SELECT column names gives the possible column that you want to display

FROM messages says what tables you want to pull from

### Command injection

```bash
; command ; 
```

Start and end with the ; to end the previous command and isolate the inside one.

The command may be then be excited
