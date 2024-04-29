---  
author:  
 name: "spook"  
date: 2022-06-06  
linktitle:  CRTO review
type:  
- post  
- posts  
title:  CRTO review
weight: 10  
series:  
- reviews 
---


# intro 
I recently passed the [Red Team Ops](https://training.zeropointsecurity.co.uk/courses/red-team-ops) course by Zero-Point Security. I picked up the course while it was on sale. It seemed like an affordable course that would build upon the knowledge learned while acquiring my OSCP. I completed the exam recently and thought writing a brief review would be nice. Red Team Ops is a course by Zero-Point Security that focuses on teaching "the basic principles, tools and techniques, that are synonymous with red teaming." The course teaches intermediate level penetration testers the skills needed to engage in red team operations.  
# course

The course is split up into several modules. The modules run through the steps one would engage in during a red team engagement. It begins with planning the engagement and ends with discussing reporting. The bulk of the modules focuses primarily on hands-on instructions on how to perform various techniques of red team tradecraft. The modules focus on explaining how to perform these actions using Cobalt Strike. 

The course instructions are concise and to the point. Some modules also have an accompanying video that shows the attack being taken place. The course is best digested in the order presented as sometimes modules rely on work done in previous modules. The course's focus on practical knowledge can leave one wanting more fundamental explanations of how certain technologies work. The course does offer explanations on how the underlying technology works occasionally. For example, the Kerberos module began with a brief overview of how Kerberos worked. The course is updated over time.

All techniques taught in the course can be and should be reproduced in the lab environment. The lab environment allows one to practice every exploit technique taught in the course. The lab is hosted via SnapLabs. Students purchase hours of lab time for a fairly reasonable rate. The lab experience was enjoyable overall. It is a private environment. The primary benefit of this situation is one does not have to worry about other students weakening the security of a machine or rendering it unexploitable. The lab uses guacamole to manage the remote environment. 

The primary downside to this setup is the inability to introduce external tools into the environment. The ability to VPN into the lab environment would be nice. I assume this cannot be done for reasons related to Cobalt Strike licensing. The attacker machine is preloaded with all the tools needed for techniques taught in the course. You can copy and paste things into the virtual machines. This feature allows you to introduce simple scripts into the environment if you wish.

It can be tempting to simply copy and paste commands from the course into the lab environment and assume one understands the content since the course material is effectively a guided walkthrough of the lab. I would recommend resisting this urge if possible. I feel the lab would be improved if there was some portion of the lab that could be approached blind.

# exam 
The exam is a CTF-style event where one has to capture six out of eight flags. The exam lasts four days. The exam is not proctored. The exam environment is similar to the lab environment. You can copy and paste scripts into the environment but otherwise cannot use external tools. One can stop the environment at any time. Reverting all the machines can also be done. You are provided 48 hours of hands-on keyboard time in the exam environment. The time given to complete the exam is extremely generous. The course was good preparation for the exam. All techniques needed to pass the exam are taught within the course. I felt the exam was fair. 

# conclusion
The course was overall enjoyable. The ability to cheaply and legally gain some hands-on experience with Cobalt strike is enough of a reason to buy the course. The exam experience was fairly relaxed and fun. I recommend it to everyone who wishes to learn more about red teaming. 