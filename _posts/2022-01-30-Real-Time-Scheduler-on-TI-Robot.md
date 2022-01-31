---
layout: post
title: Real Time Scheduler on TI Robot
status: Done
type: post
published: true
comments: true
date: 2021-11-30
---


Abstract— The ability to run multiple jobs on a single Micro
Controller is vital to get the full utilization out of components.
However having many jobs running on a single controller creates
problems of when each job receives processor time and how each
job can be scheduled to ensure every job meets it’s deadline.
RMS (Rate-monotonic scheduling) and EDF (Earliest Deadline
First) scheduling is one method to insure this. This paper takes
an in depth look at an implementation method for these two
schedulers on the MSP432P401R REV D Micro Controller and
considers different practice trade offs that were made in the
design process. It will then work through the testing of the
schedulers to insure that they are working properly and can
handle high loads.

[Full PDF](https://raw.githubusercontent.com/hourglass492/hourglass492.github.io/master/_pdfs/Final-Report-redacted.pdf)
