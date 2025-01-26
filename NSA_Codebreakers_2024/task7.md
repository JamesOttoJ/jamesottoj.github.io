---
layout: default
title: Task 7
parent: NSA Codebreakers 2024
nav_order: 8
---

# TASK 7
{: .no_toc}
- TOC
{:toc}

### Task Description
>  So the DNS server is an encrypted tunnel. The working hypothesis is the firmware modifications leak the GPS location of each JCTV to the APT infrastructure via DNS requests. The GA team has been hard at work reverse engineering the modified firmware and ran an offline simulation to collect the DNS requests.
> 
> The server receiving this data is accessible and hosted on a platform Cyber Command can legally target. You remember Faruq graduated from Navy ROTC and is now working at Cyber Command in a Cyber National Mission Team. His team has been authorized to target the server, but they don't have an exploit that will accomplish the task.
> 
> Fortunately, you already have experience finding vulnerabilities and this final Co-op tour is in the NSA Vulnerability Research Center where you work with a team of expert Capabilities Development Specialists. Help NSA find a vulnerability that can be used to lessen the impact of this devastating breach! Don't let DIRNSA down!
> 
> You have TWO outcomes to achieve with your exploit:
> 
>     All historic GPS coordinates for all JCTVs must be overwritten or removed.
>     After your exploit completes, the APT cannot store the new location of any hacked JCTVs.
> 
> The scope and scale of the operation that was uncovered suggests that all hacked JCTVs have been leaking their locations for some time. Luckily, no new JCTVs should be compromised before the upcoming Cyber Command operation.
> 
> Cyber Command has created a custom exploit framework for this operation. You can use the prototype "thrower.py" to test your exploit locally.
> 
> Submit an exploit program (the input file for the thrower) that can be used immediately by Cyber Command. 

### Files Given
- prototype exploit thrower (thrower.py)

### TBD