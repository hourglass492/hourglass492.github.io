---
layout: post
tags: Real Time Systems
title: Rate Monomus Scheduling and Earlisist Deadline First
status: done
type: post
published: true
date: '21/9/10 17:00 '
---



## RMS 
 - Task with the lowest task has the highest priority
 - For offline schedule making, construct offline schedule up to the LCM
 - Test we have is sufficient but not necessary, This means that if it succeeds then we don't have to worry
 - Test:

![Figure 1.](/_posts/real-time-systems/RMS-Test-Image.png)

 where U is comp time / period
  - RMS is optimal for **fixed priority**, however EDF and LLF are better for dynamic priorities


## Earliest Deadline First EDF
 - This is has a test of:

![Figure 2.](/_posts/real-time-systems/EDF-Test-image.png)

 - This test is necessary and sufficient
 - Priority based scheduling
 - Task with the smallest/closest deadline has the highest Priority
 - At any time the highest Priority task is executed
 - EDF Can have tasks with deadlines that are not the start of their next period


## Least Laxity First
 - Has the same test as EDF
 - This is has a test of:

![Figure 2.](/_posts/real-time-systems/EDF-Test-image.png)

 - Schedules based on laxity rather then deadline
	 - Laxity is how much spare time before the process has to be completed
	 - Takes into account deadline and how long it will take to complete
 - This can cause thrashing behavior by switching back and forth between tasks
	 - Caused by two tasks with same/similar laxity because they will continually switch back and forth
 - Also has a large amount of overhead in calculating priority 



## RMS vs EDF
 - RMS is useful because it has as rich in theory and is easy to analyze
 - EDF is harder to implement 
 - EDF is more responsive and handles a-periodic tasks
 - can have higher optimization



## Exact analysis
 - If all tasks start at the same time and a particular task (i) is able to meet its first deadline, then we know that task will always meet its deadline
	 - We know this because all other tasks with higher priority will complete first and then the task will execute. Because this is the worst case scenario, we can know what will happen




## Deadline mono-processor systems
 - This is a generalization of RMS, in this system the deadline does not have to be the same as the next arrival period
 - fixed priority system
 - Rather then order based on smallest period, orders on smallest deadline, allows you to have deadlines that are not the same as the period

