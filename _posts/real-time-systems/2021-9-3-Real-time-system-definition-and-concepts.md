---
layout: post
title: Real Time System Definition
status: done
type: post
published: true
date: '21/9/3 17:00 '
---

First set of notes for real time systems class at Iowa State

### Real Time System Definition
  1. For most systems, getting the correct result is the most important thing, however in a real time system **the timleness** of the results is also very important.
  2. If the system misses the deadline the result may no longer matter
  3. Often these systems are done under bad/constrained enviroments. For example, low power, radiation, shacking/external mechanical force


**Best effort system**
- this was what he had at the begining and it did not garantee that something would get done

### Types of real time systems

#### Hard real time systems
- Def: the consequences of missing the deadline is much higher then the reward of reaching the deadline
- avionics
- satalite
- command and control (military)
	
#### firm real time systems
- penalty and reward for reaching the deadline are of the same order of magnitude 
- In this system deadline needs to be met but not catostrofic if it is not
- radar 
- manufacturing
	
#### Soft
- Penalty is order of magantude(s) less then the reward for reaching the deadline
- in this system not huge consiquences if the deadline is not met
- video conferencing
- multimedia
- networking
- Often the system can ignore or repeate the operation easly
	
There are also firm and soft deadlines. In a firm deadline there is no benifit to completing the task if the deadline has pasted. In a soft deadline there can be some reward even after the deadline has pased.


## Example with a car
A breakdown of theparts using an example of a car:
- **Mission:** reach the destination safely
- **Control system:** Car
- **Controling sytem:** human or self driving computer
	- **sesors:** human senses or camera/lidar/etc
- **Controls:** breaks, stearing, accelerator
- Tasks:
	- **critical** stearing/breaking (posibly accelerating)
		- Must be done for safty reasons
	- **Non critical:** turn on radio, show time, etc
- **Preformace:** this is not absolute, there are many different outcomes where some are better then others. For example getting to the destination on time is the best, but slowing down due to ice and getting there a little late is still a good outcome.
- **Cost:** For example we want to maintain the veihical and reduce fuel consumtion. In other systems this could be battery drain.
- **Reliability:** Fault tollerance is extreamly important. interferance in sensors or physical enviroment, power fluctuations, bad inputs should not cause a complete failure and if nessary it should fail in a safe and predictiable way


### Dynamic and static real time systems
- in the static real time system we have a very good understanding of what will be happening in the enviroment and the workload
- For dynamic, there is a lot of uncertainty in the environment and what the workload is going to be doing

### Workloads of Real Time Systems
#### Periodic
- Time based and we know that they will happen before time
- Task T_i is characterized by (c_i, p_i) 
	- where c is computation time and p is the period that it reoccures on
- In this case the deadline would be defore the next task arrives

#### Aperiodic 
 - Event driven, We do not know how many/when we will need to do these
 - Task T_i is charactered by (a_i, r_i, c_i,d_i)
 - where a is the arrival time, r is the time it is ready, c is the computation time, and d is the deadline

#### Sporadic 
- In this case we  do not know when the next task will arrive/if it will arrive. However, we know that there will be some break between the current and next task. This gives a worst case of when a task will arrive
-Note: the c_i is the worst case senerio and it may be possible that the process will end before then

#### Task constrants
 - **Deadlines**
 - **Resources (mutual exclusion)**
   - For example only one task can access a buffer at a time
   - Note: only one task can write to a buffer at a time and buffer can not be read at the same time
##### Precedence
- some task must happen first to allow other graphs to happen
- normally shown in directed precedence graphs
- denoted T1 -> T2 for task one must happen before task two
##### Fault tolerant Requirments
- redundancy is needed to increase reliabilty
- **Most tasks are defined by their combination of requirments, workloads, and payoffs**


### Predictibility
- It is incredbly important for real time systems to have predictiable outcomes
 - Static systems should be designed to give 100% garantees
 - However, when dynamic tasks are input, the garantee can not be given because there can be overload of tasks
 - Definition: as soon as a system accepts a task, it should garantee the task will be completed as long as assumptions made at acceptance hold
	 - These assumptions can included expected faults
 - **Graceful degradion of outcomes**
	 - Real time systems should have many backup plans in case tasks fail, this way safty critical systems do not fail into the worst posible outcome
	 - For example, if a plane can not fly then it should land at airpoirt, if that's not possible, land in road, if not then water ect. It should continue to try and mitigate the damage
 
 ### Common Misconceptions with Real Time Systems
 - Real time systems are not the same as fast computing
 - It is not low level asm programing focused on interupts and device drivers
 - they do not operate in static enviroments
 - The problems in real time system have all been solved in other areas



v3
