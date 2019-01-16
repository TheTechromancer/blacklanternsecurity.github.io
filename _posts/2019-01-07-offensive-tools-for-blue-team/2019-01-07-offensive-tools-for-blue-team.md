---
title: Leveraging Offensive Tools to Help the Blue Team
layout: post
author: jem
description: How to Assess Risk with Zmap and CrackMapExec
---


As attackers on a red-team engagement, we have a multitude of tools at our disposal.  Whether scanning a system or extracting passwords from memory, there's almost always a perfect script or program for the job.  The defensive potential of these resources, however, is often overlooked.  Many offensive tools can be used to provide valuable data for the Blue Team in addition to compromising systems.

Utilizing attack tools on the blue-team side isn't unheard-of.  In his book, _Offensive Countermeasures: The Art of Active Defense_, [John Strand](https://twitter.com/strandjs) shows how every-day utilities and hacking tools can be used to frustrate and deter attackers.  [BloodHound](https://github.com/BloodHoundAD/BloodHound) is another excellent example of an offensive tool which provides invaluable data for defenders by visualizing privilege escalation paths in Active Directory.

I've recently encountered several situations in which the "unorthodox" use of security tools has made my job significantly easier.  Specifically, automating tasks associated with a [Risk Assessment](/cyberassessments/) ([NIST 800-30](https://csrc.nist.gov/csrc/media/publications/sp/800-30/rev-1/final/documents/sp800-30-rev1-ipd.pdf)).  Risk assessments tend to be more comprehensive than a standard adversarial engagement, since they require a thorough analysis of each aspect of the business.  This includes compiling an internal asset inventory list and enumerating host-based controls on endpoints.  As I will explain, this happens to be something which pentesting tools are quite good at!


![what-if-i-told-you][what-if]

In this article, I will demonstrate how to perform the following tasks for an organization, using only freely-available tools:
1. Create an internal asset inventory list
2. Enumerate host-based controls on Windows systems


Please keep in mind that although we are striving to find as many hosts as possible, this data will not be exhaustive.  Systems which are offline or unreachable during the scans will not be included in the results, so our asset inventory will reflect a "snapshot" of the network at the time of the scan.  The goal here is to take a ___representative sample___ of Windows systems on the network and analyze their security posture.


## Step 1: Preparation

Before we get started, you'll need the following:
1. A Linux OS with:
	* CrackMapExec
	* Python 3.7
	* Zmap
	* Root access
2. A privileged network port which can (preferably) reach all subnets on ICMP and 445/TCP.
3. Domain admin (or local admin on the target systems)



## Step 2: Scan All The Things With Zmap

__The Automated Way:__

We'll leverage a Python script (written here at Black Lantern Security) to perform a Zmap scan and output a pretty asset list along the way.

The script will perform an initial ping scan of the entire private IP space.  Additional port scans can then be run on demand, only targetting active hosts.  At the default rate of 1Mbps, the sweep should take less than three hours to finish.

1. Run asset inventory Python script
	~~~bash
	$ git clone github.com/blacklanternsecurity/zmap-asset-inventory
	$ cd zmap-asset-inventory
	$ sudo ./zmap_asset_inventory.py -p 445
	...
	[+] Port scan results for 445/TCP written to /home/user/.asset_inventory/cache/zmap_port_445_2019-01-01_00-19-28.txt
	...
	~~~
2. Hosts with port 445 open are written to a text file as specified in the output:
	~~~bash
	$ head -n 10 /home/user/.asset_inventory/cache/zmap_port_445*.txt
	172.18.36.165
	172.18.124.83
	172.18.125.70
	172.18.129.245
	172.18.204.22
	172.18.204.62
	172.18.204.82
	172.18.204.159
	172.18.204.182
	172.18.204.229
	~~~
3. In addition, an asset_inventory.csv is generated which includes IP addresses, hostnames, and open ports:

	![zmap-example-csv][zmap-example-csv]


__The Manual Way:__

1. Create a blacklist file (blank is okay, but do include any critical systems/subnets that may be sensitive to Zmap or CrackMapExec)
	~~~bash
	$ >blacklist.txt
	~~~
2. Run zmap
	~~~bash
	$ sudo zmap -p 445 -b blacklist.txt -B 1M 10.0.0.0/8 192.168.0.0/16 172.16.0.0/12 | tee 445_hosts.txt
	~~~


## Step 3: Gather Data on Host-Based Controls
***
__"With great power comes great responsibility."__
Let's do something nice with Domain Admin, for a change. :)

***

We will now enumerate host-based controls for each of the systems found in the previous step.  In this example, we're looking for the following services:
* Malwarebytes
* Symantec Endpoint Protection (SEP)
* Symantec Management Agent (Altiris)

Using CrackMapExec, we'll capture the output of a shell command for each Windows system.  `sc query` will be used to list all Windows services, then filtered with `findstr` to output only the services we're interested in.

For example, `sc query | findstr /i "symantec malwarebytes"` might return the following output:

![sc-query-example-output][sc-query-example-output]

Of course, be sure to change the keywords to match the services that you're looking for.  Err on the side of collecting too much data rather than not enough, since unwanted output will be stripped out in the next step.  Also keep in mind that the credentials in use here will need local admin privileges on the target systems in order for the command to execute.

~~~bash
$ sudo cme smb 445_hosts.txt -u <username> -p <password> -x ‘sc query | findstr /i “symantec malwarebytes”’ | tee cme_output_1.txt
~~~

You should see service names displayed in the output:

![cme-example-output][cme-example-output]

__NOTE: You may need to run CrackMapExec multiple times.__
Even the stable version of CrackMapExec is a relatively unwieldy and often hangs before completion.  No disrespect to its author [byt3bl33d3r](https://twitter.com/byt3bl33d3r) of course; the tool is fantastic but suffers from instability due to the scope of its functionality and the complexity of the SMB protocol.
__I recommend running it twice in opposite directions.__  Basically, if you see excessive Python errors or it fails to finish, press CTRL+C to stop the scan.  Then reverse the targets file and start it again:

~~~bash
$ tac 445_hosts.txt > 445_hosts_reversed.txt
$ sudo cme smb 445_hosts_reversed.txt -u <username> -p <password> -x ‘sc query | findstr /i “symantec malwarebytes”’ | tee cme_output_2.txt
~~~

This results in fairly good coverage.  It's okay if we have some overlap, since the results are automatically deduplicated in the next step.  Again, we're not looking for a comprehensive list, only a _representative sample_.


## Step 4: Parse Gathered Data

Parsing the CrackMapExec output could be a bit of a pain.  Thankfully, we've already done the hard work of writing a parsing tool.  Our script will output pretty percentages for each service, broken down by workstations and servers.  In addition, it can output to CSV for host-specific results.

1. Edit the __services.json__ file to reflect the software you're looking for.  Here's an example:
	~~~json
	{
	    "_friendly_name_":  "case-insensitive Windows service name",
	    "Symantec":         "SYMANTEC ENDPOINT PROTECTION",
	    "Altiris":          "SYMANTEC MANAGEMENT AGENT",
	    "Malwarebytes":     "MALWAREBYTES"
	}
	~~~

2. Run the parsing script, passing in the CrackMapExec output from the previous step:
	~~~bash
	$ git clone github.com/blacklanternsecurity.com/parse-crackmapexec
	$ cd parse-crackmapexec/
	$ ./parse_cme.py -w host_based_controls.csv cme_output*.txt

	Hosts Analyzed:   652

	     Service             Total               Workstations        Servers             Undetermined        
	=========================================================================================================
	     Symantec            624/652 (95.7%)     536/557 (96.2%)     16/19 (84.2%)       72/76 (94.7%)
	     Altiris             554/652 (85.0%)     533/557 (95.7%)     13/19 (68.4%)       8/76 (10.5%)
	     Malwarebytes        9/652 (1.4%)        3/557 (0.5%)        6/19 (31.6%)        0/76 (0.0%)
	~~~

3. Here's an example of the CSV contents:
	
	![cme-example-csv][cme-example-csv]

***
## In Summary

Perhaps Zmap and CrackMapExec aren't the most elegant solutions, but they certainly are functional tools that can provide insightful data for Blue Team.  __The potential of attack tools often extends beyond their intended use-case.__  All that is needed is a little abstraction to interpret the output into actionable data.


[what-if]: https://i.imgur.com/3hZac4L.jpg "CrackMapExec is for Sysadmins?"
[sc-query-example-output]: https://i.imgur.com/WOVmadW.png "Example output from sc query"
[zmap-example-csv]: https://i.imgur.com/KfsTsDw.png "Example CSV Output from Zmap"
[cme-example-output]: https://i.imgur.com/e9VRUT8.png "Example Output from CrackMapExec"
[cme-example-csv]: https://i.imgur.com/cWt7Zkj.png "Example CSV Output from CrackMapExec"