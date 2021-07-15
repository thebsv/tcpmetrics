# tcpmetrics

## Problem Description

New Connection Detection: The definition of new connection is a unique(srcIP:srcPort -> dstIP:dstPort), meaning if one of these four entities are varied, then it counts as a new connection.

Port Scan Detection: A port scan is defined as an anomaly which occurs for connections with a unique(srcIP -> dstIP) (key), where dstPort has over three values. There is no constraint over the dstPort numbers, there needs to be more than 3 destination port numbers for each key to be considered as a port scan.

## Solution Design

The file /proc/net/tcp may be overwritten by the kernel while it is being read, so its best to not
glob the entire file, and read it line by line instead. Reads to this file are atomic per line.
(source: https://stackoverflow.com/questions/5713451/is-it-safe-to-parse-a-proc-file ). Therefore, I opened the file in read only mode and parsed it line by line using bufio. 

A file parser package was written to read the file and export the contents as a 2D array of strings, where
each element is a field between blank spaces in the row, and each row is represented by a 1D array of strings.

A connection scanner object is used to detect new connections. This object dependson the tokens generated by file parser. The connection scanner function combs through the file and parses out the `ip:port` pairs in human readable format, and stores them in a set. The set is represented by a hashmap with a string key field and bool value, where the format of the string is `"srcIP:srcPort -> dstIP:dstPort"`.

The connection scanner object contains a function to detect port scans. The port scanner accepts a list of tokens and parses `unique(srcIP -> dstIP)` (key) fields and stores them in a hashmap with a string key field, and string value field with the value field being a csv of destination port numbers. While collecting the destination port numbers, I ensured deduplication using a hashmap. The number of elements in this csv are counted to ensure that we send only those entries which have over three elements in this value field. The output of this function is a `hashmap[srcIP -> dstIP] = dstPorts`

## Testcase Specifications and Details

File Parser
- Permutation 1: parse the example file mentioned in the doc
- Permutation 2: use a negative example, and see if it fails appropriately

ConnectionScanner
- Permutation 1: input is a 2D string which is a tokenized verison of the file, output is a connection map
- Permutation 2: input 2D tokenized version of a file with a port scan, and it detects the port scan


## Questions

Level 1

1. How would you prove the code is correct?
Using unit tests (package level tests) and integration tests ( end to end, like using the file test1 in the root folder). Additionally, I used an Ubuntu VM to test the software in the field using netcat and nmap to simulate a port scan. Therefore, testing the software in the customer's environment, or a very accurate simulation of the same is very important.

2. How would you make this solution better?

Ideally this file should be parsed using a parsing library, by creating structs with the appropriate grammar.

Convert IP function could be formalized to output better errors, and carry the IP and port in a struct, instead of
just using a string. Variable naming can be improved inside helper functions.

Numbers in the convertIP and convertPort function could be written as constants.

3. Is it possible for this program to miss a connection?

Yes, if the resolution of the connections / port scans are smaller than 10 seconds, then the program will miss the connections / port scan. It is really important to understand that time resolution / time windows matter. A continuous monitoring solution is not truly continuous, since it is still scheduled by the kernel to run at specific cpu time windows, along with the OS itself.

4. If you weren't following these requirements, how would you solve the problem of logging every new connection?

There are a couple of ways I know of to solve this problem

a) Using Snort

Although I haven't used it, I have observed people using the Network Intrusion Detection Tool Snort to monitor the network and log new connections according to specified altering rules, and the config could include `log tcp any any... `, and this could be configured to log only headers, and would need to be deduplicated.

`./snort -dv -l ./log 0.0.0.0 -c snort.conf`

Then this data could be logged to a database.

b) Configuring a router to send Netflow data

I did research on this topic during undergrad, and I was able to configure a Cisco router running on GNS3 to send flow data to a machine using UDP, and parse the packets using Perl automation running on a linux box (tap), and logged on the local fs. A suitable database could be used to do this at scale with many taps running over something like a large network.

(source: https://www.researchgate.net/publication/258790178_Usage_of_Netflow_in_Security_and_Monitoring_of_Computer_Networks)

Level 2
1. Why did you choose `x` to write the build automation?

Golang has extensive support and tooling around the language itself in addition to being clear and consice. It is the appropriate tool for the job here since it is neither a very low level language like C, nor at a higher level like python. Golang is an excellent fit for this sort of application.

2. Is there anything else you would test if you had more time?

A db / in memory db may need to be used while doing this on a server which receives a large number of connections,
since data over six cycles of this program may get quite large.

The port scan detection function works for a tuple(srcIP, dstIP) as unique keys and checks for multiple dstPort hits,
this does not take factor in possible source IP randomization / spoofing.

Testing was not done rigorously, would need to include negative cases, improper values, same dstPort and dstPort (invaild combinations and logical errors).


3. What is the most important tool, script, or technique you have for solving problems in production? Explain why this tool/script/technique is the most important.

tshark, netstat, and tcpdump are all tools that deal with socket level connections and can be used for debugging.
- tshark: continuous monitoring of a specified network interface on the host and the ability to author flags and different filtering parameters to capture network traffic correctly

- netstat: reads /proc/net/* protocol files and displays currently established connections, listening ports, and a lot of other details across all network interfaces on the host

- tcpdump: continually monitors a specified network interface to print out TCP/IP packet details and can be useful for monitoring TCP/IP packets sent by different applications on a host


