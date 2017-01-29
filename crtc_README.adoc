// vim:set ft=asciidoc syntax=ON cc=80:
= crtc - a generic-purposed expect script to automate interactive tasks
:expectnote:
:workflow!:
:doctype: book
//this is to generate a right side toc
:toc: right
//below two lines make it work for current (2016-03-09) github(asciidoc 1.5.2?)
//:toc: manual
//:toc-placement: preamble
:toclevels: 3
:toc-title: Table of Content
:numbered:
:iconsdir: 
:icons: font
:source-highlighter: prettify
:source-highlighter: highlightjs
:source-highlighter: pygments
:source-highlighter: coderay
:data-uri:
:allow-uri-read:
//:hardbreaks:
:last-update-label!:
//:nofooter:
:Author:  Ping Song
:Author Initials: SP
:Date:   Aug 2015
:Email:   pings@juniper.net
:title: crtc
:experimental:
:stylesheetdir: {user-home}/Dropbox/asciidoctor-stylesheet-factory/stylesheets/
:stylesheet: {stylesheetdir}maker.css
:stylesheet: {stylesheetdir}readthedocs.css
:stylesheet: {stylesheetdir}github.css
:stylesheet: {stylesheetdir}iconic.css
:stylesheet: {stylesheetdir}rubygems.css
:stylesheet: {stylesheetdir}foundation.css
:stylesheet: {stylesheetdir}foundation-potion.css
:stylesheet: {stylesheetdir}foundation-lime.css
:stylesheet: {stylesheetdir}rocket-panda.css
:stylesheet: {stylesheetdir}riak.css
:stylesheet: {stylesheetdir}asciidoctor.css
:stylesheet: {stylesheetdir}colony.css
:stylesheet: {stylesheetdir}golo.css


`crtc` is an `expect` script that can be used to automate the interactive tasks
in a general manner:

* login to remote device(s)
* sending commands repeatedly
* issue/log/event monitoring
* monitor user-defined issues and execute user-defined actions
* shell friendly
* command (and output) timestamping 
* logging 
* misc: 
  - anti-idle (keep alive)
  - switch the session in background/foreground, 
  - online helping, 
  - email notification, 
  - etc 

      ____ ____ _____ ____
     / ___|  _ \_   _/ ___|
    | |   | |_) || || |
    | |___|  _ < | || |___
     \____|_| \_\|_| \____|

== quick examples

****
The script was created soly in "part-time" and just worked for my own usage
case , the usage may not be fully tested yet due to the very limited time that
I can find for this "project", therefore it's very likely that a lot of
functions may be still not fully working as expected here and there. But most
of the usage examples listed in this doc were extracted from logs of my
production work , so at least these are working fine when I recorded them :). I
have tested the main part of the code and they are basically working, and I'm
feeling they are even quite extendable that new function can be added easily,
also small issues can be fixed relatively easy on demand. Just send me your
improvement request and issue feedback.
****

NOTE: For Juniper folks without a machine with "Expect" tool installed, any of
the svl-jtac servers (e.g. `svl-jtac-tool01`) will be sufficient. Try (copy and
paste) following examples and have a quick overview of what it can do for you.

=== automate device login process

[[LOGIN1]]
==== configure login info for individual device

* login to a new device named "myrouter"
+
put this below login entry into `~/crtc.conf`:

    #set domain_suffix_con jtac-west.jnpr.net
    #set domain_suffix jtac-east.jnpr.net
    set login "labroot"
    set password "lab123"

    #test only
    set login_info(myrouter)       [list                \
        "$"            "telnet 172.19.161.101"          \
        "login: "      $login                           \
        "Password:"    $password                        \
        ">"            "set cli screen-width 300"       \
        ">"            "set cli timestamp"              \
    ]
+
then login to the device:

    ~pings/bin/crtc myrouter


==== configure login info for a group of device

* all devices under same "group" shares the same login steps.
  a `group` is indicated by appending a `@GROUPNAME` string after the device
  name.  e.g.: all devices under suffix `@jtac` shares the same login steps.
+
put this below login entry into `~/crtc.conf`:

    set domain_east jtac-east.jnpr.net
    set domain_west jtac-west.jnpr.net
    set login_string "telnet $session.$domain_east" 

    set login_info($session@jtac)       [list       \
        "\\\$"        "$login_string"    \
        "login: "      "$jtaclab_login"             \
        "Password:"    "$jtaclab_pass"              \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]
+
then login to the device:

    ~pings/bin/crtc alecto@jtac

=== host name resolution

* having these in crtc.conf:

    array set hostmap {\
         pe5        172.19.161.107 \
         pe6        192.168.43.17\
    }

    set login_info($session)       [list       \
        "\\\$"        "$host"    \
        "login: "      "labroot"             \
        "Password:"    "lab123"              \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]
+
the "session" name `pe5` will be resolved to a "host" IP.
now you can login pe5 with:

    crtc pe5

* parsing the user-defined host string for login info
+
--
with below configuration, script first resolve session name "alecot-re0-con"
into string `b6tsb17:7021`, then parse it to get terminal server name and
telnet port. this is how the script login to the "console" of a router:

    array set hostmap {\
        alecto-re0-con	 b6tsb17:7021	\
        alecto-re1-con	 b6tsb17:7022	\
        havlar-re0-con	 b6tsb17:7034	\
    }

    if [regexp {^(\w+):(\d+)} $host -> hostname port] {

        #b6tsb25:7028	
        set fullname "$hostname.$domain_west"
        set login_string "telnet $fullname $port"

    }

    set login_info($session@jtac)       [list       \
        "\\\$"        "$login_string"    \
        "login: "      "$jtaclab_login"             \
        "Password:"    "$jtaclab_pass"              \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]

login to console port of router alecto:

    ~pings/bin/crtc alecto-re0-con@jtac
--

=== automate commands after login

* run a command (-c) 10 times (-n), with interval of 2 seconds (-i), don’t
  display (hide) the login steps (-H), and quit the session when all done (-q)

    ~pings/bin/crtc -c "show chassis alarms" -H -q -n 10 -i 2 myrouter
+
group all options to make it a little bit shorter:

  ~pings/bin/crtc -Hqi2n10c "show chassis alarms" myrouter

* flap a port 3 times with 10s interval, in each iteration: shutdown and
  wait 3 seconds, then bring up (rollback)

    ~pings/bin/crtc -b "configure" \
      -c "set interfaces xe-3/1/0 disable" \
      -c "commit" -c "SLEEP 3" -c "rollback 1" \
      -c "show | compare" -c "commit" \
      -B "exit" -n 3 -i 10 myrouter
+
or, dropping into shell and use shell command to do the same:

    crtc -E ">" -S "start shell" -E "%" \
      -S "su" -E "sword" -S "Juniper" \
      -c "ifconfig xe-3/1/0 down"  \
      -c "SLEEP 3"                 \
      -c "ifconfig xe-3/1/0 up"    \
      -n 3 -i 10 -g 5 myrouter
+
-g is interval between each command during the same iteration.
+
this maybe look too long, the same thing can be done with following options
configured in `~/crtc.conf` footnote:[see <<X3,config file>> for other
alternative way to do the same thing.].

  set pre_cmds1(myrouter) configure
  set cmds1(myrouter) {
      "set interfaces xe-3/1/0 disable"
      commit "SLEEP 3" "rollback 1"
      "show | compare" commit
  }
  set post_cmds1(myrouter) exit
+
then just run crtc without options:

  ~pings/bin/crtc -n3j myrouter
+
here -j is to specify a "project", which simply maps to the number of cmds
array that holds the actual commands to be executed. in this case we want to
execute commands in cmds1, to "-j" or "-j 1" will do it.

==== interupting an automation loop

Instead of just awaiting for sleep time to expire, `crtc` provides more options
for the user to interact with the script, even during the sleep time:

* press `+` to increase sleep interval by 2s
* press `-` to decrease sleep interval by 2s
* press <space>, <ENTER> , or any other key to skip the current sleep time and
  start next iteration right away.
* press `q` to "pause" the iteration and return the session control to the user,
  who can continue to type commands, and :
  - press `!R` to resume the iteration from wherever left off (before the
    iteration was interupted by pressing `q`)
  - press `!s` to stop the automation
  - press `!r` to start the automation all over again

This commands help a user to have better control in a session to
start/pause/resume/restart a pre-defined automation.

==== quick mode: login, send cmds, and exit

`-q option`:

this is useful for quick test in shell script, or you just need to get some
quick data in one shot. Imaging that in the middle of your shell script you
need some (realtime) data from the router, calling this script you can login to
a router, send the commands and grab the data, after that you need the crtc to
quit so the original shell script can continue from where it left off ... the
`-q` option is used to just "quit-after-done".

.config

this is equivalent to having this settings in config file:

    set nointeract 1

.login with `-q`
===============================================

    ping@ubuntu1404:~/bin$ crtc -c "show system uptime" -H -q alecto
    current log file ~/logs/alecto.log
    set cli timestamp

    Dec 01 13:14:24
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> bye!:)
    ping@ubuntu1404:~/bin$

===============================================

TIP: later I'll provide examples about how to use crtc in your shell script.


=== automate commands with user defined patterns

adding below patterns in `crtc.conf`:

    set user_patterns(pattern_not_resolve_msg)              [list "ould not resolve"]
    set user_patterns(pattern_connection_unable)            [list "telnet: Unable to connect to remote host"]
    set user_patterns(pattern_console_msg)                  [list "Type the hot key to suspend the connection: <CTRL>Z"]
    #set user_patterns(pattern_connection_not_responding)    [list {Timeout, server (\\d{1,3}\\.){3}\\d{1,3} not responding}]
    set user_patterns(pattern_connection_not_responding)    [list {Timeout, server (\d{1,3}\.){3}\d{1,3} not responding}]
    set user_patterns(pattern_broken_pipe)                  [list "Write failed: Broken pipe"]
    set user_patterns(pattern_connection_close_msg)         [list                                                                   \
        {Connection closed.*|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer}  \
        RECONNECT]
    set user_patterns(connection_refused)                   [list "Connection refused" RECONNECT]

    set user_patterns(pattern_gres_not_ready)               [list {Not ready for mastership switch, try after \\d+ secs.*}]

    set user_patterns(answer_yes)                           [list {\(no\)} yes]
    set user_patterns(backup_re_not_running)                [list "error: Backup RE not running" RETRY]
    set user_patterns(backup_re_not_ready)                  [list "error: Backup RE not ready for ISSU" RETRY]
    set user_patterns(nsr_not_active)                       [list "warning: NSR not active" RETRY]
    set user_patterns(issu_aborted)                         [list "error: ISSU Aborted" RETRY]


now to iterate JUNOS "ISSU" 10 time, use this command:

    crtc -n10pc "request system software in-service-upgrade /var/tmp/xxxxx reboot" getafix@jtac

to make it short and less ugly, also add this in `crtc.conf`:

    set issu_image_debug    "junos-install-mx-x86-64-15.1I20160311_0949_abhinavt.tgz" 
    set issu_image_s51      "junos-install-mx-x86-64-15.1F2-S5.1.tgz" 
    set issu_cmd_debug      "request system software in-service-upgrade /var/tmp/$issu_image_debug reboot"
    set issu_cmd_s51        "request system software in-service-upgrade /var/tmp/$issu_image_s51 reboot"

    set cmds1(pe50@attlab) [list   \
        "$issu_cmd_debug"        \
    ]

    #new method:with user_patterns, it can be simplified as below:
    set ISSU(getafix@jtac) [list   \
        "$issu_cmd_s51"        \
    ]

now a JUNOS "ISSU" automation can be done by this better-looking command:

    crtc -n10pc ISSU getafix@jtac

.how it works?

the `-p` indicates a "persistent mode". Under this mode crtc will use one of
the configured user_pattern to detect connection loss:

    set user_patterns(pattern_connection_close_msg)         [list                                                                   \
        {Connection closed.*|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer}  \
        RECONNECT]

once a match is detected, the user configured action will be executed. In this
case it's to "RECONNECT", so crtc will try to reconnect to the router and
continue the rest of the iterations.



=== "special" commands

==== `SLEEP N`

* example of using config file
+
add following in in the configuration file: `~/crtc.conf`

    set cmds1(myrouter)         [list \
        "show system uptime"        \
        "SLEEP 15"                  \
        "show chassis alarm"        \
        "SLEEP 30"                  \
        "show system processes extensive | no-more"        \
    ]
+
run the script:

    ~pings/bin/crtc -n 3 -i 5 myrouter
+
this will make crtc to login the router, and repeat the following command set
3 times:

    - execute "show system uptime"
    - sleep for 15s
    - execute "show chassis alarm"
    - sleep for 30s
    - execute "show system processes extensive | no-more"

* same,but using CLI options without bothering any config file 
+
without bothering to change config file, you can use entirely CLI options
instead. In this example we login to router named "myrouter", send "show system
uptime" 3 times with a 5s interval

    ~pings/bin/crtc -c "show system uptime" -c "SLEEP 15"        \
                    -c "show chassis alarm" -c "SLEEP 30"        \
                    -c "show system processes extensive | no-more" \
                    -n 3 -i 5 myrouter
+
NOTE: you can also mix CLI flags with config file options - leave the login
info in the config file, but use CLI flags to send commands

TIP: if value of seconds is not given, it will sleep 3s.

another example:

    set cmds2(pe50@attlab) [list   \
        ">" "request routing-engine login other-routing-engine" \
        ">" "start shell"   \
        "%" "su"            \
        "sword" "jnpr123"   \
        "#" "SLEEP 5;dtrace -s /var/tmp/chassisd_socket.d -o chassisd_snmpd_%T.log&" \
        "#" "exit"          \
        "%" "exit"          \
        ">" "exit"          \
        ">" "SLEEP 5"       \
        ">" "GRES"              \
    ]

==== GOBACKGROUP

As the name indicates, this command can be used to bring a task to run in
"background". This is similiar to the linux "task management" tools in shell
that is useful when multiple processes (e.g. one per router) need to be run at
the same time. one practical example is shown below:


. first, configure the router login process to login the router and drop into
the shell
+
--

----
set login_info(automatix_shell@jtac)       [list       \        <1>
    "\\\$"      "crtc -d0 automatix@jtac"    \                  <2>
    "automatix@jtac:automation done"    "start shell"        \  <3>
    "%"         "su"                    \                       <4>
    "sword:"    "Juniper"               \                       <4>
    "$"         "uptime"                \                       <4>
]

set login_info(dogmatix_shell@jtac)       [list       \         <1>
    "\\\$"      "crtc dogmatix@jtac"    \                       <2>
    "dogmatix@jtac:automation done"    "start shell"        \   <3>
    "%"         "su"                    \                       <4>
    "sword:"    "Juniper"               \                       <4>
    "$"         "uptime"                \                       <4>
]
----

<1> steps to login to a JUNOS router's shell
<2> use recursive crtc to login to a router
<3> once the child crtc done its job, enter JUNOS shell
<4> login with su

--

. second, config the process to "pre-build" both router sessions, and move them
into background so they can be used as needed later
+
--

----
set login_info(LOCALHOST) [list \
    "\\\$"      "uptime"          \                         <1>
    "\\\$"      "crtc -d0 automatix_shell@jtac" \           <2>
    "automatix_shell@jtac:automation done" "uptime" \       <3>
    "%"    "GOBACKGROUND"  \                                <4>
    "\\\$"      "crtc -d0 dogmatix_shell@jtac" \            <5>
    "dogmatix_shell@jtac:automation done" "uptime" \        <6>
    "%"    "GOBACKGROUND"  \                                <7>
    "\\\$" "jobs"       \                                   <8>
]
----

<1> crtc start with printing a time from *local shell* (where crtc is running)
<2> crtc invokes a "nested" or "child" crtc process `crtc -d0
automatix_shell@jtac`, this will login to the shell of router `automatix`

<3> crtc monitor the progress of the child crtc process, when it's done, run a
`uptime` command to get another timestamp, this time from *the remote router
shell*.

<4> crtc execute "GOBACKGROUND" command, which will send a signal to put the
child crtc process to background. this will release the current terminal back
to crtc, so other tasks can be performed.

<5> once the prompt from original terminal appears, crtc invokes another child
process to login to the shell of another router `dogmatix`

<6> same as in <3>, crtc monitor the progress of the router login process, and
acquire another timestamp from the shell of the new router.

<7> same as in <4>, crtc move this new child crtc process to background

<8> crtc now has 2 child processes, each representing a session into a remote
router, running in background
--

. config the `cmdN` array to do the test on the two routers
+
--

----
set cmds9(LOCALHOST) [list                              \
    "fg 1\r\n"                                          \       <1>
    "uptime"                                            \       <2>
    "ifconfig xe-0/2/0 down;ifconfig xe-0/2/0 up"       \       <3>
    "SLEEP 6"                                           \       <4>
    "ifconfig xe-0/2/0 down;ifconfig xe-0/2/0 up"       \       <5>
    "SLEEP 300"                                           \     <6>
    "GOBACKGROUND"                                      \       <7>
    "fg 2\r\n"                                          \       <8>
    "uptime"                                            \       <9>
    "gzip -f capture1.txt"                              \       <9>
    {cprod -A fpc0 -c "show nhdb summary"}              \       <9>
    {echo "check fpc0" >> capture1.txt}                 \       <9>
    {grep -E "Inactive|Uninstall" capture1.txt | wc -l} \       <9>
    "GOBACKGROUND"                                      \       <10>
]                                                               
----

<1> get router `automatix` session: move the session (first session) foreground
<2> capture a timestamp from this router
<3> flap link `xe-0/2/0` of automatix
<4> sleep 6s
<5> flap same link again
<6> sleep 300s
<7> move the router `automatix` session to background
<8> get router `dogmatix` session: move the session (2nd sesson) foreground
<9> get a timestamp from this router, and perform other test steps
<10> move this router session to background
--

. finally, to iterate the test 10 times:

    ping@ubuntu47-3:~$ crtc -n10 LOCALHOST

==== REPEAT M N

considering a test case when it is required to repeat some commands several
times during one iteration, e.g:

    set cmds1(myrouter) [list                    \
        "GRES"                                   \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
    ]

in this configuration, we need to iterate these test in sequence:

. perform `GRES`
. check if there is any BGP flap
. wait 10s
. repeat previous 2 steps 5 more times

The goal is to iterate this whole process 50 times, and in each iteration we
need to check the number of BGP connections every 10s during at least a minute
long.  the reason we want to "repeat" the check is - so we won't miss a BGP
flap that whould happen to occur randomly in 1 minute after GRES performed, but
then recovered quickly.  repeating the same check on the number of bgp sessions
will ensure that we'll catch the issue whenver it happened. while this works
fine, the config looks ugly - it's even ugly if we need to repeat 200 more
times in other cases.

A better way is to use the `REPEAT` command, so the above config can be
rewritten to:

    set cmds1(myrouter) [list                    \
        "GRES"                     \
        "show bgp summary | match estab | count" \
        "SLEEP 10"                                \
        "REPEAT 2 5"                             \
    ]

the command `REPEAT 2 5`, just means to repeat the previous whatever *2*
commands, *5* more times.

for completeness here are the complete configs (in config file `crtc.conf`) for
this specific test case:

----
set cmds1(myrouter) [list                    \ <1>
    "GRES"                                   \
    "show bgp summary | match estab | count" \
    "SLEEP 10"                                \
    "REPEAT 2 5"                             \
]

set regex_info(myrouter) {                     <2>
    {2@@Count: (\d+) lines@bgpcount}
}
#Count: 0 lines

set issue_info(myrouter) {                     <3>
    {2@bgpcount != 2}
}

set collect(myrouter) [list      \             <4>
    "show bgp summary | no-more" \
    "show log message | match bgp | last 20"
]
----

<1> perform RE switchover, then repeatedly check bgp connection numbers at
least 6 times in a minute, before moving to the next iteration
<2> with a regex, capture the number of established BGP sessions using the
number *2* command ("show bgp ...") , save it to a varible named `bgpcount`
<3> define the "issue" to be that whenever the value of varible `bgpcount` is
*NOT* 2
<4> if the "issue" is "seen", collect some more data

and this is how the `crtc` runs:

    crtc -n50j myrouter       

this will: 

* iterate the same above test procedure 50 times (`-n50`)
* use commands defined in project 1 (array cmds**1**)

==== `GRES`

this command:

    ~pings/bin/crtc -c "GRES" -c "show chassis alarm" -Hqn30i250 myrouter

will do:

* perform JUNOS "RESO" 30 times (-n30) 
* and quit(-q) after all done
* it will wait 250s between each RESO (-i250)
* suppress the detailed output (-H)

skipping `-i250` is fine, "GRES" command has built-in "visibility" to read the
JUNOS output, and delay the next RESO attempt accordingly. see more detail in
section <<GRES, GRES>>

NOTE: 240s is the minimum waiting time between 2 RESO in Juniper router.

==== MONITOR

TODO

=== "projects"

As you keep adding more data and changing the commands for different cases in
the config file `crtc.conf`, it will grow bigger and contains a lot of
different, and often conflicting command groups with each of them serving a
specific test case you ever worked on. use `-j` to specify which command group
you prefer `crtc` to send, here is an example, in `crtc.conf` you have below
configured command groups:

--
* `cmds1`
* `cmds2`
* `cmds6`
--

    set cmds1(myrouter) {
        "show system core"
        "show chassis alarm"
    }

    set cmds2(myrouter) {
        "show version | no-more"
        "show chassis fpc pic-status"
        "show chassis hardware | no-more"
    }

    set cmds6(myrouter) {
        "systeminfo"
        "ospfinfo"
        "bgpinfo"
    }

    set systeminfo(myrouter) {
        "show system uptime"
        "show chassis alarm"
    }

    set ospfinfo(myrouter) {
        "show ospf neighbor"
        "show ospf overview"
    }

    set bgpinfo(myrouter) {
        "show bgp summary | match up"
    }

to execute commands defined in `cmds2`, run this:

    crtc -j2 myrouter

`-j` is used to specific a "project" number, in this case it is *2*, so `cmds2`
will be executed. All other command groups can remain in the config file in
case they are needed for other tests.

=== integration with shell script

==== to pull data out of a device in shell

copy and paste these 2 lines in the shell server:

    ver=`crtc -HWqc "show version | no-more" myrouter | grep -i "base os boot" | awk '{print \$5}'`
    echo "myrouter is running software version: $ver"

+
you'll get:

    myrouter is running software version: [12.3R3-S4.7]
+
this will print the current version that router alecto is running. the `-W`
means "use crtc in shell", so all messages printed by crtc will now be
"pipe-friendly" - without this option some messages will be just printed to the
terminal regardless of whether crtc is running in another shell script or not.

* To pull running version from any junos device
+
put the 2 lines in a file, 
name it `printver.sh`, in jtac server:
+
    #file: printver.sh
    #!/bin/bash
    ver=`~pings/bin/crtc -HWqc "show version | no-more" $1 | grep -i "base os boot" | awk '{print \$5}'`
    echo "router $1 is running software version: $ver"
+
then run the shell script with a router name as the only parameter footnote:[of
course, better run `chmod 755 printver.sh` to make the file executable]. It
will detect and report the current release info from any given Junos router :
+
JTAC lab router:

    [pings@svl-jtac-tool02 ~]$ ./printver.sh tintin@jtac
    router tintin@jtac is running software version: [12.3R3-S4.7]
+
customer router(`@att` needs to be configured in config file):

    [pings@svl-jtac-tool02 ~]$ sh printver.sh DTxxxxVPE@att
    router DTxxxxVPE@att is running software version: [14.1-20141106.0]

==== to "scan" and pull data from all devices within a subnet:

e.g. to pull hardware inventory data from whole subnet 172.19.161.x:

    for i in {2..254}; do crtc -HW5w5qac "show chassis hardware" 172.19.161.$i@jtac; done;

+
if only to pull info from specific IP in the subnet:

    for i in 2 3 5; do crtc -HW5w5qac "show chassis hardware" 172.19.161.$i@jtac; done;
+
or, pull info from a group of seperated IP addresses that are not in same subnet:

    for i in 1.1.1.1 2.2.2.2 3.3.3.3; do crtc -HWw3qac "show chassis hardware" -w 5 $i@jtac; done;

+
--
* the `-A` flag is used to automatically "page" the long output from the
  command, which will otherwise requires: `| no-more`; 
* `-H` hide the login steps, 
* `-q` make the crtc quit right after data collected. without this crtc will go
  into `interact` mode on first session , as a result the shell script will
  stay in first router session and won't proceed to the next router until you
  manually `exit` each session.
* `-w5W5` to wait maximum 3s before timeout current session, this is to make crtc
  exit timely if the remote IP is a "dead" peer, so the shell script will
  proceed anyway.
--
+
NOTE: in practice, the list of IP can go very much longer than these. a
reasonable requests will be to pull data periodically from 50 to 500 routers in
a network, for monitoring purpose. in that case a better practice is to put
this long-one-liner into a file, and execute it as a shell script.

==== work with environment varible

suppose we want to use the login steps configured in `crtc.conf`, like in
<<LOGIN1>> , but we want to change the value of varible `login` and `password`
to sth else.  one way is to just change the value in `crtc.conf`:

    set login "labroot1"
    set password "lab456"

another way is to change it directly from shell:

    ping@ubuntu47-3:~$ export CRTC_login=labroot1
    ping@ubuntu47-3:~$ export CRTC_password=lab456

now crtc will login the router with the changed login and password.

    ping@ubuntu47-3:~$ crtc myrouter

to s**K**ip the environment variable and just force using the original varible
defined in config file, use `-K`:

    ping@ubuntu47-3:~$ crtc -K labrouter@attlab


=== timestamp all your commands

* to timestamp all your unix, Junos shell, PFE command commands

----
lab@mx86-jtac-lab> start shell      
Dec 03 08:20:30                     <1>
% ls -l /var/log/messages           <2>
-rw-rw----  1 root  wheel  4194094 Dec  3 08:21 /var/log/messages

%!timestamp 1 - local timestamp on! <3>

Dec 03 05:27:37 2014(local)
% ls -l /var/log/messages           
Dec 03 05:27:47 2014(local)         <4>
-rw-rw----  1 root  wheel  4195050 Dec  3 08:25 /var/log/messages

root@mx86-jtac-lab% vty 1
Dec 03 05:28:17 2014(local)

BSD platform (Pentium processor, 1536MB memory, 0KB flash)

VMXE(mx86-jtac-lab vty)# show pfe statistics traffic
Dec 03 05:28:35 2014(local)         <5>
PFE Traffic statistics:
                    0 packets input  (0 packets/sec)
                    0 packets output (0 packets/sec)

PFE Local Traffic statistics:
...<snipped>...
----

<1> the last timestamp provided by Junos "set cli timestamp" 
<2> no timestamp provided in native unix shell mode
<3> you press `!t`: and `crtc` will prompt a local timestamp will be provided
for each cmd you typed, also crtc prompted that timestamp option is set
<4> shell commands now got timestamped
<5> pfe commands now got timestamped

* to timestamp e320 shell command

    [pings@svl-jtac-tool02 ~]$ ~pings/bin/crtc -Ht e320-svl
    ...<snipped>...
    slot 16->print__11Ic1Detector       #<------Junos-e vxWorks shell command
    Dec 02 15:25:14 2014(local)         #<------"local" timestamp by crtc

    state=NORMAL
    sysUpTime=0x03B33619
    passiveLoader=0x0C001994
    crashPusherRequested=1
    ...<snippet>...

=== running `crtc` in background

Very often you want to logging into different remote devices to check the
network issue. your current temrinal will be occupied by the first crtc session
after you login into one remote device. In order to to login to the 2nd or even
more devices in the same terminal, there must be a way to "hang up" current
job, start the 2nd session, work on the 2nd session when the 1st session is
still running in the background.

NOTE: other common practices to solve this is to use multiple terminals -
either to open multiple terminal windows or open multiple terminal "tabs" in
one window.

There are multiple options of doing this:

* running `crtc` within `GNU screen` (so you can shutdown your own PC and leave,
  while the session keeps running in the remote server) + first login to a
  server where `GNU screen` is available:

    ssh svl-jtac-tool02
+
then start `crtc` "within" a GNU screen window:

    [pings@svl-jtac-tool02 ~]$ screen -fn -t myrouter ~pings/bin/crtc myrouter
+
now the session to myrouter is "held" by a screen `window` named `myrouter`,
which can be running independently with your client PC. shutting down client PC
won't affact anything to the running script and it will just keep running
behind the scene. This should look familiar to GNU screen user because it is
the most typical usage scenario of the tool.

* use Expect `dislocate` tool

    ping@ubuntu1404:~$ dislocate crtc -H myrouter
+
press `ctrl-]` then type `ctrl-D` or `exit` to "detach" the session:

    lab@alecto-re1>  
    to disconnect, enter: exit (or ^D)
    to suspend, press appropriate job control sequence
    to return to process, enter: return
    /usr/bin/dislocate1> exit   
    ping@ubuntu1404:~$
+
same as in the case of GNU screen, crtc script is still running in the
background even if it is not shown in the current terminal.  whenever you need
it back, run `dislocate` again to connect the session back

    ping@ubuntu1404:~$ dislocate  
    connectable processes:
     #   pid      date started      process
     1  29571  Sat Apr 11 11:11:28  crtc -H myrouter
     2  27225  Sat Apr 11 10:29:56  crtc rams@jtac
    enter # or pid: 1 
    Escape sequence is ^]
     
    Apr 11 11:12:12 

    lab@alecto-re1>  

* move current session to run in background, just press `ctrl-g` any time
during the session

==== `ctrl-g`     suspend current login session, make it running in background.

With crtc, afer logged into the router successfully, anytime during the
session, the current session can be suspended by pressing `ctrl-g` . 

NOTE: This is not possible if the session was established using the default
telnet client.

    {master}                                                                  
    lab@alecto-re0> set cli screen-width 300                                  
    Screen width set to 300                                                   
                                                                              
    {master}                                                                  
    lab@alecto-re0> set cli timestamp                                         
                                                                              
    Nov 22 12:51:05                                                           
    CLI timestamp set to: %b %d %T                                            
                                                                              
    {master}                                                                  
    lab@alecto-re0> [Sat Nov 22 12:50:33 EST 2014::..c-g was pressed!..]      
                                                                              
    [1]+  Stopped                 crtc alecto@jtac                           
    ping@ubuntu1404:~$ ^C                                                     

to continue, `fg` it to run in the front.

    ping@ubuntu1404:~$ fg      
    crtc alecto@jtac          
                               
    Nov 22 12:52:55            
                               
    {master}                   
    lab@alecto-re0>            

This make it easier to manage multiple active sessions in the
same terminal at the same time, without using external tools like GNU
screen/tmux/etc.

==== some implementation details

`crtc` will "monitor" every keystroke from a user and execute the implemented
actions once the associated key was "seen" by `crtc`. currently, by default
press `ctrl-g` will trigger a "hangup" system signal to current script, which will
effectively move it running "background". It can then be killed (by sending a
SIGKILL signal) as a normal process with the traditional unix `kill` command .

.`ctrl-g` to move the script running in background
===============================================

    {backup}                                                                 
    lab@alecto-re0> c-g was pressed, move this session background!           
                                                                             
    [1]+  Stopped                 crtc alecto-re0-con@jtac                   
    ping@ubuntu1404:~$ kill -9 %1                                            
                                                                             
    [1]+  Stopped                 crtc alecto-re0-con@jtac                   
    ping@ubuntu1404:~$                                                       
    [1]+  Killed                  crtc alecto-re0-con@jtac                   
    ping@ubuntu1404:~$                                                       

===============================================

the `ctrl-g` key make the script behaving in a "compatible" manner within unix
environment. traditionally in many telnet/ssh clients, (most of) keystrokes
will be sent uninterpretedly to the remote device, instead of being intercepted
locally. This is useful in that you can send control characters to trigger some
actions to the processes in the remote device that you are working on. But that
makes it hard to suspend the local client. While we can continue this behavior
in `crtc` script, it will be handy to make use of some not-so-useful keystroke
to do a useful job from the client - that is why `ctrl-g` is choosen to trigger
a hang-up of the client, this is configurable from config file though.

.moving crtc backgound/forground just like a normal apps
====
with `ctrl-g` it is easy to run multiple crtc instance to login to different
routers in the same terminal.

    ping@ubuntu1404:~/bin$ crtc -H alecto
    current log file ~/logs/alecto.log
    it's all yours now!:)
    set cli timestamp

    Dec 02 14:07:28
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re1>
    Dec 02 14:07:28

    {master}
    lab@alecto-re1> c-z was pressed, move this session background!

    [1]+  Stopped                 crtc -H alecto
    ping@ubuntu1404:~/bin$ crtc tintin@jtac
    current log file ~/logs/tintin.log
    telnet tintin.jtac-east.jnpr.net
    ping@ubuntu1404:~/bin$ telnet tintin.jtac-east.jnpr.net
    Trying 172.19.161.18...
    Connected to tintin.jtac-east.jnpr.net.
    Escape character is '^]'.
     Warning Notice

    Please contact Pratima or Brian before making any changes

    tintin-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3R3-S4.7 built 2014-06-30 05:41:54 UTC

    lab@tintin-re0> set cli screen-width 300
    Screen width set to 300

    lab@tintin-re0> it's all yours now!:)
    set cli timestamp

    Dec 02 14:07:49
    CLI timestamp set to: %b %d %T

    lab@tintin-re0>
    Dec 02 14:07:49

    lab@tintin-re0> c-z was pressed, move this session background!

    [2]+  Stopped                 crtc tintin@jtac
    ping@ubuntu1404:~/bin$ jobs
    [1]-  Stopped                 crtc -H alecto
    [2]+  Stopped                 crtc tintin@jtac

now you can use the standard unix job control commands to select which task
(session) you want to work on:

go to alecto session:

    ping@ubuntu1404:~/bin$ fg 1
    crtc -H alecto

    Dec 02 14:08:53

    {master}
    lab@alecto-re1>
    Dec 02 14:08:55

    {master}
    c-z was pressed, move this session background!

    [1]+  Stopped                 crtc -H alecto

go to tintin session:

    ping@ubuntu1404:~/bin$ fg 2
    crtc tintin@jtac

    Dec 02 14:08:59

    lab@tintin-re0>
    Dec 02 14:08:59

    lab@tintin-re0>

====


=== login multiple hosts simultaneously

following command will login to 3 routers all together.

    crtc -h router1 router2 router3

with shell expansion it can be shortened as:

    crtc -h router{1..3}

to switch to the 1st session press `\1`, to go to 2nd session press `\2`, and
so on. `\i` will print info of "current" host.

send same command(s) multiple times(-n 3 -i 5)to all routers at the same time
(-P):

    crtc -n3i5Pc "show system uptime" -h alecto@jtac automatix@jtac

this "multi-hosts" feature is currently only experimental and not well
implemented yet (no much usage case)


==== "parallel" mode

TODO

=== "event script"

to emulate an "Junos event script", add these configurations in `crtc.conf`:

    #event script example
    set event1 "LINK_DOWN"
    set event2 "LINK_UP"
    set action1(myrouter)            {
        "show system uptime" 
        "#link down detected!"
    }
    set action2(myrouter)            {
        "show system uptime"
        "#link up detected!"
    }

    set eventscript(myrouter)       [list       \
        "$event1"        "action1"              \
        "$event2"        "action2"              \
    ]

now run crtc to monitor the configured events:

    crtc -Jc "monitor start messages" myrouter

This will:

* monitor syslog messages for event "LINK_DOWN" and "LINK_UP"
* when "LINK_DOWN" event is "seen", commands configured in `action1` will be
  executed.
* when "LINK_UP" event is "seen", commands configured in `action2` will be
  executed.

=== "op script"

to emulate an Junos "op script", add these configs in `crtc.conf`:

    set cmds6(myrouter) {
        "systeminfo"
        "ospfinfo"
        "bgpinfo"
    }

    set systeminfo(myrouter) {
        "show system uptime"
        "show chassis alarm"
    }

    set ospfinfo(myrouter) {
        "show ospf neighbor"
        "show ospf overview"
    }

    set bgpinfo(myrouter) {
        "show bgp summary | match up"
    }

This is to define some new "commands": 

* `systeminfo`
* `ospfinfo`
* `bgpinfo`

Now these commands can be refered within a session, and `crtc` will "resolve"
each of them into the real, target commands. 

To refer these newly defined commands, use the `!c` "inline" command to invoke
interactive question&answers, which allows you to input the defined commands in
a session:

    labroot@alecto-re0> !c      #<------
    select your choice below:
    1: enter command, or command array(cmds_cli)
       configured in config file to be executed
    q: quit
    1                           #<------
    Enter the command/command array you want to iterate:
       []
    ospfinfo                    #<------
     
    Enter how many iterations you want to run: [1]
    2                           #<------
    Enter intervals between each iteration:  [0]
    3                           #<------
    will iterate command/command group [ospfinfo]  2 rounds with interval 3
        between each iteration,(y)es/(n)o/(q)uit?[y]
    cmds_c(myrouter) = ospfinfo
     
    <<<< start the iterations (2 rounds)

    `- - - - - - - - - -  iteration:1  - - - - - - - - - - `
    <<<<[iteration:1]=> myrouter:
    <<<<  ospfinfo

    labroot@alecto-re0> show ospf neighbor 
    Feb 16 23:07:45
    Address          Interface              State     ID               Pri  Dead
    10.192.0.41      xe-3/1/0.0             Full      192.168.0.6      128    36
    10.192.0.45      xe-4/1/0.0             Full      192.168.0.7      128    38
    1.1.1.2          so-1/2/0.0             Full      100.100.100.100  128    33
    2.2.2.2          so-1/2/1.0             Full      20.20.20.20      128    36

    labroot@alecto-re0> show ospf overview 
    Feb 16 23:07:45
    Instance: master
      Router ID: 192.168.0.69
      ...<snippet>...

    labroot@alecto-re0>  
    <<<<count {3}s before proceeding...
    <<<<type anything to skip...

    `- - - - - - - - - -  iteration:2  - - - - - - - - - - `
    <<<<[iteration:2]=> myrouter:
    <<<<  ospfinfo

    labroot@alecto-re0> show ospf neighbor 
    show ospf overview
    Feb 16 23:07:48
    Address          Interface              State     ID               Pri  Dead
    10.192.0.41      xe-3/1/0.0             Full      192.168.0.6      128    33
    10.192.0.45      xe-4/1/0.0             Full      192.168.0.7      128    35
    1.1.1.2          so-1/2/0.0             Full      100.100.100.100  128    39
    2.2.2.2          so-1/2/1.0             Full      20.20.20.20      128    33

    labroot@alecto-re0> show ospf overview 
    Feb 16 23:07:48
    Instance: master
      Router ID: 192.168.0.69
      Route table index: 0
      ...<snippet>...

    labroot@alecto-re0>  
    <<<<count {3}s before proceeding...
    <<<<type anything to skip...
    all done!
    labroot@alecto-re0>  

NOTE: the command definition can be "nested". you can define a command named
`routinginfo`, which refers to `ospfinfo`, `isisinfo` and `bgpinfo`, each of
which can either refers to the real junos commands, or other commands you wish
to define in the `crtc.conf` file.

=== "programmable" data capture

==== example1

in below config:

    set login_info(vmx-avpn.riot@jtac)       [list     \
        "$"      "crtc -vHQ0 vmx-avpn.vpfe@jtac"       \
        "vmx-avpn.vpfe@jtac:automation done"    "su"   \
        "sword:" "root"                                \
        "#"      "$riot_stats_logging"                 \
        "Logs are stored in (.+)\r" "cat %1"           \
    ]

the last pattern-action pair:

    "Logs are stored in (.+)\r" "cat %1"           \

what it does is, to capture regex:

    "Logs are stored in (.+)\r"

from the previous command output, and use %1 to refer the "back-reference" to
the data captured with the 1st '()'. the next command is then composed based on
this referenced data and sent to the device. this can be conveniently used to
program the command based on the previous command output, dynamically.

==== example2

NOTE: this should work, but with a much more complicated method, not fully
tested yet. will develop more on demand.

//TODO: to be tested:
//* the % need to contain some escapes
//* sometime just want to search a keyword appearance, no need variable

with below arrays defined in crtc.conf:

    set cmds2(myrouter) [list                       \
        "show ospf neighbor"                        \
        "show interface %xe310 terse"               \
        "ping 10.192.0.108 count 2 source %ip"      \
    ]

    set regex_info(myrouter) {
        {1@@(\S+)\s+Full@xe310}
        {2@@inet\s+(\S+)/30@ip}
        {3@@(\s+) packet loss)@percentage}
    }

    set issue_info(myrouter) {
        {3@percentage=="100%"}
    }

when running crtc as below:

    crtc -n10j2 myrouter

it will run each command in sequence, and it will use the data captured in
previous command, to substitute the corresponding variables specified in the
subsequent commands, creating a programmable data capture command list.

.this is how it works:

. `j2` specifies project 2, so data defined in cmd array `cmds2` will be
  read and executed
. the first cmd `show ospf neighbor` is executed first
. from the output of the first cmd, crtc will scan and capture the neighbor
  interface name, using a user defined regex `(\S+)\s+Full`
. if anything got captured, the value will be saved into a user provided
  variable named `xe310`
. crtc continue to run the 2nd cmd `show interface %xe310 terse`
. the `%` indicate a varible evaluation will happen - crtc will evaluate the
  value of variable `xe310`, and substitute this varible with it's value
  captured in previous cmd, in this case the value will be `xe-3/1/0.0`, then
  the final cmd `show interface xe-3/1/0.0 terse` will be sent to the router
. this process will continue: from the output of the `show interface ..`
  command, crtc scan the output and try to capture and IPv4 IP address, using
  regex `inet\s+(\S+)/30` defined in `regex_info` array. the captured value, if
  any, will be saved into the variable named `ip`, for later references
. the last cmd is `ping 10.192.0.108 source %ip`, again, the `%ip` will be
  substituted by the value of variable `ip`, in this case for example it's
  `10.192.0.42` so the real cmd sent to the router will be `ping 10.192.0.108
  source 10.192.0.42`
. from the output crtc will search for the percentage of packet loss, and then
  use that to determine if "a problem" occurs, actions can then be defined (in
  `collect` array).

=== hide login details

It's useful to have all login details printed out if the login was made
manually - the telnet/ssh client will print out the username/password prompt
and wait for your input.

there are some other conditions that it may be better to "hide" the login
process :

. the use of a script to automate the whole process make these messages and
username/password interactions useless. 
. In some cases it's more secure to not even display the username as well as
password. 
. the output will look neat if login process was skipped.

the display of login process can be suppressed by setting `hideinfo` option in
`crtc.conf` file:

    set hideinfo 1

same effect can be achieved by using `-H` flag in command line.

.login with `-H option`:
===============================================

    ping@ubuntu1404:~$ crtc -c "show system uptime" -H alecto
    current log file ~/att-lab-logs/alecto.log
    set cli timestamp

    Nov 30 22:36:28
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> it's all yours now!:)
    show system uptime
    Nov 30 22:36:28
    Current time: 2014-11-30 22:36:28 EST
    System booted: 2014-08-24 17:55:36 EDT (14w0d 05:40 ago)
    Protocols started: 2014-11-24 08:58:14 EST (6d 13:38 ago)
    Last configured: 2014-11-24 08:51:12 EST (6d 13:45 ago) by root
    10:36PM  up 98 days,  5:41, 1 user, load averages: 0.52, 0.26, 0.11

    {master}
    lab@alecto-re0>

===============================================


=== persistent mode

if use running crtc with `-p` (or `set persist_mode 1` in config file), the
session will become a little bit "persistent" - once got disconnected, the
script will be able to detect this situation and try the best to re-login again
- your session will become very "sticky" into the router and won't be kicked
off anymore :)

press `!p` in the session will toggle the persistent mode

.persistent mode
===============================================

    telnet> quit                                                             
    Connection closed.                                                       
                                                                             
    will reconnect in 30s                                                    
    ping@ubuntu1404:~$ telnet b6tsb17.jtac-west.jnpr.net 7021                
    Trying 172.22.194.102...                                                 
    Connected to b6tsb17.jtac-west.jnpr.net.                                 
    Escape character is '^]'.                                                
                                                                             
    Type the hot key to suspend the connection: <CTRL>Z                      
                                                                             
                                                                             
    alecto-re0 (ttyd0)                                                       
                                                                             
    login: lab                                                               
    Password:                                                                
                                                                             
    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC    
    {backup}                                                                 
    lab@alecto-re0> set cli screen-width 300                                 
    Screen width set to 300                                                  
                                                                             
    {backup}                                                                 
    lab@alecto-re0> set cli timestamp                                        
                                                                             
    Nov 24 08:10:40                                                          
    CLI timestamp set to: %b %d %T                                           
                                                                             
===============================================

at this time, there is no easy way to kick off the session - even the router
was reloaded, switched over, or your vty was cleared intentionally. as soon as
the session got disconnected, the script will detect the change and will just
try to login again and again "persistently". 

////
just !p to toggle will be it.

one way to get out of this situation is to start another unix shell terminal
and kill the script from unix level.  There is an other method to do this with
crtc script which we will demonstrate shortly
////


=== logs

name of log file can be specified with some special chars, which will then be
substituted by certain info, e.g:

    set log_filename  "%S-%T.log"

This will make log file looks "seahawks-re0@jtac-2016_0715_2254_47.log"

TODO...

logs will be recorded in file specified with `log_fullname` option. if this
option is not set, use "NAME-OF-SESSION.log" under folder specified by
`log_dir` option.

.logs
====
given config option below:

    set log_dir "~/att-lab-logs"
    set log_fullname "~/abc.log"

the log file will be `~/abc.log`

....
ping@ubuntu47-3:~$ crtc myrouter
<<<CRTC:myrouter:start to login, please wait ...
<<<CRTC:myrouter:to interupt the login process, press <ESC>!
<<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q
telnet -K  alecto.jtac-east.jnpr.net
ping@ubuntu47-3:~$ telnet -K  alecto.jtac-east.jnpr.net
Trying 172.19.161.100...
Connected to alecto.jtac-east.jnpr.net.
Escape character is '^]'.

alecto-re1 (ttyp0)

login: labroot
Password:

--- JUNOS 11.4R5-S4.7 built 2015-05-06 18:55:53 UTC
labroot@alecto-re1> set cli screen-width 300
Screen width set to 300

labroot@alecto-re1> set cli timestamp
Feb 29 11:34:11
CLI timestamp set to: %b %d %T

labroot@alecto-re1>
<<<CRTC:myrouter:login succeeded!
log file: ~/abc.txt         #<------
<<<CRTC:myrouter:automation done

labroot@alecto-re1>
....

if not set `log_fullname` :

    set log_dir "~/att-lab-logs"

the log file will be `~/att-lab-logs/myrouter.log` , for router `myrouter`:

----
ping@ubuntu47-3:~$ crtc myrouter
<<<CRTC:myrouter:start to login, please wait ...
<<<CRTC:myrouter:to interupt the login process, press <ESC>!
<<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q
telnet -K  alecto.jtac-east.jnpr.net
ping@ubuntu47-3:~$ telnet -K  alecto.jtac-east.jnpr.net
Trying 172.19.161.100...
Connected to alecto.jtac-east.jnpr.net.
Escape character is '^]'.

alecto-re1 (ttyp0)

login: labroot
Password:

--- JUNOS 11.4R5-S4.7 built 2015-05-06 18:55:53 UTC
labroot@alecto-re1> set cli screen-width 300
Screen width set to 300

labroot@alecto-re1> set cli timestamp
Feb 29 11:36:35
CLI timestamp set to: %b %d %T

labroot@alecto-re1>
<<<CRTC:myrouter:login succeeded!
log file: ~/logs/myrouter.log       #<------
<<<CRTC:myrouter:automation done

labroot@alecto-re1>
----

====

NOTE: the folder configured in option `log_dir` will be created if not existing
yet.


=== debug mode

TODO:

=== nested crtc

TODO

    set login_info(hoho@att) [list              \
        "$"      "ssh junotac@135.157.14.229"    \
        "sword"  "m0nday"                        \
        "\\\$"   "ssh -l $attproduction_account 199.37.160.81" \
        "sword"  "$attproduction_pass"                   \
    ]

    set login_info($session@hoho)       [list        \
        "$"         "crtc hoho@att"              \
        "succeed" "ssh -l $attproduction_account $session" \
        "sword"     "$attproduction_pass"   \
        ">"         "set cli timestamp"             \
        ">"         "set cli screen-width 300"             \
    ]


=== command substitutions


* `%H` will be substituted to host (router name)
* `%T` will be substituted to current time.

.save config file to a file name that has a timestamp and router name embeded
====

    crtc -Hv0qa "set prefix_mark 0" -c "config" -c "save backup%H_%T" myrouter

this will backup junos config file with a name including timestamp and router
name.

====

=== set "arbitrary" option: -a flag, and !a command

analogous to Expect's built-in flag `-c`

    expect -c "set debug 1" crtc myrouter

with crtc this can be done as:

    crtc -a "set debug 1" myrouter

any other options can be set with -a.

=== built-in multi-party kibitz "on the fly"

.login a router with crtc

                                                                                              
    labroot@seahawks-re0>                                                                     
    Jun 12 17:33:07                                                                           

.press `!K` to send invitation to another user
                                                                                              
    labroot@seahawks-re0> !K                                                                  
        - invite a user to share your current terminal session?[lrq<ENTER>]                   
        (l)ocal user: press "l" or just hit enter                                             
           - send invitation to a user in local server (where crtc script was started)        
        (r)emote user:press "r"                                                               
           - spawn a new shell and from which you can login to remote host and then send      
           invitation to a user in that host                                                  
        (q)uit: press "q"                                                                     
           - quit and return back to current session                                          
                                                                                              
==== local sharing: sharing a terminal session with users in local server

.press `l` or hit enter will send invitation to local user

    who are you going to invite?                                                              
    user1                                                                                     
    will invite user1 ...                                                                     
    kibitz succeeded!                                                                         
                                                                                              
    <<<CRTC:resuming session of seahawks-re0@jtac...                                          
                                                                                              
    labroot@seahawks-re0>                                                                     
    Jun 12 17:33:27                                                                           

.now user1 in local machine got connected
                                                                                              
    user1@ubuntu47-3:~$
    Message from ping@ubuntu47-3 on pts/83 at 16:46 ...
    Can we talk? Run: kibitz -23312
    EOF
    kibitz -23312
    Escape sequence is ^]
    session seahawks-re0@jtac is being shared between ping user1 now...

    Jun 12 17:33:23

    labroot@seahawks-re0>
    Jun 12 17:33:27

.now invite a 2nd local user: user2

    labroot@seahawks-re0> !K                                                                  
        - invite a user to share your current terminal session?[lrq<ENTER>]                   
        (l)ocal user: press "l" or just hit enter                                             
           - send invitation to a user in local server (where crtc script was started)        
        (r)emote user:press "r"                                                               
           - spawn a new shell and from which you can login to remote host and then send      
           invitation to a user in that host                                                  
        (q)uit: press "q"                                                                     
           - quit and return back to current session                                          
                                                                                              
    who are you going to invite?                                                              
    user2                                                                                     
    will invite user2 ...                                                                     
    kibitz succeeded!                                                                         
                                                                                              
    <<<CRTC:resuming session of seahawks-re0@jtac...                                          
                                                                                              
    labroot@seahawks-re0>                                                                     
    Jun 12 17:33:45                                                                           
                                                                                              
    labroot@seahawks-re0> show system uptime                                                  
    Jun 12 17:35:11                                                                           
    Current time: 2016-06-12 17:35:11 EDT                                                     
    System booted: 2016-06-06 05:50:22 EDT (6d 11:44 ago)                                     
    Protocols started: 2016-06-06 13:25:34 EDT (6d 04:09 ago)                                 
    Last configured: 2016-06-06 13:24:03 EDT (6d 04:11 ago) by labroot                        
     5:35PM  up 6 days, 11:45, 2 users, load averages: 0.15, 0.07, 0.01                       
                                                                                              
    labroot@seahawks-re0>                                                                     

.2nd user joined the session

    labroot@seahawks-re0> please hold on while ping is trying to invite more users...
    session seahawks-re0@jtac is being shared between ping user1 user2 now...

    Jun 12 17:33:44

    labroot@seahawks-re0>
    Jun 12 17:33:45

    labroot@seahawks-re0> show system uptime
    Jun 12 17:35:11
    Current time: 2016-06-12 17:35:11 EDT
    System booted: 2016-06-06 05:50:22 EDT (6d 11:44 ago)
    Protocols started: 2016-06-06 13:25:34 EDT (6d 04:09 ago)
    Last configured: 2016-06-06 13:24:03 EDT (6d 04:11 ago) by labroot
     5:35PM  up 6 days, 11:45, 2 users, load averages: 0.15, 0.07, 0.01

    user2@ubuntu47-3:~$
    Message from ping@ubuntu47-3 on pts/84 at 16:46 ...
    Can we talk? Run: kibitz -23359
    EOF
    kibitz -23359
    Escape sequence is ^]
    session seahawks-re0@jtac is being shared between ping user1 user2 now...

    Jun 12 17:33:44

    labroot@seahawks-re0>
    Jun 12 17:33:45

==== distributed sharing: sharing session with users in remote servers

.type `r` after `!K` will can start a new shell

    labroot@seahawks-re0> !K                                                             
        - invite a user to share your current terminal session?[lrq<ENTER>]              
        (l)ocal user: press "l" or just hit enter                                        
           - send invitation to a user in local server (where crtc script was started)   
        (r)emote user:press "r"                                                          
           - spawn a new shell and from which you can login to remote host and then send 
           invitation to a user in that host                     
        (q)uit: press "q"                                        
           - quit and return back to current session             
    no free shell available, will spawn one...                   
    spawned new shell[24187]...                                  
    press \l to list, \NUMBER to switch into a session           
                                                                 
.type `\l` to list currently shell available to use

    labroot@seahawks-re0> host lists:                            
    0:exp7/23292 1:exp11/24187                                   

.type \1 to switch to the new shell#1

    labroot@seahawks-re0> switched to session: 1:exp11/24187     
    ping@ubuntu47-3:~$                                           

from the new shell, login to any remote server and to press `!K` again to send
sharing invitation in the server.

    who to invite on this server?                                                  
    pings ttyla                                                                     
    will invite pings on tty -tty ttyla ...                                         
    pings is not logged in yet, check and try again later!                          
    <<<CRTC:resuming session of seahawks-re0@jtac...                                
                                                                                    
the user in remote server accepted the invitation

    pings@svl-jtac-tool02:~$
    Message from pings@svl-jtac-tool02.juniper.net on ttysm at 14:16 ...
    Can we talk? Run: kibitz -95467
    EOF
    kibitz -95467       #<------
    Escape sequence is ^]
    session seahawks-re0@jtac is being shared between [ping user1 user2 pings] now...
    to exit, type ctrl+], and then "exit"

    Jun 12 17:51:26

    labroot@seahawks-re0>
    Jun 12 17:51:27

now all 5 users (1xoriginal 2xlocal 1xremote) are sharing the same terminal
session to router seahawks.

`!b` can be pressed to toggle the sharing/not sharing.

=== performance fine tune

* turn off `set expect_matchany 0`
* turn off dynamic interact composed from user config: `set enable_user_patterns1`
* turn off all features under interact: `nofeature`
* turn off logging: `set log_when 0`

=== misc usages case

with this added in the config file `crtc.conf` under home dir:

    set cmds1(sonata@jtac) {
        {cprod -A fpc0 -c 'set dc bc "getreg chg rdbgc0"'}
        {cprod -A fpc1 -c 'set dc bc "getreg chg rdbgc0"'}
    }

this command :

    crtc -pr3tiUn 10000 -b "start shell" sonata@jtac 

will behaves as following:

* monitor the shell commands 10000 times 
* with a 1s interval, 
* print a local timestamp for each command,
* before executing cmds provided in the cmds1 array in config file
* if being disconnected (RE switchover, etc):
  - reconnect in 3s 
  - restart the commands sequence from all over (-U) (default is continue from
  where left over)

this is quite similiar with running a shell script from within the junos shell:

    while [ 1 ]
    do
      date
      cprod -A fpc0 -c 'set dc bc "getreg chg rdbgc0"'
      cprod -A fpc1 -c 'set dc bc "getreg chg rdbgc0"'
      sleep 1
    done

this is usefull for the testings when the connections will be kicked off
frequently.

=== more other working examples

TODO:

    crtc -pc "show system uptime" -n 20 -i 5 -r 5 alecto@jtac

    crtc -d3n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps"
        -S "\$pps == 0" anewrouter

    crtc -E "$" -S "telnet alecto.jtac-east.jnpr.net" -E
        "login: " -S "lab" -E "Password:" -S "lab123"
        -E ">" -S "set cli timestamp" -c "show system uptime" -n 3 -i 5
        anewrouter

    crtc -n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" 
        -I "1@pps==pps_prev" anewrouter

    crtc -n 10 -i 20 -R "2@2601\s+rmt\s+LD" anewrouter

    #comment out regex_info and issue_info in config, then:
    ping@ubuntu1404:~$ crtc -D100yc "show interfaces ge-1/3/4" -n 100 \
    -i2 -R "1@@Physical link is (\w+)@updown" -I '1@updown == "Down"' \
    -l pings@juniper.net alecto@jtac

== usage cases

=== device login

==== login to any of your individual device:
 
add the login steps into the `login_info` array like below in your config file
`crtc.conf`:

    set login_info(myrouter) { "\\\$" "telnet x.x.x.x" "login: " "YOUR_LOGIN_NAME" "Password:" "YOUR_PASS" } 
 
this can be improved:

* the long line can be broken into several shorter lines with a style similiar
  to Junos config. footnote:[or 1TBS style according to
http://en.wikipedia.org/wiki/Indent_style#Variant:_1TBS[wikipedia]]
* the "Password", can be shortened as "sword", to make it safer to match.
* the weird-looking `\\\$` can be replaced as `$`, which will be explained later.

the improved login process looks like this:

    set login_info(myrouter) {
        "\\\$" "telnet x.x.x.x" 
        "login: " "YOUR_LOGIN_NAME" 
        "sword:" "YOUR_PASS" 
    } 
 
The step reads as "pattern" "action" pairs:

* "expect", or looking for a "$" sign, as soon as found, send a "telnet
  x.x.x.x" command to the terminal
* "expect" a prompt "login:", then input login name
* "expect" a prompt "sword" , and input password

if the login_info refers other varibles, use `list` keyword plus `\` as
continuation indicator, 

    set your_login_name "ping"
    set your_password "password"
    set login_info(myrouter)       [list            \
        "$"        "telnet x.x.x.x"                 \
        "login: "      "$your_login_name"           \
        "Password:"    "$your_password"             \
    ]
 
then login your router:

    ~pings/bin/crtc myrouter

TODO: explain `$` vs. `\\\$`.

a real example is showned below.

.login to a device
====
.the final config :

    set domain_suffix_con jtac-west.jnpr.net
    set domain_suffix jtac-east.jnpr.net

    #test only
    set login_info(myrouter)       [list               \
        "$"        "telnet alecto.$domain_suffix"    \
        "login: "      "lab"                        \
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]

.capture of the login process:

    ping@ubuntu1404:~$ crtc myrouter
    current log file ~/att-lab-logs/myrouter.log
    telnet alecto.jtac-east.jnpr.net
    ping@ubuntu1404:~$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re0> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re0> it's all yours now!:)
    set cli timestamp

    Nov 30 16:41:38
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0>
    Nov 30 16:41:38

    {master}
    lab@alecto-re0>

====

[NOTE]
====
* you can use variables in the config and do the varible substitutions
  footnote:[actually the config file is just a TCL source code,] . in this
  case 2 vars are used to represent the domain names:
+
    set domain_suffix_con jtac-west.jnpr.net
    set domain_suffix jtac-east.jnpr.net
+
the varible `domain_suffix` is then used in the login_info array element, and
will be substituted into the actual value `jtac-west.jnpr.net`

        "$"        "telnet alecto.$domain_suffix"    \

* after login completed (password accepted and login succeeded), more commands
  can be "attached" in the login_info array. It is handy to always have a
  bunch of commonly used commands (like set timestamp, terminal width,etc) sent
  before starting the work

====

==== login to any router under a "jtac" class

all routers under a "class" share the same login steps. the only thing
differences are the `session` name or the `host` name. this make it possible to
just add one "class" of login steps for multiple routers.

.config:

     set domain_suffix_con jtac-west.jnpr.net
     set domain_suffix jtac-east.jnpr.net
     set jtaclab_login lab
     set jtaclab_pass lab123

     #define a new category "jtac" for all jtac devices
     set login_info($session@jtac)       [list       \
         "$"        "telnet $host.$domain_suffix"    \
         "login: "      "$jtablab_login"             \
         "Password:"    "$jtaclab_pass"              \
         ">"            "set cli screen-width 300"   \
         ">"            "set cli timestamp"          \
     ]

.login:

    ping@ubuntu1404:~$ crtc tintin@jtac
    current log file ~/att-lab-logs/tintin.log
    telnet tintin.jtac-east.jnpr.net
    ping@ubuntu1404:~$ telnet tintin.jtac-east.jnpr.net
    Trying 172.19.161.18...
    Connected to tintin.jtac-east.jnpr.net.
    Escape character is '^]'.
    Please contact ...<snipped>...
    tintin-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3R3-S4.7 built 2014-06-30 05:41:54 UTC

     ************MVPN setup - NYPJAR1 ****************

    lab@tintin-re0> set cli screen-width 300
    Screen width set to 300

    lab@tintin-re0> it's all yours now!:)
    set cli timestamp

    Nov 30 20:12:06
    CLI timestamp set to: %b %d %T

==== login to console of juniper jtac lab routers

the trick is to load the hostmap data provided by lab administrator, which
defines which terminal server/port number to be used when login the console of
a specific router. to differentiate the console session with a mgmt session, a
fake hostname:ROUTERNAME-ren-con is defined. The tcl `regexp` command is used
to extract these info out of the mapping.

.config:

    #juniper jtac router console login: 
    #map a (fake) session name to terminal server data
    array set hostmap {\
        ......
        eros-re0-con                        b6tsb17:7015    \
        eros-re1-con                        b6tsb17:7016    \
        alecto-re0-con                      b6tsb17:7021    \
        alecto-re1-con                      b6tsb17:7022    \
        rams-re0-con                        b6tsa26:7023    \
        bills-re0-con                       b6tsa26:7024    \
        bears-re0-con                       b6tsb09:7013    \
        chargers-re0-con                    b6tsb09:7014    \
        ......
    }

    #domains
    set domain_suffix_con jtac-west.jnpr.net
    set domain_suffix jtac-east.jnpr.net
    set jtaclab_login lab
    set jtaclab_pass lab123

    #analyze the session name and compose the login command:
    #for console session: extract "terminal server" name and "port number"
    #for mgmt session: use the normal "host" to login
    if [regexp {(\w+):(\d+)} $host -> terminal_server port] {
        set login_string "telnet $terminal_server.$domain_suffix_con $port"
    } else {
        set login_string "telnet $host.$domain_suffix"
    }

    #define a new category "jtac" for all jtac devices
    set login_info($session@jtac)       [list           \
        "$"        "$login_string"                      \
        "login: "      "$jtablab_login"             \
        "Password:"    "$jtaclab_pass"              \
        ">"            "set cli screen-width 300"       \
        ">"            "set cli timestamp"              \
    ]

.login:
===============================================

    ping@ubuntu1404:~$ crtc alecto-re0-con@jtac                             
    current log file ~/att-lab-logs/b6tsb17:7021.log                        
    telnet b6tsb17.jtac-west.jnpr.net 7021                                  
    ping@ubuntu1404:~$ telnet b6tsb17.jtac-west.jnpr.net 7021               
    Trying 172.22.194.102...                                                
    Connected to b6tsb17.jtac-west.jnpr.net.                                
    Escape character is '^]'.                                               
                                                                            
    Type the hot key to suspend the connection: <CTRL>Z                     
    alecto-re0 (ttyd0)                                                      
    login: lab                                                              
    Password:                                                               
    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC   
    {backup}                                                                
    lab@alecto-re0> set cli screen-width 300                                
    Screen width set to 300                                                 
                                                                            
    {backup}                                                                
    lab@alecto-re0> set cli timestamp                                       
                                                                            
    Nov 24 08:09:27                                                         
    CLI timestamp set to: %b %d %T                                          
                                                                            
    {backup}                                                                
    lab@alecto-re0> it's all yours now!:)                                   
                                                                            
    Nov 24 08:09:28                                                         
                                                                            
    {backup}                                                                
    lab@alecto-re0>                                                         

===============================================

[NOTE]
====
as expected, to logout, `exit` won't kick yourself out, press the "escape
character" `ctrl-]` to get into telnet client application, and then type `quit`
to exit telnet client

    {backup}            
    lab@alecto-re0> exit
    Nov 24 08:10:05     
    alecto-re0 (ttyd0)  
    login:              
    telnet> quit        
    Connection closed.  

====

==== login to a customer's device

usually for security reason, there won't be direct access to customer's device.
sometime one or two intermediate springboards are used, or a "menu" pop out so
you have to select your target device, or sometime a "token" number (with a
"PIN") is required to input before proceeding.

here is an example of handling the "menu":

.config:

in my own PC:

    set login_info(qfx11@att)       [list           \
        "$"        "ssh svl-jtac-tool02"            \
        "$"         "ssh jtac@12.40.233.51"         \
        "assword:"  "PASSWORD"                      \
        "Enter Option:"     "11"                    \
        "assword:"  "PASSWORD"                      \
        ">"         "set cli screen-width 300"      \
        ">"         "set cli timestamp"             \
    ]

this is to login jtac server first, then login customer's springboard, from
there a menu is popped up for selection.
Or, if you run crtc from the svl server already, then no need the 1st line:

    set login_info(qfx11@att)       [list           \
        "$"         "ssh jtac@12.40.233.51"         \
        "assword:"  "PASSWORD"                      \
        "Enter Option:"     "11"                    \
        "assword:"  "PASSWORD"                      \
        ">"         "set cli screen-width 300"      \
        ">"         "set cli timestamp"             \
    ]


.login:
===============================================

    ping@ubuntu1404:~/bin$ crtc qfx11@att                                        
    current log file ~/att-lab-logs/qfx11.log                                     
    ssh svl-jtac-tool02                                                           
    ssh jtac@12.40.233.51                                                         
    ping@ubuntu1404:~/bin$ ssh svl-jtac-tool02                                    
    Last login: Fri Nov 21 10:20:35 2014 from 172.25.163.165                      
    Copyright (c) 1980, 1983, 1986, 1988, 1990, 1991, 1993, 1994                  
            The Regents of the University of California.  All rights reserved.    
                                                                                  
    FreeBSD 7.4-RELEASE (PAE) #0: Tue Jul 23 08:43:51 PDT 2013                    
                                                                                  
    ******************************************************************************
    **                                                                          **
    **      Hostname:               svl-jtac-tool02.juniper.net                 **
    **      IP Address:             172.17.31.81                                **
    **      Internet Address:       66.129.239.20                               **
    **      Operating System:       FreeBSD 7.4-RELEASE                         **
    **                                                                          **
    ******************************************************************************
    For operational availability issues, contact helpdesk.                        
    For functional enhancement requests, contact jboyle                           
    ******************************************************************************
    JTAC Shell Servers:  svl-jtac-tool01, svl-jtac-tool02, wfd-jtac-tool01        
    ******************************************************************************
                                                                                  
    Saturday Nov 22 12:00p,                                                    
      Proactive reboot planned to clean up stray processes after filer issues     
    ssh jtac@12.40.233.51                                                         
    [pings@svl-jtac-tool02 ~]$ ssh jtac@12.40.233.51                              
    Warning: Permanently added '12.40.233.51' (DSA) to the list of known hosts.   
    jtac@12.40.233.51's password:                                                 
    Last login: Fri Nov 21 12:27:08 2014 from 66.129.239.20                       
    #               For Authorized Use Only                                       
    #                                                                             
    # All of activities on this system are monitored. Anyone using this system    
    # expressly consents to such monitoring and is advised                        
    # that if such monitoring reveals possible evidence of criminal activity,     
    # system personnel may provide the evidence of such monitoring to law         
    # enforcement officals and may be executed to the fullest extent of law       
    #                                                                             
    #                                                                             
                 0.  EXIT                                                         
    1.  ymappr01            2.  ymappr02                                          
    3.  ybpnyubfg           4.  ymafha02                                          
    5.  ymappr03            6.  ymappr04                                          
    7.  ymappr05            8.  ymappr10                                          
    9.  zgcawgnk100         10. zgnawgnk100                                       
    11. zggawgnk100         12. atrpgf01                                          
    13. atrzsp3             14. atbzsp3                                           
    15. atrzsp5             16. atbzsp2                                           
    17. ymappr06            18. ymappr07                                          
    19. ymappr08            20. atbzsn1                                           
    21. atbzsn2             22. atrzsp11                                          
    23. zgvawyfvp03           24. pvcppr1                                         
    25. atrppr2             26. pvcppr2                                           
    27. atrzsp4               28. atbzsn3                                         
    29. atbzsn4               30. atBcnp4                                         
    31. atrzsp9             32. zgvaw003gf                                        
    33. atrzsp10              34. EbhgrFreire                                     
    35. atrppr3             36. atrzsp6                                           
    38. atrzsp7             38. atrzsp8                                           
    39. zgvaw00001ppr9        40. zgvaw00002ppr9                                  
    41. zgraw00001ppr9        42.zgraw00002ppr9                                   
    43. zgcaw00001ppr9        44.zgcaw00002ppr9                                   
    45. zgnaw00001ppr9                                                            
    Enter Option: 11                                                              
                          Warning Notice                                          
                                                                                  
    Password:                                                        
    --- JUNOS 13.2X51-D30_vjunos.50 built 2014-11-21 03:45:16 UTC    
    {master:0}                                                       
    jtac@zggawgnk100> set cli screen-width 300                       
    Screen width set to 300                                          
                                                                     
    {master:0}                                                       
    jtac@zggawgnk100> set cli timestamp                              
    Nov 22 02:18:05                                                  
    CLI timestamp set to: %b %d %T                                   
                                                                     
    {master:0}                                                       
    jtac@zggawgnk100>                                                

===============================================

****
the password and all other sensitive text info has been encrypted in this
example.
****


=== send commands

sometime you prefer always to send some quick commands after logged into a
router successfully. this can be done in many different ways:

* just extend the `login_info` array and attach more commands to be sent<<X1>>
* use a seperate `cmds1` array to hold all extra commands
* use command line option `-c`
* use command line options: `-e` and `-s`

==== -c option

the `-c` (command) option is a easy way to send commands right after a
successful login, it does not require any config change

.config

NONE

this is equivalent to having this settings in config file:

    set cmds1(alecto)         [list \
        "show version"              \
        "show system uptime"        \
    ]

.login
===============================================

    ping@ubuntu1404:~$ crtc -c "show version" -c "show system uptime" alecto
    current log file ~/att-lab-logs/alecto.log
    telnet alecto.jtac-east.jnpr.net
    ping@ubuntu1404:~$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re0> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re0> set cli timestamp

    Nov 30 22:06:22
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> show version
    Nov 30 22:06:22
    Hostname: alecto-re0
    Model: m320
    JUNOS Base OS boot [12.3-20140210_dev_x_123_att.0]
    ...<snipped>...
    JUNOS Routing Software Suite [12.3-20140210_dev_x_123_att.0]

    {master}
    lab@alecto-re0> show system uptime
    Nov 30 22:06:22
    Current time: 2014-11-30 22:06:22 EST
    System booted: 2014-08-24 17:55:36 EDT (14w0d 05:10 ago)
    Protocols started: 2014-11-24 08:58:14 EST (6d 13:08 ago)
    Last configured: 2014-11-24 08:51:12 EST (6d 13:15 ago) by root
    10:06PM  up 98 days,  5:11, 1 user, load averages: 0.00, 0.04, 0.01

    {master}
    lab@alecto-re0> it's all yours now!:)

    Nov 30 22:06:22

===============================================

[NOTE]
====
* this option `-c` (and also the similiar `-e`/`-s` option "two-tuples" can be
used multiple times. this is convenient in case more commands need to be sent
to the router from the command line.
====

here is an example of sending longer list of commands

shutdown and wait 3 seconds, then bring up (rollback) after 3 seconds:

    ~pings/bin/crtc -c "configure" -c "set interfaces xe-3/1/0 disable" -c "commit" -c "SLEEP 3" -c "rollback 1" -c "show | compare" -c "commit" -c "exit" alecto@jtac
 
this maybe look too long, So the same thing can be done with following
configured in `~/crtc.conf`:

    set pre_cmds1(alecto@jtac) [list \
         "configure"        \
    ]
    
    set cmds1(alecto@jtac)         [list   \
        "set interfaces xe-3/1/0 disable"   \
        "commit"   \
        "SLEEP 3"   \
        "rollback 1"   \
        "show | compare"   \
        "commit"   \
    ]
    
    set post_cmds1(alecto@jtac) [list \
         "exit"        \
    ]
 
now just run short command without options:

    ~pings/bin/crtc alecto@jtac

==== `-e` and `-s` options

in addition to configuration file and `-c` option, the 3rd method is to use
`-e` (expect) and `-s` (send) option - the script is expecting for a specific
prompt (`>` in this case of Juniper router privilidge mode) specified by `-e`
option, and once that is seen , a command (specified by `-s`) will then be sent
to the router. 

A good usage example is that if you need to type in another (root) password to
promote yourself with higher privilidge in order to perform some sensitive
commands:

vmx:

    ping@ubuntu1404:~/bin$ crtc -e "\\\$" -s "su" -e "Password:" -s "lab123" -e "#" -s "ethtool em2" -e "#" -s "exit" vmx
    current log file ~/logs/10.85.4.17.log
    telnet -K  10.85.4.17
    ping@ubuntu1404:~/bin$ telnet -K  10.85.4.17
    Trying 10.85.4.17...
    Connected to 10.85.4.17.
    Escape character is '^]'.
    Ubuntu 14.04.1 LTS
    MX86-host-BL660C-B1 login: labroot
    Password:
    Last login: Thu Dec  4 23:09:54 PST 2014 from ubuntu1404.jnpr.net on pts/2
    Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-32-generic x86_64)
    labroot@MX86-host-BL660C-B1:~$ date
    Thu Dec  4 23:11:13 PST 2014
    labroot@MX86-host-BL660C-B1:~$ su
    Password:
    root@MX86-host-BL660C-B1:/home/labroot# ethtool em2
    Settings for em2:
            Supported ports: [ FIBRE ]
            Supported link modes:   1000baseT/Full
                                    10000baseT/Full
            Supported pause frame use: No
            Supports auto-negotiation: Yes
            Advertised link modes:  1000baseT/Full
                                    10000baseT/Full
            Advertised pause frame use: No
            Advertised auto-negotiation: Yes
            Speed: 10000Mb/s
            Duplex: Full
            Port: Other
            PHYAD: 0
            Transceiver: external
            Auto-negotiation: on
            Supports Wake-on: umbg
            Wake-on: g
            Current message level: 0x00000007 (7)
                                   drv probe link
            Link detected: yes
    root@MX86-host-BL660C-B1:/home/labroot# exit
    exit
    labroot@MX86-host-BL660C-B1:~$

or e320:

    crtc -e "#" -s "support" -e "Password:" -s "plugh" -e "#" -s "shell" -e "->" -s "memShow" -e "->" -s "exit" -e "#" -s "exit" e320-svl
    ping@ubuntu1404:~/bin$ telnet -K  172.19.165.56
    Trying 172.19.165.56...
    Connected to 172.19.165.56.
    Escape character is '^]'.


               !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
               !   YOU HAVE CONNECTED TO VERIZON, INC.    !
               !                                          !
               !  UNAUTHORIZED ACCESS WILL BE PROSECUTED  !
               !   TO THE FULLEST EXTENT OF THE LAW ! !   !
               !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

    Telnet password: *******
    Logged in on vty 0 via telnet.
    Copyright (c) 1999-2014 Juniper Networks, Inc.  All rights reserved.

               !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
               ! NO CHANGES TO THIS CONFIG ARE TO BE MADE !
               !  WITHOUT ENGINEERING OR IP NOC SUPPORT!  !
               !                                          !
               !      TACACS LOGS WILL BE AUDITED! ! !    !
               !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
    E320-SVL:vol>enable 15
    E320-SVL:vol#
    E320-SVL:vol#support
    Password: *****
    E320-SVL:vol(support)#shell
    ->
    -> it's all yours now!:)

    E320-SVL:vol(support)#shell
    -> memShow
     status      bytes    blocks   avg block  max block  avg searches
     ------   ---------- -------- ---------- ----------  ------------
    current
        free  2928251816     5200     563125 2868448424
        alloc  520366168   251384       2070      -
    cumulative
        alloc 3072303840  476687427          6      -           1

    value = 0 = 0x0
    -> exit
    E320-SVL:vol(support)#exit
    E320-SVL:vol#

[NOTE]
====
the `-c` command presumes a "commonly used" prompt value (one of the `%>#$`
characters) will pop out, which is by default use the following pattern:

    set pattern_common_prompt	{(% |> |# |\$ |%|>|#|\$)$}

it seems to be good enough for most of the network devices in this industry.
But it can be changed if it doesn't match the prompt that you are trying to
login.
====

todo: add `-v` flag for "pattern_common_prompt" option


=== automate shell tasks

In this example we use `crtc` to create a file sync service between two
servers. For those who got used to `crontab`, this is quite close in term of
the service provided. But there are extra benefits with `crtc`:

* much easier to monitor/troubleshoot the scheduled tasks
* able to schedule tasks involving passwords

////
NOTE: regardless of this, crontab is fine. but I have issues to config
passwordless ssh. don't know why ...

no privilege to start crontab job as non-root in jtac sesrver


    pings@svl-jtac-tool01:~$ service cron status
    eval: cannot open /var/run/cron.pid: Permission denied
    cron is not running.

    pings@svl-jtac-tool01:~$ service cron start
    eval: cannot open /var/run/cron.pid: Permission denied
    Starting cron.
    cron: can't open or create /var/run/cron.pid: Permission denied

    pings@svl-jtac-lnx01:~/CSdata/rsync1$ ping 12.3.167.13
    PING 12.3.167.13 (12.3.167.13) 56(84) bytes of data.
    From 12.126.218.246 icmp_seq=2 Packet filtered
    From 12.126.218.246 icmp_seq=3 Packet filtered
    From 12.126.218.246 icmp_seq=4 Packet filtered
    ^C
    --- 12.3.167.13 ping statistics ---
    4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 3009ms

it seems IT blocked this communication to scooby2 server:

    pings@svl-jtac-tool02:~$ ping 12.3.167.13
    PING 12.3.167.13 (12.3.167.13): 56 data bytes
    36 bytes from 12.126.218.246: Communication prohibited by filter
    Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
     4  5  00 5400 0a39   0 0000  2f  01 02fe 172.17.31.81  12.3.167.13

    36 bytes from 12.126.218.246: Communication prohibited by filter
    Vr HL TOS  Len   ID Flg  off TTL Pro  cks      Src      Dst
     4  5  00 5400 0d81   0 0000  2f  01 ffb5 172.17.31.81  12.3.167.13

    ^C
    --- 12.3.167.13 ping statistics ---
    2 packets transmitted, 0 packets received, 100.0% packet loss


    pings@svl-jtac-lnx01:~/CSdata/rsync1$ ping 12.3.167.13
    PING 12.3.167.13 (12.3.167.13) 56(84) bytes of data.
    From 12.126.218.246 icmp_seq=2 Packet filtered
    From 12.126.218.246 icmp_seq=3 Packet filtered
    From 12.126.218.246 icmp_seq=4 Packet filtered
    ^C
    --- 12.3.167.13 ping statistics ---
    4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 3009ms
////

to sync two pairs of folders between one of jtac server and scooby server, via
rsync:

    pings@svl-jtac-tool01:~$ crtc -Mj1n10000i30 rsync

it keep running rsync to "post" all files found in an `outgoing` folder of one
end of rsync, to the `incoming` folder of the other end.

    jtac-tool:outgoing   -> scooby:incoming 
    scooby:outgoing      -> jtac-tool:incoming 

.configurations

the script will be running in a loop, posting/pulling data between the 2
servers, until it reaches 10000 iterations. The gap between each
iteration are 30 seconds. The script is configurable in many ways:

. via a config file
. via shell environment variables
. on the fly when it is running

.config file `~pings/bin/crtc.conf`

this is the rsync related configuration in the config file `crtc.conf`:

----
#between jtac-tools and scooby server
set rsync_log "/volume/CSdata/pings/rsync4jtac.log"
set rsync_outgoing_dir_local "/volume/CSdata/pings/outgoing/"
#set rsync_incoming_dir_remote "/mnt/NAS/media/incoming/"
set rsync_incoming_dir_remote "/incoming/"
set rsync_incoming_dir_local "/volume/CSdata/pings/incoming/"
#set rsync_outgoing_dir_remote "/mnt/NAS/media/outgoing/"
set rsync_outgoing_dir_remote "/outgoing/"
set rsync_remote "12.3.167.13"
set rsync_remote_account "jtac"
set rsync_remote_pass "jnpr"

#-L: always copy file/folder resolved by link
#-P: Partial: continue from where left off
#-c: --checksum, skip based on checksum, not mod-time & size
set rsync_opt "-avvPzLc --log-file=$rsync_log"
set rsync_post "rsync $rsync_opt $rsync_outgoing_dir_local $rsync_remote_account@$rsync_remote:$rsync_incoming_dir_remote"
set rsync_pull "rsync $rsync_opt $rsync_remote_account@$rsync_remote:$rsync_outgoing_dir_remote $rsync_incoming_dir_local"

set cmds1(rsync-scooby) [list                  \
 "\\\$"     "uptime"                 \
 "\\\$"  "$rsync_post"               \
 "sword" "$rsync_remote_pass"        \
 "\\\$"  "$rsync_pull"               \
 "sword" "$rsync_remote_pass"        \
]
----

changing these configs will change the script running behavior.
all changes in the config file will be read by the script and take effect on
the fly.

////
.to make use of this service

to sync new files or folders, just cp them into the `outgoing` folder of one
server, and they will be sync.ed to the other server. The synchronization will
start every 30s, as configured.

below are the "outgoing" folders currently defined on both server, so copy
anything you want to be sync.ed into these folders:

    jtac server:        /volume/CSdata/pings/outgoing/
    scooby server:      /mnt/NAS/media/outgoing/    

.jtac server:

    pings@svl-jtac-tool01:~/CSdata/outgoing$ ls -l     
    total 0                                            
    -rw-r--r--  1 pings  others  0 Mar  7 08:44 abc    
    pings@svl-jtac-tool01:~/CSdata/outgoing$           

.scooby server:

    jtac@scooby2:/mnt/NAS/media/incoming$ ls -l
    total 0
    -rw-r--r-- 1 jtac jtac 0 Mar  7  2016 abc

.in jtac server put a new file in outgoing folder:

    pings@svl-jtac-tool01:~/CSdata/outgoing$ touch 123
    pings@svl-jtac-tool01:~/CSdata/outgoing$ ls -l     
    total 0                                            
    -rw-r--r--  1 pings  others  0 Mar  7 12:43 123    
    -rw-r--r--  1 pings  others  0 Mar  7 08:44 abc             #<------

.synchronization will start in 30s

    jtac@scooby2:/mnt/NAS/media/incoming$ ls -l
    total 0
    -rw-r--r-- 1 jtac jtac 0 Mar  7  2016 123
    -rw-r--r-- 1 jtac jtac 0 Mar  7  2016 abc   #<------

////

.to access the script

The script is running inside a screen session so it can keep running in the
server without a client attached (e.g.: telnet/ssh client).

To acquire the access to the script, login to same jtac-tool server (e.g.
svl-jtac-tool01 in this case) where the script is running and acquire the
view of the screen session:

    [nzhao@svl-jtac-tool01 ~]$ export TERM=xterm
    [nzhao@svl-jtac-tool01 ~]$ screen -x pings/rsync

This will get the sharing view of the screen terminal where the script is
running. all operations under this screen will be seen by any other authorized
users . 

To obtain the control of the script exclusively:

    [nzhao@svl-jtac-tool01 ~]$ export TERM=xterm
    [nzhao@svl-jtac-tool01 ~]$ screen -dR pings/rsync

NOTE: user authorization can be added/removed as needed per request.

.shell environment variables

set and export variables from current shell and new value will take effect when
crtc runs next time. the name of the shell variable is what was defined in the
`crtc.conf` file, with a prefix `CRTC_`. 

This will change the folders of both end of rsync to new values when crtc is
running next time.

    pings@svl-jtac-tool01:~$ export CRTC_rsync_outgoing_dir_local="folder1"
    pings@svl-jtac-tool01:~$ export CRTC_rsync_incoming_dir_remote="folder2"
    pings@svl-jtac-tool01:~$ export CRTC_rsync_incoming_dir_local="folder3"
    pings@svl-jtac-tool01:~$ export CRTC_rsync_outgoing_dir_remote="folder4"

    pings@svl-jtac-tool01:~$ crtc -Mj1n10000i30 rsync

.change the script running status on the fly

by default the script will pause and wait until 30s expires before the next
iteration of rsync. to change this behavior:

* press `+` to increase it by 2s
* press `-` to decrease it by 2s
* press <ENTER> or any other key to skip the sleep and start next rsync iteration right away.
* press `q` to pause the iteration
* after iteration paused, 
  - press `!R` to resume the iteration from wherever left
  - press `!s` to stop the automation
  - press `!r` to start it all over again

////

==== rsync issue

    pings@svl-jtac-tool01:~$ rsync  -avvPzLc --log-file=/volume/CSdata/pings/rsync4jtac.log /volume/CSdata/pings/outgoing/
     jtac@12.3.167.13:/mnt/NAS/media/incoming/                                                                            
    opening connection using: ssh -l jtac 12.3.167.13 rsync --server -vvlogDtprcze.iLf --partial . /mnt/NAS/media/incoming
    /                                                                                                                     
    Warning: Permanently added '12.3.167.13' (DSA) to the list of known hosts.                                            
    jtac@12.3.167.13's password:                                                                                          
    sending incremental file list                                                                                         
    delta-transmission enabled                                                                                            
    123 is uptodate                                                                                                       
    abc is uptodate                                                                                                       
    jinstall64-16.1-20160302_ib_16_1_jdi.0-domestic-signed.tgz                                                            
      1073046677 100%   33.87MB/s    0:00:30 (xfer#1, to-check=3/7)                                                       
    junos-install-mx-x86-64-15.1I20160311_0949_abhinavt.tgz                                                               
       838368323 100%   20.53MB/s    0:00:38 (xfer#2, to-check=2/7)                                                       
    signed.tgz is uptodate                                                                                                
    WARNING: jinstall64-16.1-20160302_ib_16_1_jdi.0-domestic-signed.tgz failed verification -- update discarded (will try 
    again).                                                                                                               
    WARNING: junos-install-mx-x86-64-15.1I20160311_0949_abhinavt.tgz failed verification -- update discarded (will try aga
    in).                                                                                                                  
    jinstall64-16.1-20160302_ib_16_1_jdi.0-domestic-signed.tgz                                                            
      1073046677 100%    5.14MB/s    0:03:19 (xfer#3, to-check=3/7)                                                       
    junos-install-mx-x86-64-15.1I20160311_0949_abhinavt.tgz                                                               
       838368323 100%   14.40MB/s    0:00:55 (xfer#4, to-check=2/7)                                                       
    ERROR: jinstall64-16.1-20160302_ib_16_1_jdi.0-domestic-signed.tgz failed verification -- update discarded.            
    ERROR: junos-install-mx-x86-64-15.1I20160311_0949_abhinavt.tgz failed verification -- update discarded.               
    total: matches=123234  hash_hits=2615676  false_alarms=68 data=6531216                                                
                                                                                                                          
    sent 6536449 bytes  received 1666575 bytes  5321.46 bytes/sec                                                         
    total size is 1921550917  speedup is 234.25                                                                           
    rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1042) [sender=3.0.7]     

https://lists.samba.org/archive/rsync/2005-December/014176.html

////

TODO

=== file sync between juniper server and ATT server
:expectnote:
:doctype: book
//this is to generate a right side toc
:toc: right
//below two lines make it work for current (2016-03-09) github(asciidoc 1.5.2?)
//:toc: manual
//:toc-placement: preamble
:toclevels: 3
:toc-title: Table of Content
:numbered:
:iconsdir: 
:icons: font
:source-highlighter: prettify
:source-highlighter: highlightjs
:source-highlighter: pygments
:source-highlighter: coderay
:data-uri:
:allow-uri-read:
//:hardbreaks:
:last-update-label!:
//:nofooter:
:Author:  Ping Song
:Author Initials: SP
:Date:   Aug 2015
:Email:   pings@juniper.net
//:title: crtc
:experimental:
:stylesheetdir: {user-home}/Dropbox/asciidoctor-stylesheet-factory/stylesheets/
:stylesheet: {stylesheetdir}maker.css
:stylesheet: {stylesheetdir}readthedocs.css
:stylesheet: {stylesheetdir}github.css
:stylesheet: {stylesheetdir}iconic.css
:stylesheet: {stylesheetdir}rubygems.css
:stylesheet: {stylesheetdir}foundation.css
:stylesheet: {stylesheetdir}foundation-potion.css
:stylesheet: {stylesheetdir}foundation-lime.css
:stylesheet: {stylesheetdir}rocket-panda.css
:stylesheet: {stylesheetdir}riak.css
:stylesheet: {stylesheetdir}asciidoctor.css
:stylesheet: {stylesheetdir}colony.css
:stylesheet: {stylesheetdir}golo.css

to ease the file copy/image delivery work between Juniper shell servers and ATT
servers, a small script is running to periodically synchronize files between
these servers.

                          ATT 
                        ..........................................
                        .                       135.16.32.251    .
    jtac server ----//--.-- scooby ------------------------ atlas.
                  (12.3.167.13) (192.168.46.146)        10.74.18.229
                        .                \              /        .
                        .                 \            /         .
                        .                  \          /          .
                        .                   ATT routers          .
                        .            ATT servers/VMXhypervisors  .
                        .                                        .
                        ..........................................


==== where is the script and how to run

.where

    pings@svl-jtac-tool01:~$ ~pings/bin/jtacsync.sh  

.how to run

    pings@svl-jtac-tool01:~$ export PATH=$PATH:~pings/bin/
    pings@svl-jtac-tool01:~$ jtacsync.sh  

NOTE: to run `jtacsync.sh` script(file sync) or `crtc`(device login) script,
`export PATH=$PATH:~pings/bin/` has to be executed to include my folder into
your `PATH`.

==== folders being sync.ed

with the script running the file sync will be started every 10m, on these
folders/servers:

    shellserver(Juniper): outgoing -> Scooby(att): incoming -> atlas(att): incoming
    shellserver(Juniper): incoming <- Scooby(att): outgoing <- atlas(att): outgoing

locations of these folders:

Jtac servers (Juniper shell server):

    ~pings/incoming         (linking to /volume/CSdata/pings/incoming)
    ~pings/outgoing         (linking to /volume/CSdata/pings/outgoing)

Scooby (ATT ssh server):

    /home/jtac/incoming     (linking to /mnt/sdb4/incoming/)
    /home/jtac/outgoing     (linking to /mnt/sdb4/outgoing/)

Atlas (ATT ftp server):

    /junos/images/incoming
    /junos/images/outgoing

==== steps to copy a file from juniper server to att device

. generate a symbolic link @ jtac server “outgoing” folder, pointing to the
location of the image that you want to transfer:

    pings@svl-jtac-tool01:~$ cd ~pings/outgoing/
    pings@svl-jtac-tool01:~$ ln –s /…/…/jinstall1.tgz  jinstall1.tgz

. check @ Scooby server or atlas server's “incoming” folders (monitor any new
files and the size)
+
--

    pings@svl-jtac-tool01:~$ crtc scooby@attlab 
    jtac@scooby:~$ ls -l incoming/

    pings@svl-jtac-tool01:~$ crtc –H atlas@attlab
    natest@135.16.32.251:/junos/images> ls -lct /junos/images/incoming/

once new files showing up and sizes are correct, you can login att
routers/hypervisor and download files from any of the two servers:

* atlas (10.74.18.229) 
* Scooby (192.168.46.146)

.. login att routers:

    pings@svl-jtac-tool01:~$ crtc ROUTERNAME@attlab
    start shell

    //downloading a jinstall from scooby (192.168.46.146)
    scp jtac@192.168.46.146:incoming/jinstall1.tgz ./
    password is "jnpr"

.. login att VMX host/Hypervisor:

    pings@svl-jtac-tool01:~$ crtc DT401JVPE-host@attlab.hv
    ftp 10.74.18.229
    input username "natest"
    input password "crash"

    //downloading a jinstall from atlas(10.74.18.229)
    ftp> cd /junos/images/incoming
    250 Directory successfully changed.
    ftp> get jinstall1.tgz

--

==== steps to "upload" a file (coredump, logs, etc) from att device to juniper shell server

. login att routers:

    pings@svl-jtac-tool01:~$ crtc ROUTERNAME@attlab
    start shell

    //uploading a coredump tarball to scooby (192.168.46.146)
    scp coredump.tgz jtac@192.168.46.146:outgoing/
    password is "jnpr"

. login att VMX host/Hypervisor:

    pings@svl-jtac-tool01:~$ crtc DT401JVPE-host@attlab.hv

    //uploading a coredump tarball to atlas (10.74.18.229)
    ftp 10.74.18.229
    input username "natest"
    input password "crash"

    ftp> cd /junos/images/outgoing
    250 Directory successfully changed.
    ftp> put coredump.tgz

the uploaded file will be available in juniper shell server's `incoming` folder
in a while.

    pings@svl-jtac-tool01:~/incoming$ ls 
    coredump.tgz

==== the `jtacsync.sh` script

    #!/bin/sh
    count=1
    while true; do
        echo "running crtc $count times..."
        crtc -Mj1W60n10i30q rsync-scooby
        echo "sleeping 10m..."
        sleep 600
        count=$((count + 1))
    done




=== scan all routers

configure a new group "jtac.fqdn" to login routers under charge of a certain
group. in this test we are trying to pull data from jtac department: 

    set login_string "telnet $host"
    set jtaclab_login labroot
    set jtaclab_pass lab123

    set login_info($session@jtac.fqdn)       [list       \
        "$"         "$go_jtac_server"           \
        "sword"     $unixpass                       \
        "$"        "$login_string"    \
        "login: "      "$jtaclab_login"             \
        "Password:"    "$jtaclab_pass"              \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]


put this in a shell script `scanjtac.sh`:

    for i in rockets-re0.ultralab.juniper.net willi-re0.ultralab.juniper.net\
        flip.ultralab.juniper.net redskins-re0.ultralab.juniper.net; do 

        crtc -HW5w5qa \
           -c "show version" \
           -c "show chassis hardware | no-more" \
        $i@jtac.fqdn; 

    done;

run the script:

    ping@ubuntu47-3:~$ jtacscan.sh > scan-0325.txt

    <<<CRTC:mango-re0.ultralab.juniper.net@jtac.fqdn:start to login, please wait ...
    <<<CRTC:mango-re0.ultralab.juniper.net@jtac.fqdn:to interupt the login process, press <ESC>! 
    <<<CRTC:mango-re0.ultralab.juniper.net@jtac.fqdn:to exit script(kill): press <ESC> and !Q    

    <<<CRTC:mango-re0.ultralab.juniper.net@jtac.fqdn:login succeeded!     
    log file: ~/logs/mango-re0.ultralab.juniper.net.log                   
    <<<CRTC:mango-re0.ultralab.juniper.net@jtac.fqdn:automation done      
    bye!:)                                                                

    <<<CRTC:bombay-re1.ultralab.juniper.net@jtac.fqdn:start to login, please wait ...
    <<<CRTC:bombay-re1.ultralab.juniper.net@jtac.fqdn:to interupt the login process, press <ESC>!
    <<<CRTC:bombay-re1.ultralab.juniper.net@jtac.fqdn:to exit script(kill): press <ESC> and !Q   

    <<<CRTC:bombay-re1.ultralab.juniper.net@jtac.fqdn:login succeeded!   
    log file: ~/logs/bombay-re1.ultralab.juniper.net.log                 
    <<<CRTC:bombay-re1.ultralab.juniper.net@jtac.fqdn:automation done    
    bye!:)                                                               
    ...<snippet>...

one thing that can be noticed is that the redirection `>` seems "not work" in here,
because we are still seeing the messages from `crtc` even after we direct the
output to a file. To verify this we can "tail" the file `scan-0325.txt` to see
if the data had been redirected into the file correctly.

    ping@ubuntu47-3:~$ tail -f scan-0325.txt
    show version
    Mar 26 02:13:45
    Hostname: mango-re0
    Model: mx960
    JUNOS Base OS boot [12.1X43.15]
    ......

    {backup}
    labroot@mango-re0> 

    show version
    Mar 26 02:53:07
    Hostname: bombay-re1
    Model: mx960
    JUNOS Base OS boot [12.1X43.15]
    ......

so actually everything works. What happened is CRTC selectively send some
messages to `/dev/tty` instead of `standard output` . The benefit is that these
messages will be processed seperatedly and won't be "merged" with the data
pulled from the devices when the script was redirected to a file - this
provides a good indication about the whole progress of the script. 

TIP: This seperation can be turned off by `-A` knob.

=== problem replication

add following configurations in the config file: `crtc.conf`

    set easyvar 1
    set max_rounds 10
    set interval_cmds 20

    #login steps
    set login_info(anewrouter)       [list          \
        "$"        "telnet alecto.$domain_suffix"   \
        "login: "      "lab"                        \
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]

    #commands to be executed after login
    set cmds1(anewrouter) {
        "show interfaces ge-1/3/0.601 extensive | match pps"
        "show vpls connections instance 13979:333601"
    }

    #define what info to be captured in the cmd output, via regex

    #for the 1st cmd(show interface), capture packets and pps counters, save in
    #variables named #"packets" and "pps", respectively
    set regex_info(1) "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps"

    #for the 2nd cmd(show vpls), capture a match pattern as "rmt   LD",
    #indicating a "remote down" vpls status
    set regex_info(2) "2@@2601\s+rmt\s+LD"

    #define what should be treated as an "issue": in this example it reads:
    #"if var pps value got changed - (not same as its previous value), then we
    #hit the issue!"
    set issue_info(1) "1@pps!=pps_prev"

    #if issue got replicated, send following commands as actions:
    set collect(anewrouter) [ list                              \
        "show interfaces ge-1/3/0.601 extensive | no-more"  \
    ]

    #otherwise (issue not appear), send other commands as actions before next
    #iteration
    set test(anewrouter) [ list                 \
        "configure private"                     \
        "run show system uptime"                \
        "exit"                                  \
    ]

////
    #this is old way to doing the same thing as regex_info(1), requires tcl
    code
    #issue definition code: apply to the 1st cmd in cmds1
    #set cmds1_code_anewrouter_1 {
    #    #match from any line of the output
    #    set does_match [regexp {Input  packets:\s+(\d+)\s+(\d+) pps} $cout_line -> packets pps]
    #    if {($does_match)} {
    #        if {$pps == 0} {
    #            myputs "pps 0, no traffic, the issue appears!"
    #            return 1
    #        }
    #    }
    #}
////


now run crtc to login the router and start the replication:

    crtc anewrouter

what exactly it does are:

. login to a router named "anewrouter", with the steps configured in
  `login_info` array
. once login succeeds, it then sends commands configured in the `cmds1` array
. repeat this 10 times (`max_rounds`) , with 20s pause between each iteration
  (`interval_cmds`)
. in each iteration, try to use a regex (`regex_info`)to examine "issues" in
  each command output:
  * in the first command, extract traffic counters and put int the provided varible
  named `packets` and `pps` respectively
  * in the 2nd command, use another pattern "rmt    LD" to check vpls states
. define the issue (`issue_info`) as *any one of* footnote:[if `-x` being used,
  the issue will be defined as *all* conditions must be met ] the following two
  criterias being met:
    * `pps` (traffic rate) ever changes
    * VPLS remote connection status matches "rmt    LD" pattern (remote down)
. if the issue being reproduced , the `collect` array will be executed and
  following will happen:
    * run "show interfaces ..." command
    * send an email notificaiton to the user
    * script exit (or other actions if configured)
. otherwise (issue not "seen"), the `test` array will be executed and following
  will happen:
    * run "configure" command
    * run "run show system uptime" command
    * run "exit" command
    * repeat above proess

TODO: to add more explanations to following examples...

`-x` option change the default rule of defining an issue as "any one of" the
defined criterias being satisfied, to "all" defined criterias need to be
satisfied (regex being matched).

as long as ge-1/3/4 status became "Up", shut it down

    crtc -yn 10 -i 20 \
        -c "show interfaces ge-1/3/4 | match Physical" \
        -R "1@@Physical link is (\w+)@updown" \
        -I {1@updown == "Up"} \
        -Y "configure" -Y "set interfaces ge-1/3/5 disable" -Y "commit" \
        -Y "exit" -Y "show interfaces ge-1/3/5" alecto@jtac\

while backup link remains up (and forwarding traffic), as soon as master link
ge-1/3/4 came up, switch AE traffic from backup link ge-1/3/5 to the primary.

    crtc -xyn 10000 -i 0 -C NONE \
      -c "show interfaces ge-1/3/4 | match Physical" \
      -c "show interfaces ge-1/3/5 | match Physical" \
      -R "1@@Physical link is (\w+)@updown1" \
      -R "2@@Physical link is (\w+)@updown2" \
      -I '1@updown1 == "Up"'  \
      -I '2@updown2 == "Up"' \
      -Y "configure" \
      -Y "set interfaces ge-1/3/5 disable"  \
      -Y "commit" \
      -Y "exit" \
      -Y "show interfaces ge-1/3/5" \
      alecto@jtac

while link ge-1/3/5 was down, as soon as master link ge-1/3/4 went down,
remove the "disable" command under ge-1/3/5 (to bring it up and forward the
traffic)

    crtc -xyn 10000 -i 0 -C NONE \
      -c "show interfaces ge-1/3/4 | match Physical" \
      -c "show interfaces ge-1/3/5 | match Physical" \
      -R "1@@Physical link is (\w+)@updown1" \
      -R "2@@Physical link is (\w+)@updown2" \
      -I '1@updown1 == "Down"'  \
      -I '2@updown2 == "Down"' \
      -Y "configure" \
      -Y "delete set interfaces ge-1/3/5 disable"  \
      -Y "commit" \
      -Y "exit" \
      -Y "show interfaces ge-1/3/5" \
      alecto@jtac


=== full parameterization

this example demonstrate the "full parameterization" capability of the crtc
script.

let's say if you don't want to bother creating or editing a config file, most
of the config options in the config file can also be refered as command line
options.

So same exact test listed in the previous example can be performed with this
long one-liner command:

    crtc -yn 10 -i 20 \
      -E "$" -S "telnet alecto.jtac-east.jnpr.net" -E "login: " -S "lab" \
      -E "Password:" -S "lab123" -E ">" \
      -c "show interfaces ge-1/3/0 extensive | match pps" \
      -c "show vpls connections instance 13979:333601" \
      -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" \
      -R "2@@2601\s+rmt\s+LD" \
      -I "1@pps!=pps_prev" \
      -Y "show interfaces ge-1/3/0.601 extensive | no-more" \
      -N "configure" -N "run show system uptime" -N "exit" \
      a_new_router

. login to a router named "anewrouter", with the steps configured by `-E`
  and `-S` options pairs
. once login succeeds, it then sends two commands provided by `-c` options,
. repeat this 10 times, with 20s pause between each iteration (-n 10 -i 20)
. in each iteration, try to use the regex (-R) to match the command output
  of each:
  * in first command, check traffic counters:"packets" and "pps"
  * in 2nd command, check vpls states: "rmt    LD"
. declare the issue (-I) being reproduced if *any one of* the following two
  criterias being met:
    * `pps` (traffic rate) ever changes
    * VPLS remote connection status ever became "LD"
. is the issue reproduced?
. if yes (-Y) the following will be performed:
    * run "show interfaces ..." command (-Y)
    * send an email notificaiton to the user
    * script exit
. otherwise (-N), run more following test 
    * run "configure" command
    * run "run show system uptime" command
    * run "exit" command
    * repeat above proess

==== compromise between config file and command line options

with following configured in ~/crtc.conf :

    set login_info(anewrouter)       [list          \
        "$"        "telnet alecto.$domain_suffix"   \
        "login: "      "lab"                        \
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]
    
    set cmds1(anewrouter) {
        "show interfaces ge-1/3/0.601 extensive | match pps"
        "show vpls connections instance 13979:333601"
    }

the following command monitor either of the 2 commands, if the `pps` was found
to be `0` in the 1st command, then the issue got reproduced.

    crtc -n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps"
        -I "\$pps == 0" anewrouter

same as above, but the issue will be thought of as "reproduced" as long as the
`pps` value changes:

    crtc -yn 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" 
        -I "1@pps!=pps_prev" anewrouter

monitor the 2nd command to see if remote vpls connection appears "LD" status

    crtc -n 10 -i 20 -R "2@@2601\s+rmt\s+LD" anewrouter

another difference between these 2 commands is that, the 2nd one used "-y"
command line option, mapping to `easyvar` config option. with this option you
don't need to use `$` when refering to a varible -- this avoids the need to
have to escape a `$` with `\` in UNIX shell. so these 2 are equal:

    crtc -....(options without a -y)... -I "1@\$var1...\$var2" anewrouter
    crtc -y...(other options)... -I "1@var1...var2" anewrouter

=== capture a specific values

This example:

     crtc -H3yn 10 -i 20 \
         -E "$" -S "telnet alecto.jtac-east.jnpr.net" \
         -E "login: " -S "lab" -E "Password:" -S "lab123" -E ">" \
         -c "show interfaces ge-1/3/0.601 extensive | match pps" \
         -c "show vpls connections instance 13979:333601" \
         -R "1@1@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" \
         -I "1@pps==pps_prev" -R "2@@2601\s+rmt\S+LD" \
         -Y "show interfaces ge-1/3/0.601 extensive | no-more" \
         -N "configure" -N "run show system uptime" -N "exit" \
         -V "\$pps \$packets" anewrouter

will do the same test as above examples, except that:

* it won't display the interactions (-H3 : "hide" even more interactions)
* if the issue does not occur, nothing will be displayed
* if the issue occurs, it will print the monitored "packet" and "pps" value
in one line output.

a simpler example:

just capture pps and packets 20 times:

    crtc -H3yn 10 -i 20 \
        -E "$" -S "telnet alecto.jtac-east.jnpr.net" \
        -E "login: " -S "lab" -E "Password:" -S "lab123" -E ">" \
        -c "show interfaces ge-1/3/0 extensive | match pps" \
        -R "1@2@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" \
        -V "\$pps \$packets" anewrouter


=== timestamping all commands

.config

this is equivalent to having this setting in config file:

    set timestamp 1


.`-t` option

    ping@ubuntu1404:~/bin$ crtc -tH earth@jtac
    current log file ~/logs/earth.log
    it's all yours now!:)
    set cli timestamp

    Dec 02 07:58:32
    CLI timestamp set to: %b %d %T

    lab@earth-re0>
    Dec 02 07:58:32

it looks no much different in the beginning. Now typing a return you will now
get 2 timestamps:

    lab@earth-re0>
    Dec 02 07:57:59 2014(local)         #<------
    Dec 02 07:58:40                     #<------

    lab@earth-re0>

the 2nd one is the original timestamp provided by Junos `set cli timestamp`
command. the 1st one is provided by crtc script , using the local time wherever
the script is running. you might notice the slight time difference between the
two footnote:[it's not hard to eliminate the gap and sync the local time
display with the remote, but I'll add this ability later]

    lab@earth-re0> show system alarms
    Dec 02 08:33:13 2014(local)
    Dec 02 08:33:54
    2 alarms currently active
    Alarm time               Class  Description
    2014-12-02 01:05:57 EST  Major  FEB 1 Failure
    2014-12-02 00:57:34 EST  Major  FEB not online for FPC 1

now, every command invoked by a carriage return will also invoke this extra
local timestamp in the output, and will be recorded in the log file. 

having redundent timestamp (in Junos) looks stupid, the extra timestamp provided
by crtc can be disabled by typing `!t` (toggle timestamping).

    lab@earth-re0> !t
    timestamp 0

    Dec 03 08:52:09

    lab@earth-re0> show system alarms
    Dec 03 08:50:08
    2 alarms currently active
    Alarm time               Class  Description
    2014-12-02 01:05:57 EST  Major  FEB 1 Failure
    2014-12-02 00:57:34 EST  Major  FEB not online for FPC 1

    lab@earth-re0>

[NOTE]
====

This might look unnecessary since Junos already provided this feature. However,
not all network devices support this feature - think about other "non-junos"
devices which don't come with this feature by default - cisco , unix, Junos-e,
older version of Junos devices (before 7.x?), or even junos shell mode .., this
might be a handy trick to timestamp all of your command for easier analysis
later on.

====

==== timestamp command from non-Junos devices

.timestamping Juniper E-serials
=====================================================

    slot 16->print__11Ic1Detector
    Dec 02 15:25:14 2014(local)         #<------timestamp

    state=NORMAL
    sysUpTime=0x03B33619
    passiveLoader=0x0C001994
    crashPusherRequested=1
    crashPusherReady=1

                             Detect         Credit   Last       Ask      Last
    Detector Name            Count  Credits Every    Credit     Every    Ask
    ------------------------ ------ ------- -------- ---------- -------- ----------
    Hw2CamClassifierDetector      0   5/  5    300 s 0x0000A942  5000 ms 0x03B3345D

=====================================================

[[X5]]
.`!t` command

suppose you've logged into the remote device, but forgot to invoke crtc with
`-t` option. Now you realize you need this feature. without exiting current
session, just press `!t` (an exclamation mark and `!` and a letter `t`
literally), you will have this feature. press `!t` again will toggle it off.

.timestamping UNIX server
=====================================================

    [pings@svl-jtac-tool02 ~]$ crtc -H vmx
    current log file ~/logs/10.85.4.17.log
    it's all yours now!:)

    date

    Last login: Tue Dec  2 08:32:42 PST 2014 from svl-jtac-tool02.juniper.net on pts/5
    Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.13.0-32-generic x86_64)

       Documentation:  https://help.ubuntu.com/

      System information as of Tue Dec  2 08:32:42 PST 2014

      System load:    16.34               Users logged in:       1
      Usage of /home: 78.4% of 343.02GB   IP address for virbr0: 192.168.122.1
      Memory usage:   15%                 IP address for br-ext: 10.85.4.17
      Swap usage:     0%                  IP address for br-int: 172.16.0.3
      Processes:      618

      Graph this data and manage this system at:
        https://landscape.canonical.com/

    110 packages can be updated.
    60 updates are security updates.

    labroot@MX86-host-BL660C-B1:~$ date
    Tue Dec  2 08:34:08 PST 2014
    labroot@MX86-host-BL660C-B1:~$
    labroot@MX86-host-BL660C-B1:~$
    labroot@MX86-host-BL660C-B1:~$
    labroot@MX86-host-BL660C-B1:~$ uptime
     08:34:17 up 33 days, 21:34,  6 users,  load average: 16.39, 16.35, 16.34
    labroot@MX86-host-BL660C-B1:~$
    labroot@MX86-host-BL660C-B1:~$
    timestamp 16-host-BL660C-B1:~$ !t

    Dec 02 15:35:43 2014(local)         #<------timestamp turned on
    labroot@MX86-host-BL660C-B1:~$ uptime
    Dec 02 15:35:45 2014(local)
     08:34:22 up 33 days, 21:34,  6 users,  load average: 16.36, 16.35, 16.33
    labroot@MX86-host-BL660C-B1:~$ uname -a
    Dec 02 15:35:52 2014(local)
    Linux MX86-host-BL660C-B1 3.13.0-32-generic #57-Ubuntu <...truncated...>

=====================================================

.timestamp Junos shell/PFE commands
=====================================================

    lab@mx86-jtac-lab> set cli screen-width 300
    Screen width set to 300

    lab@mx86-jtac-lab> it's all yours now!:)
    set cli timestamp
    Dec 03 08:20:24
    CLI timestamp set to: %b %d %T
    lab@mx86-jtac-lab> start shell      #<------the last timestamped Junos "CLI command" 
    Dec 03 08:20:30
    % ls -l /var/log/messages           #<------no timestamp
    -rw-rw----  1 root  wheel  4194094 Dec  3 08:21 /var/log/messages
    %                                   #<------press !t (may not display)
    timestamp 1                         #<------now timestamp option is set

    Dec 03 05:27:37 2014(local)
    % ls -l /var/log/messages           
    Dec 03 05:27:47 2014(local)         #<------shell/pfe commands now both got timestamped
    -rw-rw----  1 root  wheel  4195050 Dec  3 08:25 /var/log/messages
    root@mx86-jtac-lab% vty 1
    Dec 03 05:28:17 2014(local)

    BSD platform (Pentium processor, 1536MB memory, 0KB flash)

    VMXE(mx86-jtac-lab vty)# show pfe statistics traffic
    Dec 03 05:28:35 2014(local)
    PFE Traffic statistics:
                        0 packets input  (0 packets/sec)
                        0 packets output (0 packets/sec)

    PFE Local Traffic statistics:
    ...<snipped>...

=====================================================


=== features

simplest interact, no extensive pattern match in `interact` function call.
use this to disable all "features" under interactive mode, and get the best
performance.

* 2: rich-features (slow/monitoring mode)
     - generate interact code dynamically based on user patterns
     - set enable_user_patterns 3
     - a keystroke will move to fast mode
* 1: flexible
    - ignore user pattern, only use static interact code
    - set enable_user_patterns 1
    - timeout will move to slow mode
* 0: no features (fast mode)

.no features

fastest, but no features under interact mode

TODO

.rich-features

with most features under interact, maybe slow in some system

TODO

.about "flexible features":
when crtc init, it goes to monitoring mode; a user keystroke (any keyboard hit)
will move it to fast mode; when user stop typing, after some timeout it will
move back to slow/monitoring mode again.

=== enable_user_patterns

when set, crtc script will read and parse `user_patterns` array in the config
file, and generate dynamic expect and interact code which can detect user
configurable patterns and take the corresponding actions configured for these
patterns. This can be very powerful and useful in some cases. e.g. you want the
script to automatically reconnect whenever a "Connection closed" pattern is
"seen", below code configured in crtc config file will suffice in many cases:

    set user_patterns(pattern_connection_close_msg) \
        [list     \
        {Connection closed.*\r|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer}  \
        RECONNECT interact_only]

in some other cases you want the script to "retry" the same command over and
over until succeed, you can use below config:

    set user_patterns(connection_timeout)  \
        {"Connection timed out; remote host not responding" RETRY expect_only}

combining these configs will generate an interesting "persistent" effect of the
script: 

----
VMXE(DT408JVPE vty)# show jnh 0 pool summary
                Name        Size      Allocated     % Utilization
               EDMEM    33554432        7715791               22%
               IDMEM      317440         289384               91%
                OMEM    33554432          37288             < 1%%
         Shared LMEM    33555456       33591820              100%

VMXE(DT408JVPE vty)#
VMXE(DT408JVPE vty)# show jnh 0 pool usage
[Connection to dt408jvpe closed                 <1>

<<<interact: detected event:                    <2>
<<<-Connection to dt408jvpe closed-
<<<which matches defined pattern(event):
-<<<Connection closed.*\r|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer-
action is RECONNECT                             <3>
persistent mode set 3, will reconnect in 10s
type !p to toggle persistent mode               <4>
<<<<count {10}s before proceeding...            <5>
<<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
.........[Fri Jun 10 12:28:29 EDT 2016]:[dt408jvpe@attlab]:myinteract:..process - exp7(in) :exp7(out) :kibitz()..
set cli timestamp
Jun 10 12:41:02
CLI timestamp set to: %b %d %T

j-tac-ps1@DT408JVPE> Read from remote host 12.3.167.8: Operation timed out <6>
Connection to 12.3.167.8 closed

<<<interact: detected event:
<<<-Connection to 12.3.167.8 closed-
<<<which matches defined pattern(event):
-<<<Connection closed.*\r|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer-
action is RECONNECT                             <7>
persistent mode set 3, will reconnect in 10s
type !p to toggle persistent mode
<<<<count {10}s before proceeding...
<<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
.........[Fri Jun 10 23:11:35 EDT 2016]:[dt408jvpe@attlab]:myinteract:..process - exp7(in) :exp7(out) :kibitz()..
set cli timestamp
Jun 10 23:24:10
CLI timestamp set to: %b %d %T

j-tac-ps1@DT408JVPE>
[Connection to dt408jvpe closed                 <8>

<<<interact: detected event:
<<<-Connection to dt408jvpe closed-
<<<which matches defined pattern(event):
-<<<Connection closed.*\r|Connection to (\d{1,3}\.){3}\d{1,3} closed|Connection to \S+ closed|Connection reset by peer-
action is RECONNECT
persistent mode set 3, will reconnect in 10s
type !p to toggle persistent mode
<<<<count {10}s before proceeding...
<<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
.....detected -Connection timed out; remote host not responding-        <9>
user_action configured as RETRY                 <10>
will repeat same command and continue...
.press ctrl-c to get a prompt >$...

......

<<<<count {10}s before proceeding...
<<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
detected -Connection timed out; remote host not responding-
user_action configured as RETRY                 <11>
will repeat same command and continue...
.press ctrl-c to get a prompt >$...

<<<<count {10}s before proceeding...
<<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
set cli timestamp                               <12>
Jun 12 01:09:52                 
CLI timestamp set to: %b %d %T

j-tac-ps1@DT408JVPE>
Jun 13 06:43:13
----

<1> TODO
<2>
<3>
<4>
<5>
<6>
<7>
<8>
<9>
<10>
<11>
<12>

powerful as it looks, this feature may severely lower down the performance of
the script. the reason is that if there are too many pattern in the config
file, both `expect` and `interact` function call of the script will have a big
amount of patterns attached, and since for each pattern the Expect has to look
for a match anytime a new chunk of characters are received, this effectively
slow down the script. disable this feature in the case that performance is a
concern.

config file:

    set enable_user_patterns 0

inline command: `!u`


=== expect_matchany

"expect_matchany" can tell the diff between a stalled flow or a "flowing" flow
without a pattern match (usually in the end when the command prompt returns).
expect_matchany first uses `.+` to collect chars and then look for pattern
match. 

expect_matchany shortens the big "waittime_cmd", which otherwise needs to be
set to relatively long, to handle the "big commands" that take long time to
return.  with expect_matchany, it can "recognize" something is still ongoing if
the command gives info indicating the progress - just any popped out messages
will indicate that the command execution is going well but just not finished
yet, and it's not a condition of "getting stuck" in somewhere...

TIP: test shows it's acutally not that slow - definitely not loop for every
received character, as I presumed before...

issues:

how to emulate the "timeout" special pattern?

current implementation: don't look for "timeout" literally from the match
within ".+" match. do it outside.

=== prefix_mark

prefix_mark can be used to "insert" some certain strings into each line of a
command output - making log files "grep friendly". so it is useful when
searching info from log files with grep.

=== auto re-attemp with diff account

add this in crtc.conf:

    set reconnect_eval {

        global session
        global SKIP_retry1

        if ![info exists SKIP_retry1] {
            set attlab_account $attlab_account2
            set attlab_pass $attlab_pass2
            set SKIP_retry1 1
            puts "retry $SKIP_retry1 time!"
        } elseif {$SKIP_retry1==1} {
            set attlab_account $attlab_account3
            set attlab_pass $attlab_pass3
            set SKIP_retry1 2
            puts "retry $SKIP_retry1 time!"
        } elseif {$SKIP_retry1==2} {
            set attlab_account $attlab_account4
            set attlab_pass $attlab_pass4
            set SKIP_retry1 3
            puts "retry $SKIP_retry1 time!"
        } else {
            set attlab_account $attlab_account5
            set attlab_pass $attlab_pass5
            unset SKIP_retry1
            puts "too much wrong login, will exit..."
        }

        set login_info(myrouter) [list                  \
            "$" "telnet alecto-re0.$domain_ultralab"    \
            "login: " "$jtaclab_account"                \
            "Password:" "$jtaclab_pass"                 \
            ">" "set cli screen-width 300"              \
            ">" "set cli timestamp"                     \
        ]

        set login_info($session@attlab)       [list        \
            "$"         "ssh jnpr-se@12.3.167.8"        \
            "assword"  "$jnprse_pass"                   \
            ">$"           "$host"                      \
            "login: "      "$attlab_account"               \
            "Password:"    "$attlab_pass"                  \
            ">"            "set cli screen-width 300"   \
            ">"            "set cli timestamp"   \
        ]

        if !$in_jserver {
            set login_info($session@attlab) [subst -nobackslashes {    \
                "$"     "ssh pings@$juniperserver"      \
                "sword" {$unixpass}                     \
                $login_info($session@attlab)            \
            }]
        }

    }

    set user_patterns(login_retry_diff_account)             [list "ogin: $" "RECONNECT $reconnect_eval"]

when login devices under @attlab domain, crtc will try multiple different
account when ran into errors.

=== share output to other terminal or a file

.redirect to other terminal
. get terminal file name via `stty` command
. redirect crtc script to the other terminal file

----
labroot@alecto-re0> !Quit the script!                       │labroot@alecto-re0>
ping@ubuntu47-3:~$                                          │May 07 13:53:38
ping@ubuntu47-3:~$                                          │
ping@ubuntu47-3:~$                                          │labroot@alecto-re0> show system uptime
ping@ubuntu47-3:~$ crtc myrouter > /dev/pts/90              │May 07 13:55:01
<<<CRTC:myrouter:start to login, please wait ...            │Current time: 2016-05-07 13:55:01 EDT
<<<CRTC:myrouter:to interupt the login process, press <ESC>!│Time Source:  LOCAL CLOCK
<<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q   │System booted: 2016-04-21 20:47:46 EDT (2w1d 1
                                                            │Protocols started: 2016-04-21 20:49:32 EDT (2w
<<<CRTC:myrouter:login succeeded!                           │Last configured: 2016-04-21 20:56:06 EDT (2w1d
log file: ~/logs/myrouter.log                               │ 1:55PM  up 15 days, 17:07, 2 users, load aver
<<<CRTC:myrouter:automation done                            │
                                                            │labroot@alecto-re0>
----
                                                                                   
.redirect crtc to a normal file

share the interactio to multiple people in real time:

----
ping@ubuntu47-3:~$ crtc -a "set double_echo 1" myrouter > temp-double-echo.txt     
<<<CRTC:myrouter:start to login, please wait ...                                   
<<<CRTC:myrouter:to interupt the login process, press <ESC>!                       
<<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q                          
                                                                                   
<<<CRTC:myrouter:login succeeded!                                                  
log file: ~/logs/myrouter.log                                                      
<<<CRTC:myrouter:automation done                                                   
set cli timestamp                                                                  
May 07 14:43:06                                     │ping@ubuntu47-3:~$ tail -f temp-double-echo.txt 
CLI timestamp set to: %b %d %T                      │labroot@alecto-re0>                           
                                                    │May 07 14:44:11                               
labroot@alecto-re0> show version                    │labroot@alecto-re0> show version
May 07 14:43:08                                     |May 07 14:43:08                         
Hostname: alecto-re0                                |Hostname: alecto-re0                    
Model: m320                                         |Model: m320                                   
Junos: 15.1R3.6                                     |Junos: 15.1R3.6                               
JUNOS Base OS boot [15.1R3.6]                       |JUNOS Base OS boot [15.1R3.6]                 
JUNOS Base OS Software Suite [15.1R3.6]             |JUNOS Base OS Software Suite [15.1R3.6]       
JUNOS platform Software Suite [15.1R3.6]            |JUNOS platform Software Suite [15.1R3.6]      
JUNOS Web Management [15.1R3.6]                     |JUNOS Web Management [15.1R3.6]               
----

=== semi-automation

configure this in config file `crtc.conf`:

    set login_info(myrouter) [list                  \
        "$" "telnet alecto-re0.$domain_ultralab"      \
        "login: " "USER_INPUT"                  \
        "Password:" "USER_INPUT"                 \
        ">" "set cli screen-width 300"              \
        ">" "set cli timestamp"                     \
    ]

and run crtc:

----
ping@ubuntu47-3:~$ crtc myrouter
<<<CRTC:myrouter:start to login, please wait ...
<<<CRTC:myrouter:to interupt the login process, press <ESC>!
<<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q
ssh -o "StrictHostKeyChecking no" pings@172.17.31.80
ping@ubuntu47-3:~$ ssh -o "StrictHostKeyChecking no" pings@172.17.31.80
Password:
please input answer to "login: ":   <1>

Warning: No xauth data; using fake authentication data for X11 forwarding.
telnet -K alecto-re0.ultralab.juniper.net
...<snipped>...
   from JTAC labs, more info: http://quickstart.jtaclabs.juniper.net/

pings@svl-jtac-tool01:~$ telnet -K alecto-re0.ultralab.juniper.net
Trying 172.19.161.101...
Connected to jtac-m320-r005.ultralab.juniper.net.
Escape character is '^]'.

alecto-re0 (ttyp0)

login: wronglogin
wronglogin
Password:
please input answer to "Password:":
wrongpass

Login incorrect
login: labroot
labroot
Password:lab123


--- JUNOS 15.1R3.6 built 2016-03-24 18:39:40 UTC
labroot@alecto-re0> set cli screen-width 300
Screen width set to 300

labroot@alecto-re0>
<<<CRTC:myrouter:login succeeded!

log file: ~/logs/myrouter.log
<<<CRTC:myrouter:automation done
set cli timestamp
May 09 00:57:40
CLI timestamp set to: %b %d %T

labroot@alecto-re0>
----

=== event script

Configuring Event script:

1. Add this in ~pings/bin/crtc.conf file:


router attga301me6@attlab

    #These defined events to be monitored:
    #PIM log:
    set event_pim {rpd\[\d+\]: %DAEMON-5-RPD_PIM_NBRDOWN: Instance PIM\S+ PIM neighbor[^&]+removed due to:[^&]+ hold-time period}
    #BGP log:
    set event_bgp {rpd\[\d+\]: %DAEMON-4: bgp_hold_timeout:\d+: NOTIFICATION sent to [^&]+Hold Timer Expired Error}

    #These defined actions(cmds) to be executed when any of the events is “seen”:

    set ukern_trace(attga301me6@attlab)            {
        "start shell pfe network fpc0"
        "sh ukern_trace 45"
        "sh ukern_trace 46"
        "sh ukern_trace 47"
        "sh ukern_trace 65"
        "exit"
    }

    #These “bind” each event to its associated action:
    set eventscript(attga301me6@attlab)       [list      \
        "$event_pim"        "ukern_trace"      \
        "$event_bgp"        "ukern_trace"      \
    ]



Same for router chgil302me6@attlab …

    #######router chgil302me6@attlab ###############

    set ukern_trace(chgil302me6@attlab)            {
        "start shell pfe network fpc0"
        "sh ukern_trace 45"
        "sh ukern_trace 46"
        "sh ukern_trace 47"
        "sh ukern_trace 65"
        "exit"
    }

    set eventscript(chgil302me6@attlab)       [list      \
        "$event_pim"        "ukern_trace"      \
        "$event_bgp"        "ukern_trace"      \
    ]


more routers can be added like this

2. Now Run crtc script to monitor the two routers:

    #terminal1:
    pings@svl-jtac-tool01:~$ ~pings/bin/crtc attga301me6@attlab

    #terminal2:
    pings@svl-jtac-tool01:~$ ~pings/bin/crtc achgil302me6@attlab

each of the script now will monitor the defined event (log), and if any thing
showing in your terminal matches to the event, the actions (cmds) will be
executed

3. Type “monitor start message” in each router, to display the logs in real
time, so crtc script can “monitor”.

.test:

you can do “file show test” in attga301me6@attlab, and you will see the action will get triggered…

.Logs:   

Everything you see from your terminal, will be recorded in ~logs/ROUTERNAME.log
(not ROUTERNAME@attlab).  e.g.: ~/logs/attga301me6.log


=== monitor device

    pings@svl-jtac-tool01:~$ ~pings/bin/crtc -Aj2n1000i600 attga302me6@attlab

Whait does:

. Login att router attga302me6
. send cmds configured in "project 2” (-j2)
. for any command, print complete output despite of the paging pause (-A)
. send them 1000 iterations, with internals of 10min (-n1000i600)


cmds configured in “project 2” is in `~pings/bin/crtc.conf`:

    #monitoring att router for hongpeng {{{2}}}
    set vmx_pfe_cli(attga302me6@attlab) {                 
        "show ddos policer stats all"                     
        "show pfe statistics traffic"                     
        "show jnh host-path-stats"                        
        "show jnh 0 exceptions terse"                     
    }                                                     
                                                          
    set vmx_re_cli(attga302me6@attlab) {                  
        "show system statistics"                          
    }                                                     
                                                          
    set vmx_pfe_cli_hostpath(attga302me6@attlab) {        
        "show ttp stats"                                  
        "show packet statistics"                          
    }                                                     

    #project 2
    set cmds2(attga302me6@attlab) [list            \
        vmx_re_cli                          \             
        "start shell pfe network fpc0"      \             
        vmx_pfe_cli                         \             
        vmx_pfe_cli_hostpath                \             
        exit                                \             
    ]                                                     

== introduction

.An *old topic* 

There were no good ways/tools to interact with a remote host/router from within
a shell script...*the expect* script (or other expect-like or expect-inspired
tools) might be the best available tool that can help on this.

.*The goal*

The goal of implementing this script, is to provide a generic method , to
automate tasks that involves interactions with a remote host, being it a
router, switch, firewall, a server, or even the local host itself. 

.*The idea*

The idea is to ``disassamble'' the interactive tasks into different stages or
components, then implement those most common components in the form of some
generic modules. once this is done, then most individual tasks can be composed
by just reassambling one or more of these common modules, which optionally can
be further fine tuned by a bunch of "config options" (or "knobs").  This can be
implemented as parameterized command line interface (CLI), or as configuration
command in a configuration file. the benefit is obvious - the script can be
used either as an individual terminal based tool, or being called in a 3rd
party script with different parameters as needed.

.*modules*

Here are the most common modules in most of the interactive tasks:

////
* login process: login to the router with defined steps
* send commands: send a serial of commands and receive the outputs
* (optional) monitor outputs: examine the output of some commands for specific
keyword(s)/value(s), maybe periodically
* (optional) see if the monitored value(s) meet some pre-defined criteria
* if yes, it is assumed the issue is seen ("reproduced"), then:
* (optional) collect more information after the issue is reproduced
* (optional) otherwise repeat some test over and over
* ...
////

//[width="90%",cols="3,10",options="header,footer"]
[width="90%",cols="3,10",options="header"]
|=====================================================================
|The Modules                    |Functions
| loggin process                | login to the router with defined steps
| sending commands(optional)    | send a serial of commands and receive the
outputs
| (optional) monitoring outputs | examine the output of some commands for
specific keyword(s)/value(s), maybe periodically
| (optional) comparison         |see if the monitored value(s) meet some
pre-defined criteria
| (optional) data collection    |if yes, it is assumed the issue is seen
("reproduced"), then more information after the issue is reproduced
| (optional) more test          |if no, repeat some test over and over
|=====================================================================


.*options*

options are all user-changeable attributes of a session, for example:

* `anti_idle_timeout`: for how long you would like to send a character to the
  session , just to prevent a telnet sesssion from being timed out?
* `anti_idle_string`: what string(s) you prefer to use for that purpose?
* `max_round`: how many iterations you would like to send those commands?
* `interval_cmds`: what's the wait interval between each iteration of a batch
  of commands execution?
* `interval_cmd`: what's the wait interval between each individual command?
* ...

=== some design considerations

.*problem in most small script*

one problem in most small, "one-time use" script is that:it always assume a
lot of things:

* presuming telnet is used as login prototocol, 
* one level direct login , 
* prompt should be just ">", 
* etc. 

if the presumption changes next time the script won't work - you need to
understand the code and be able to change these attributes in order to make it
work in other scenarios. 

This script was designed with at least 2 things in mind:

configurable::
+
--
- what kind of device you want to login ?(junos, junos-e, cisco, linux server, or even local machine...)
- how do you want to login to a device? (telnet, ssh, ftp, rlogin, ...)
- how many level it takes to get to the target machine? (via 2 springboard,
an a equipment list/menu and another login)
- user name, password, yes/no answers, menu selections
- what command you want to send right after the login process?
- do you even want to automate the command sending after login? or just stay
interactive mode and type command by yourself?

One benefit is that all config options can be centralized in one seperate
configuration file, so the user of the script can just change the script
behavior and don't need to read and understand the code at all. 
--

re-use local existing resource::

Use whatever tools avaiable in the local machine (ssh, telnet,ftp, rcp, sftp,
your another scripts, etc). There is no presumption/limitation about what tools
you want to use, Telnet/SSH clients are the most common ones though.  

fully parameterized::
+
--
all configurable options can be either specified in a centralized config file,
or directly from the command line.

this makes the tool "script-friendly" and can be re-used conveniently in other
scripts. This will very much leverage the power of UNIX CLI and make the
further extension more efficient.
--

Some useful/interesting features will be highlighted below.

=== feature highlights

this small script currently supports most of the features I could expect from
other commercial tools dealing with interactive sessions (e.g. secureCRT).
footnote:[and it supports some small features I don't see from other tools:)]
I may extend it whenever I feel other useful features/options can be easily
implemented.

auto-login:: you "program" the login steps in a config file, the script will
follow the steps to login to remote device.

hide login details::  this is to suppress the (maybe sensitive) login steps
that normaly would be printed to the terminal (not in secureCRT?).

anti-idle:: by default a session will be terminated if nothing has been typed
in a certain period of time. this feature make it possible to retain the
session by sending "keepalives" - usually a configurable ("control") character
(`anti_idle_string`) after every certain period of time (`anti_idle_timeout`).

persistent session (or, "reconnect on disconnect"):: as soon as the session got
disconnected, the script will detect this event, reconnect, and continue either
from where it left off, or start executing the commands from all over again
(not in secureCRT?).

timestamping:: a lot of devices don't print a timestamp when a command was
typed. the `timestamp` option make it possible to timestamp any typed command,
or even all lines from command output, from any device it logged in. This might
be useful when when testing time-sensitive issues [not in secureCRT?].

sending commands after login:: to send some predefined commands, either right
after a successful login, or anytime in the session.

"quickmode":: login, send commands, and then instead of staying interactive
(the default) but just exiting. this is useful when used in a shell script
which, needs to carry on after a command output is grabbed.

issue definition and replication:: this maybe an interesting feature. Currently
there are two options that one allows the user to provide a regex to catch some
specific pieces of information from the command output, and the other one
allows the user to provide an "logical expression" to define what exactly
should be treated as an "issue". Depending on the evaluation of the "issue",
the script will either optionally examine pre-defined actions and execute them
if the issue is "seen", or execute some configured "test" before continuing to
the next iteration. This can be used in problem replications.

email notification:: sending an email notification is easy and cheap way to
report issues to the user from a script. The receipee can be the current user
or whoever configured (in the `emailto` option), when any of the defined
criterias is met [not in secureCRT].

"autopaging":: with this knob turned on, the command "show config | no-more"
for example can be simplied as just "show config". the script will monitor the
command output and it will, once detected some pending content, keep pulling
the rest of the output (by keep "pressing" spaces or other configurable keys
for you) until hitting the end of the output [not in secureCRT].

Overall, this small script implemented:

* something like "the secureCRT", but ...
* in command line mode (so server/script-friendly) and ...
* was tailored with just those most interested and useful part of the features.  
  (cutting off all GUI related features, e.g, the "font", "color" ,etc -- who
  cares about the font in a terminal? ). 
* plus something misc small features (email sending, timestamping, hide login,
  issue monitoring, etc). 

so it is much smaller and more programmable, and can be running from a terminal
of any unix/linux/mac machine equiped with a standard expect tool, which is the
only software requirement.

TIP: The well known benefit of a CLI script implementation (vs. a GUI) is that,
it is much easier to be integrated into other tools, being it a
perl/shell/python/tcl script, or even another instance of the same script. All
standard unix pipe/redirection/job management mechanism applies and all
traditional terminal tools can be used.

== how to run the script

As mentioned, the `crtc` tool runs on any `*nix` or `*nix-like` machine equiped
with basic `expect` software. it does not require any additonal libraries/3rd
party modules/etc. and as a small script basically there is nothing to
"install" but simply run it from my folder. 

All examples in this doc work in linux, freebsd, Mac, cygwin.

////
Optionally, if you want to automate login to your own devices, you can do it in
one of the 2 ways:

* use a <<X3,config file>>

    - add your own login data for your new hosts
    - add commands to be executed along with/after the login
    - change the default option settings
    - <<X3, more details about config file>> will be provided later in this
    doc.

* use command line options
////

[NOTE]
====
the same file `crtc` includes all
codes, documents footnote:[even this tutor doc], options default values (can be
overided by options in config files or options in command line) , etc:

* tcl/expect codes (the script)
* default value of all config options
* usage, "online" document (this doc)
====

To run the script, follow *one of* these steps:

* just execute it directly from my folder:

    ~pings/bin/crtc -c "show chassis alarms" -n 3 -i 30 alecto@jtac
+
IMPORTANT: this method does not work for "recursive" `crtc` execution - a
"recursive" execution happens when one `crtc` script is calling another `crtc`
script instance in the configured login or command data, which will be explained
further later.

* add my script location to your system `PATH` variable, and execute `crtc`
directly without referring where it is.

    pings@svl-jtac-tool02:~$ export PATH=$PATH:~pings/bin/ 
    pings@svl-jtac-tool02:~$ crtc REMOTEDEVICE
+
TIP: this is recommended, since no need to copy and you are always use the
updated version of the script.

* "install" `crtc` under your own folder
+
--
if you perfer to run it under your home folders or wherever, just copy the 2
files over and `chmod` the script will be it. 

TIP: Optionally you can also change your `PATH` environment variable, to make
the system to be able to locate this script from your folder.  or put the PATH
in your `.bashrc` to make it persistent.  

    svl-jtac-tool02#cd YOUR_FOLDER
    (YOUR_FOLDER)#cp /homes/pings/bin/crtc ./
    (YOUR_FOLDER)#cp /homes/pings/bin/crtc.conf ~/
    (YOUR_FOLDER)#chmod 755 crtc
    (YOUR_FOLDER)#export PATH=$PATH:YOUR_FOLDER

////
## add your own data
to add the login info of your own remote devices that your are going to login
with this script, edit the crtc.conf file (with vi, vim, whatever editor) 

    vim ~/crtc.conf

add your own server entries and login data:

    set login_info(myserver)       [list  \
        "$"        "ssh bob@123.1.1.1"    \
        "Password:"    "mypass"           \
        "$"            "date"             \
    ]
////

then run the script without the need to refer my folder:

    #crtc myserver
--

=== the script: just one file

here is the file in jtac server (e.g `svl-jtac-tool02`): 

    [pings@svl-jtac-tool02 ~]$
    [pings@svl-jtac-tool02 ~]$ cd bin
    [pings@svl-jtac-tool02 ~/bin]$ ls -l | grep crtc   
    -rwxr-xr-x   1 pings  others    5802 Nov 21 10:48 crtc      #<-the script
    -rw-r--r--   1 pings  others   82313 Feb 16 20:20 crtc.conf #<-config file

as can be seen the script itself is just one file `crtc` , it will read a
config file `crtc.conf` before start running.

[[X3]]
=== config file

a config file is a place where you can define or fine-tune all of the data,
options , or the default values of options, that will influence the behavior of
the script.  

The `crtc` script will try to locate a config file in below sequence:

* from what is specified in `-C CONFIG_FILE_FULLNAME` option
* from the same folder where `crtc` script is located
* from home folder

accordingly, it can be generated in one of the following ways:

. create a config file manually

* create a config file named `crtc.conf` and place it under your home folder.  
* if you would like to use a config file with other names or under other
  location, specify a `-C CONFIG_FILE_FULLNAME` command option when running crtc.

. copy an existing template config file into your home folder

    cp ~pings/bin/crtc.conf ~/

. generate a config template from `crtc` directly with `-g` option when
executed:

    crtc -g
+
this will generate a config file template `crtc.conf.template`, which can be
renamed/modified as wished.

////
The config file is needed only in the case that:

* you want to login some new devices.
* you prefer to change the config options (to control the login behavior) in
  the config file instead of using command line options - this makes the command
  line much shorter.

the very basic usage:

    /homes/pings/bin/crtc HOST

where `HOST` is name of whatever device you defined in the config file under
your home folder `~/crtc.conf`.  the script came with login data of all Juniper
JTAC routers under domain name `jtac-east.jnpr.net` (for management port) and
domain name `jtac-west.jnpr.net` (for console port), which is configurable in
the config file:

    set domain_suffix_con jtac-west.jnpr.net
    set domain_suffix jtac-east.jnpr.net

so without any extra configuration, it can be used to login any JTAC routers
(mgmt and console) under these domains.
////

=== config options 

the value of an option (another name - "config knob") can be changed at
different time of a session, and at least in 5 different places (levels):

. initial default value defined in crtc script when it got initiated
. config file
. environment variable
. command line options
. "inline" in the session (keystroke commands)

TIP: the keystroke commands is some "keystrokes" you can type in after the
session started. 

basically this is how the script runs and the sequence that a config option get
its value:

. start with a default "hard-coded" value in the `crtc` script itself
. check and see if a config file exists (`~/crtc.conf`), or specified from
  command line (`-C` FILENAME), read the options setting from it once found
. check if there is any environment variable configured for `crtc` (name
started lik `CRTC_NNN`)
. accept and reset options from command line
. start to automate the login to the remote device with configued steps and
  send provided commands
. once login succeed, return control to the user
. from now on what the user can do:

  * type commands interactively to the device, 
  * type some special "keystroke commands" to change the options 

Taking an example of the `timestamp` option:

* the default, initial value is *0*
* `set timestamp 1` in config file will overide it to *1*
* `export CRTC_timestamp 0` from shell (before running crtc) will change it to *0*
* now running `crtc` with option `-t 0`, will set the `timestamp` back to *1*
* after `crtc` finish the login automation and return the control to you ,
  press a "keystroke command" `!t` (a `!` and a `t` , literally) will toggle
  the `timestamp` value to *0*, type `!t` will toggle it again back to *1*.

The same applies to other options.

== `crtc` options

=== options list

To provide the most control to the login process, `crtc` current supports
options and "inline" commands that can be used before, or/and after the login
process begins.  To get a list, run `crtc` with `-h` option:

    ping@ubuntu47-3:~$ crtc -h 

or, in a session type `!h` command, both will print a list with a short
explanation of currently supported options and inline commands, :

    ping@ubuntu47-3:~$ crtc -h 
    Usage:/home/ping/bin/crtc/crtc [OPTIONS]
      OPTIONS:
         -G                 :generate a config file template
         -h/H               :this usage/detail tutorial
         -K                 :kibitz(,not in use,TODO)
         -e                 :edit config file
         -l                 :list configured hosts info
         -v                 :print version/license info
    Usage:/home/ping/bin/crtc/crtc [OPTIONS] session
      OPTIONS:
         -A                 :set/unset auto-paging (no-more)
         -a                 :set arbitrary attributes(options)
         -b/B "show cli"  :commands before/after loop
         -c "show version | no-more" :commands (in a loop)
         -C <config file|NONE> :read config file
         -d                 :set/unset debug info
         -D                 :max_hits: max times issue will be detected
         -e ">" -s "show version | no-more" :expect and send
         -E "$" -S "telnet alecto.." :same, but for login
         -f router1.log     :router log file name
         -F log_dir         :folder name of log file
         -g                 :gap between each cmd
         -h <host1> <host2> .. :login to multiple hosts
         -i <SECONDS>       :interval between each iteration of all cmds
         -j <NUMBER>        :project
         -J                 :event script
         -k                 :exit session also/not exit script
         -l "pings@juniper.net" :email address to send log attachment
         -L                 :email subject
         -m                 :monitor mode(keep sending cmds)
         -n <count>         :send command <count> time(s)
         -o                 :set/unset "login only" (ignore all cmds)
         -O                 :emailcont: email contant
         -p                 :set/unset "persist mode"
         -P                 :when used with -h,run commands in parallel
         -q                 :set/unset "quick mode" (quit after done)
         -Q                 :disable all "features"
         -r <SECONDS>       :reconnect interval
         -R "1@packets@s+(d+)s+(d+) pps@packets@pps" :regex and vars string
         -I "1@pps == pps_prev"  :issue definition
         -t                 :set/unset timestamping all commands
         -T                 :timeout all output lines
         -u                 :continue_on_reconnect
         -U                 :print "login_succeed_signature" when login succeeded
         -H                 :set/unset "hidden mode" (hide login step details)
         -w <SECONDS>       :waittime_login
         -W                 :use crtc in shell: set in_shell
         -V                 :print value matched by the regex from -R
         -x                 :all_med - define issue as all cmds met -I criterias
         -X                 :lock_session, set key_interact to an impossible value
         -y                 :use barewords instead of $var for varibles in -I expression
         -Y                 :commands when match found(Yes) in -R
         -N                 :commands when match not(No) found in -R
         -z                 :compress/don't compress log before attach to email
         -Z                 :no_anti_idle: disable anti_idle_timeout
    to read the full manual:
         /home/ping/bin/crtc/crtc -H
    inline commands:
      !a  toggle auto_paging option
      !b/B  not in use
      !c  specify and repeat a command
      !C  editing current config file
      !d  toggle debug option
      !D  dump (debugging) input/output characters
      !e!  start expect-command pair group automation
      !e?  same, but asking for array's name
      !E  attach log in email and send to user
      !f/F  not in use
      !g/G  not in use
      !H  toggle hideinfo option
      !I  enter interpreter mode
      !j/J  not in use
      !k  toggle exit_sync option
      !K  not in use
      !l log file operations
        !le: email current log file
        !ln: start a new log file 
        !ls: stop/suspend logging to current log file
        !lS: stop/suspend all logging
        !lr: resume logging to current log file
        !lv: view the log file
        !ll: list the log file(name)
      !v(=!lv):view the log file
      !m  print host list
      !M  not in use
      !n  not in use
      !N  toggle no_feature
      !o  toggle login_only option
      !O  reload the config
      !p  toggle persistent option
      !P  toggle parallel option
      !q  toggle nointeract (quick mode) option
      !Q  exit the script
      !r  repeat the previous cmds execution
      !R  resume the previous cmds from where left over
      !s  stop the remaining of previous automations
      !S  not in use
      !t  toggle timestamp option
      !T  toggle timestamp_output_line option
      !u/U  not in use
      !v  print version/license info
      !V  not in use
      !w/W  not in use
      !x/X  not in use
      !y/Y  not in use
      !z  not in use
      !Z no_anti_idle
    session management
      i
      N  N refers a session number
    helps
      !?  list all keystoke commands
      !h  print usage
      !i  print detail tutor docs
    other commands:
      Ctrl-g suspend the script
      Ctrl-(SIGQUIT) to stop the cmds automations
      automation control:
          q   quit the automation
          <SPACE> continue(escape sleeping)
          <ENTER> same as <SPACE>
          Q   exit the script
      SPECIAL CMDS:
         GRES               :Junos GRES request
         SLEEP <SECONDS>    :sleep some seconds before sending next command
         REPEAT M N         :repeat the last M commands N times
         UPGRADE /var/tmp/..:upgrade junos release

NOTE: there is one rule here when running `crtc` with options: the session name
must be the last parameter in the command line.

=== option maps

TODO

[options="header", width="50%",cols="2,2,4,4,3"]
|========================================================
|CLI |inline cmd
         |options               |options_cli index (internal use) 
                                                       |data structure(internal use)
|-A  |   |auto_paging           |auto_paging           |               
|-B  |   |                      |post_commands         |post_cmds_cli 
|-b  |   |                      |pre_commands          |pre_cmds_cli  
|-c  |!c |                      |commands              |cmds1          
|-C  |   |config_file           |config_file           |               
|-d  |   |debug                 |debug                 |               
|-D  |   |max_hits              |max_hits              |               
|-e  |   |                      |expect                |cmds_cli      
|-E  |   |                      |EXPECT                |login_info_cli 
|-F  |   |log_dirname           |log_dirname           |               
|-f  |   |log_filename          |log_filename          |               
|-g  |   |interval_cmd          |interval_cmd          |               
|-H  |   |hideinfo              |hideinfo              |               
|-h  |   |                      |hosts                 |hostlist       
|-i  |   |interval_cmds         |interval_cmds         |               
|-I  |   |                      |issue                 |issue_info     
|-k  |   |exit_sync             |exit_sync             |               
|-K  |   |kibitz                |kibitz                |               
|-L  |   |emailsub              |emailsub              |               
|-l  |   |emailto               |emailto               |               
|-m  |   |max_rounds  1000000000|max_rounds  1000000000|               
|-n  |   |max_rounds            |max_rounds            |               
|-N  |   |                      |reproduced_no         |test_cli       
|-O  |   |emailcont             |emailcont             |               
|-o  |   |login_only            |login_only            |               
|-P  |   |parallel              |parallel              |               
|-p  |   |persistent            |persistent            |               
|-q  |   |nointeract            |nointeract            |               
|-Q  |!N |feature               |feature               |               
|-r  |   |reconnect_interval    |reconnect_interval    |               
|    |   |reconnect_on_event    |reconnect_on_event    |
|-R  |   |                      |regex_vars            |regex_info     
|-s  |   |                      |                      |               
|-S  |   |                      |                      |               
|-t  |   |timestamp             |timestamp             |               
|-T  |!T |timestamp_output_line |timestamp_output_line |               
|-u  |   |continue_on_reconnect |continue_on_reconnect |               
|-U  |   |                      |CONTINUE_ON_RECONNECT |               
|-V  |   |print_matched_value   |print_matched_value   |               
|-v  |   |version               |version               |               
|-w  |   |waittime_login        |waittime_login        |               
|-x  |   |all_met               |all_met               |               
|-X  |   |lock_session          |lock_interact         |               
|-y  |   |easyvar               |easyvar               |               
|-Y  |   |                      |reproduced_yes        |collect_cli    
|-z  |   |compress_log          |compress_log          |               
|-Z  |!Z |                      |no_anti_idle          |               
|========================================================

=== internal data structure (arrays)

==== `login_info`

    set login_info(myrouter)       [list               \
        "$"        "telnet alecto.$domain_suffix"    \
        "login: "      "lab"                        \
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]

this is used to store all information needed to login to a remote device.

[NOTE]
====
due to the tcl method of handling regex, if "$" doesn't work, use "\\\$".
see appendix for more details on this.

`$` MAY or may NOT work:

    set login_info(myrouter) [list          \
        "$"         "ssh myjumpstation1"    \
        "password"  "mypass"                \
        "$"      "ssh my_target_machine" \
        ">"         "set cli timestamp"     \
    ]

`\\\$` will work:

    set login_info(myrouter) [list          \
        "$"         "ssh myjumpstation1"    \
        "password"  "mypass"                \
        "\\\$"      "ssh my_target_machine" \
        ">"         "set cli timestamp"     \
    ]

====

==== `cmdsN`

the *N* in `cmdsN` can be between *1* to *10*, this is used to store all
commands to be sent to the remote device *AFTER* login was successful. all
commands listed in this array will be executed, unless `-o` command line option
or `login_only` config option is set, in that case `crtc` will just login and
data in `cmdsN` array won't be executed. the `cmdsN` array can be executed
multiple time, if the `max_rounds` were configured to a number bigger than one.

==== `pre_cmdsN`

same as `cmdsN`, but will be executed just 1 time *BEFORE* the `cmdsN` array.

==== `post_cmdsN`

same as `pre_cmdsN`, but will be executed just 1 time *AFTER* the `cmdsN` loop.

////
==== internal array `cmds2`

after the script finished the login automation and returned the control back to
the user(you), if you still need to start a new automation , e.g. to send more
"patten-command" pairs , this array can be used to hold all those
pattern-command pairs. Press `!c!` will invoke the script to send the commands
listed in this array to the current session. More details about this later.

==== internal array `cmds3`

same as `cmds2` array, but no need to wait for an expected "pattern" for each
command. the script will presume a set of default patterns that are good enough
for most network devices.

    set pattern_common_prompt	{(% |> |# |\$ |%|>|#|\$)$}
////

==== `regex_info`

==== `issue_info`

==== `collect`

contains a list of commands that will be sent if the expected issue appears

==== `testN`

contains a list of commands that will be sent if the expected issue didn't
appear.
    
[NOTE]
====
[[X1]]
* you can also put all commands to be sent in the same `login_info` array. 

    set login_info(myrouter)       [list               \
        "$"        "telnet alecto.$domain_suffix"    \
        "login: "      "lab"                        \
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
        ">"            "configure"                  \
        "#"            "set interfaces xe-3/1/0 disable"   \
        "#"            "show | compare"   \
        "#"            "commit"   \
    ]
+
  but put in a seperate array `cmds1` is better. the `login_info` was used
  mainly for login process, attach all commands in same array will make the
  `login_only` (just login and ignore all later commands) option useless.

* another way to send commands after successful login is through command line
  parameters (or options), which will be introduced later in this doc.
====

==== an example

Here is an example illustrating the usage of these internal arrays.

for example, right after logged in to the router, you want to:

* flap a port 3 times 
* with 10s interval between each iteration
* in each iteration: 
  - disable the port
  - commit the config change 
  - wait 3s before bringing it back up
  - rollback config to bring the port back
  - commit the config change

these actions can be configured in a `cmdsN` array, e.g `cmds1`:

    set cmds1(myrouter) {"configure" "set interfaces xe-3/1/0 disable" commit "SLEEP 3" rollback 1 "show | compare" commit exit}

you may not like to enter `configure` (to enter "privilidge mode") and `exit`
(to exit the "privilidge mode") commands over and over again, and it maybe
better to enter "privilidge mode" once, from there to repeat those port flap
actions and then exit once the test is done. 

To achieve that:

* put `configure` in `pre_cmds1`
* put `exit` in `post_cmds1` 

so the config looks:

    set pre_cmds1(myrouter) configure
    set cmds1(myrouter) {"set interfaces xe-3/1/0 disable" commit "SLEEP 3" rollback 1 "show | compare" commit}
    set post_cmds1(myrouter) exit

The 2nd command list looks too long, you can use a continuation sign `\` to cut
it shorter and make it one command per line.

    set pre_cmds1(myrouter) configure
    set cmds1(myrouter)         {
        "set interfaces xe-3/1/0 disable"
        "commit"
        "SLEEP 3"
        "rollback 1"
        "show | compare"
        "commit"
    }
    set post_cmds1(myrouter) exit

if you would like to do the same test often, but may need to change some
parameters to different values sometime, you can use "variable substitutions"
in your commands. This can be done via the `list` command, with which you can
includes variables in commands, the format is:

    [list "command1" "commands2 with $param", "command3" ...]

now the commands list can be written to this:

    set sleeptime 3
    set port "xe-3/1/0"

    set pre_cmds1(myrouter) configure

    set cmds1(myrouter)         [list   \
        "set interfaces $port disable"   \
        "commit"   \
        "SLEEP $sleeptime"   \
        "rollback 1"   \
        "show | compare"   \
        "commit"   \
    ]

    set post_cmds1(myrouter) exit

now run the `crtc` script and the above commands will be executed.

    ~pings/bin/crtc myrouter

Just changing the varible "sleeptime" and "port" if different values are needed
for a new test.

the other internal arrays (`regex_info`, `issue_info`, `collect` and `testN`)
will be illustrated later.

=== user_patterns

TODO:

`user_patterns` array can be used to configure the script behavior when the
user defined messages appear. it can be used either under automation/batch mode
when crtc is still iterating the command sending, or under interact mode when
user get the control.

some considerations about the data structure designed:

* user_patterns(router) introduce complexity - every device need to
re-define the same
* use one list, but use 2nd element as an action, making it looks like
pattern action pair, so existing code may be used


the value could be:

* list of 1 pattern: 
  - a single-instance pattern: [list "ould not resolve"]
  - a multi-instance pattern:  [list "ould not resolve|bla bla"]

* list of 1 pattern + 1 action, where action can be:
  - a real command or string to send: [list {\(no\)} yes]
  - a special command:     
    *** RETRY:     [list "error: Backup RE not running" RETRY]
    *** RECONNECT: [list "Connection refused" RECONNECT]

* list of 1 pattern + 1 actions + optional attributes

  - set user_patterns(nsr_not_active) \

    [list "warning: GRES not configured" cmds3 interact]
          ------------------------------ ----- --------

          user defined pattern(s)              this applies to interact 
                                               also

                                         command
                                         command group or           
                                         special command:           
                                           RETRY RECONNECT EXIT ... 

TODO:  there might be "overlaps" between tests, e.g. below pattern was
configured for GRES, but it may appear in both GRES and ISSU:

    #set user_patterns(gres_success) [list "routing engine becomes the master"]

current workaround is to simply comment it out when doing ISSU test.


[WARNING]
====
this will have "performance impact" as it slow down the whole
command output process - this will be an issue only when there is VERY
long output, e.g.: 

   "show config | no-more"

in that case set attribute:

   -a "set enable_user_patterns 0"

====

== more usage examples

=== monitoring: keep sending cmd(s) in a loop

supposing you need to monitor some datapoints from a remote device, not one or
two shot, but periodically , or even continuously. keep repeating the command
manually is boring. the `-n` option specifies how many iterations you want to
repeat the same cmd(s), and `-i` specified the interval between iterations.
just like the way the windows `ping` works with its `-n` and `-i` options.

in a shell script, you might want to get the output of some command from the
remote device in realtime, and then contiue the shell script with the collected
data.  unless you are scripting directly within the router (.e.g. Junos unix
shell mode), this involves at least the following actions:

* login to the router (telnet/ssh , username, password, sprintboard, etc)
* send one or more commands (`-c`, or `-e` `-s`)
* repeat it 3 times, with 5s interval between each iteration (`-n` `-i`)
* collect the output for your analysis 
* trim other garbage texts (e.g username/pass/warning/annotation/etc) (`-H`)
* disconnect once above task done and run next shell command (`-q`)

here is how things can be done with this tool, in the form of a "one-liner":


.use `-n N -i INTERVAL` to monitor alarms
===============================================

    ping@ubuntu1404:~/bin$ crtc -c "show system alarm" -n 3 -i 5 -q -H alecto
    current log file ~/logs/alecto.log
    set cli timestamp

    Dec 01 13:30:32
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> iteration:1
    show system alarm 
    Dec 01 13:30:32s
    2 alarms currently active
    Alarm time               Class  Description
    2014-11-24 08:58:27 EST  Minor  PEM 1 Absent
    2014-11-24 08:58:27 EST  Minor  PEM 0 Absent

    {master}
    lab@alecto-re0> iteration:2
    show system alarm
    Dec 01 13:30:37s
    2 alarms currently active
    Alarm time               Class  Description
    2014-11-24 08:58:27 EST  Minor  PEM 1 Absent
    2014-11-24 08:58:27 EST  Minor  PEM 0 Absent

    {master}
    lab@alecto-re0> iteration:3
    show system alarm
    Dec 01 13:30:42s
    2 alarms currently active
    Alarm time               Class  Description
    2014-11-24 08:58:27 EST  Minor  PEM 1 Absent
    2014-11-24 08:58:27 EST  Minor  PEM 0 Absent

    {master}
    lab@alecto-re0> bye!:)
    ping@ubuntu1404:~/bin$

===============================================


.to monitor interface counter and ospf neighborship
===============================================
without `-q`, the session won't exit after the script complete all iterations.
you can continue typing commands manually.

    ping@ubuntu1404:~/bin$ crtc -c "show interfaces xe-3/1/0 | match \"input packets\"" -c "show ospf neighbor" -H -n 3 -i 5 alecto@jtac
    current log file ~/logs/alecto.log
    iteration:1
    set cli timestamp

    Dec 01 14:06:29
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> show interfaces xe-3/1/0 | match "input packets"
    Dec 01 14:06:29
        Input packets : 13790950

    {master}
    lab@alecto-re0> iteration:2
    show ospf neighbor
    Dec 01 14:06:29
    Address          Interface              State     ID               Pri  Dead
    10.192.0.41      xe-3/1/0.0             Full      192.168.0.6      128    37
    10.192.0.45      xe-4/1/0.0             Full      192.168.0.7      128    38

    {master}
    lab@alecto-re0> show interfaces xe-3/1/0 | match "input packets"
    Dec 01 14:06:34
        Input packets : 13790960

    {master}
    lab@alecto-re0> iteration:3
    show ospf neighbor
    Dec 01 14:06:34
    Address          Interface              State     ID               Pri  Dead
    10.192.0.41      xe-3/1/0.0             Full      192.168.0.6      128    32
    10.192.0.45      xe-4/1/0.0             Full      192.168.0.7      128    33

    {master}
    lab@alecto-re0> show interfaces xe-3/1/0 | match "input packets"
    Dec 01 14:06:40
        Input packets : 13790964

    {master}
    lab@alecto-re0> it's all yours now :)

===============================================

.`-m`

this can be shortened by -m option, the cmds will be sent endless time, with
whatever `interval` value configured in the config file. if the `interval` was
not configured then the default value `0` will be used, meaning to send as fast
as the router can response.

    set interval 0

login:

same as above, but just execute the cmd(s) endlessly.


=== group multiple options

crtc support compact option list, so the following 3 commands are the same:

    crtc -c "show system uptime" -q -H -n 3 -i 5 alecto@jtac
    crtc -c "show system uptime" -qHn 3 -i 5 alecto@jtac
    crtc -c "show system uptime" -qHn3 -i5 alecto@jtac

this makes the command options shorter to type.

****
I tried to make the options compatible as unix conventions (see <<X2>>), but I
don't find a good "existing parser" that I can just borrow, the "cmdline" package
doen't look good, plus I don't want to introduce any external dependencies - I
have to implement a cmdline parser myself.
****

////
this is fixed.

[NOTE]
===============================================

currently if the value of an option exceeds 1 digits, it should be separated
with the option name with at least 1 space. that said, this is NOT supported
(yet):

    crtc -c "show system uptime" -qHn30 -i5 alecto@jtac

but these are fine:

    crtc -c "show system uptime" -qHn 30 -i5 alecto@jtac
    crtc -c "show system uptime" -qHn3 -i5 alecto@jtac

===============================================
////

=== some "special" commands

[[GRES]]
==== `GRES` command (Junos platform specific)

when "sending" a "GRES" command, what `crtc` does under the scene is: 

* to interact with Junos router to perform a real JUNOS GRES operation. 
* reconnect to the router whenever connection got bounced.

This will be useful for GRES related test. 

TIP: The command "GRES" itself will never get sent literally to the router at
all.  

to request GRES to a dual-RE Junos device, simply use the "fake" `GRES` command:

    crtc -c GRES alecto

this will invoke Junos `request chassis routing-engine master switch` command and
"press" 'yes' to proceed the RE switchover procedure. the session will be
disconnected as expected , and the script will detect this and also exit .

."GRES" command will trigger GRES operation and make the crtc script exit
=====================================================

    ping@ubuntu1404:~/bin$ crtc -c GRES alecto
    current log file ~/logs/alecto.log
    telnet alecto.jtac-east.jnpr.net
    ping@ubuntu1404:~/bin$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re0> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re0> iteration:1
    set cli timestamp

    Dec 01 19:02:31
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0>
    Dec 01 19:02:31

    {master}
    lab@alecto-re0> request chassis routing-engine master switch
    Dec 01 19:02:31
    Toggle mastership between routing engines ? [yes,no] (no) yes
    Dec 01 19:02:32

    Resolving mastership...
    Complete. The other routing engine becomes the master.

    {backup}
    lab@alecto-re0> it's all yours now!:)
    exit
    Dec 01 19:02:32


    Connection closed by foreign host.

=====================================================

==== "GRES" with "persistent" session

if this is all what you wanted, that's fine. but being disconnected means you have
to restart the session again, and if the router is not ready yet, you will have
to wait for a while and retry - all these should/can be handled by the script: 

* detect if the remote device is reachable and 
* if yes, re-login automatically.
* if not, wait for while and keep detecting

    Feb 27 15:17:04
    Command aborted. Not ready for mastership switch, try after 229 secs.

    labroot@alecto-re0> [crtc:ops! sorry...will redo  230s later then...]
    [crtc:will count 230 seconds and retry...]

    <<<<count {200}s before proceeding...
    <<<<type anything to skip...
     
    <<<<count {30}s before proceeding...
    <<<<type anything to skip...


    labroot@alecto-re0> request chassis routing-engine master switch 
    Feb 27 15:20:54
    warning: Traffic will be interrupted while the PFE is re-initialized
    Toggle mastership between routing engines ? [yes,no] (no) yes 

    Feb 27 15:20:54
    Resolving mastership...
    Complete. The other routing engine becomes the master.
    will reconnect in 10s


."GRES" command with `-p` option will trigger GRES and then relogin the router automatically
=====================================================

    ping@ubuntu1404:~/bin$ crtc -pc GRES alecto                          
    current log file ~/logs/alecto.log
    telnet alecto.jtac-east.jnpr.net
    ping@ubuntu1404:~/bin$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re0> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re0> set cli timestamp

    Dec 01 19:41:36
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0>
    Dec 01 19:41:36

    {master}
    lab@alecto-re0> request chassis routing-engine master switch
    Dec 01 19:41:37
    Toggle mastership between routing engines ? [yes,no] (no) yes
    Dec 01 19:41:38

    Command aborted. Not ready for mastership switch, try after 11 secs.

    {master}
    lab@alecto-re0>
    Dec 01 19:41:50

    {master}
    lab@alecto-re0> request chassis routing-engine master switch
    Dec 01 19:41:50
    Toggle mastership between routing engines ? [yes,no] (no) yes
    Dec 01 19:41:51

    Resolving mastership...
    Complete. The other routing engine becomes the master.

    {backup}
    lab@alecto-re0> it's all yours now!:)
    exit
    Dec 01 19:41:51

    Connection closed by foreign host.

    will reconnect in 30s
    ping@ubuntu1404:~/bin$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re1 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re1> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re1> set cli timestamp

    Dec 01 19:42:24
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re1> 

=====================================================


there is a tricky part in Junos GRES - if you do it too frequent, say, do it
again in less than 4m after the previous execution, the router will reject the
request with a message. This tool can handle this situation well:


.hold and retry when GRES request was rejected
=====================================================

    ping@ubuntu1404:~$ crtc -c GRES alecto                                                                 
    current log file ~/att-lab-logs/alecto.log                                                             
    telnet alecto.jtac-east.jnpr.net                                                                       
    ping@ubuntu1404:~$ telnet alecto.jtac-east.jnpr.net                                                    
    Trying 172.19.161.100...                                                                               
    Connected to alecto.jtac-east.jnpr.net.                                                                
    Escape character is '^]'.                                                                              
                                                                                                           
    alecto-re0 (ttyp0)                                                                                     
                                                                                                           
    login: lab                                                                                             
    Password:                                                                                              
                                                                                                           
    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC                                  
    {master}                                                                                               
    lab@alecto-re0> set cli screen-width 300                                                               
    Screen width set to 300                                                                                
                                                                                                           
    {master}                                                                                               
    lab@alecto-re0> set cli timestamp                                                                      
                                                                                                           
    Nov 23 01:36:24                                                                                        
    CLI timestamp set to: %b %d %T                                                                         
                                                                                                           
    {master}                                                                                               
    lab@alecto-re0>                                                                                        
    Nov 23 01:36:24                                                                                        
                                                                                                           
    {master}                                                                                               
    lab@alecto-re0> request chassis routing-engine master switch                                           
    Nov 23 01:36:25                                                                                        
    Toggle mastership between routing engines ? [yes,no] (no) yes                                          
    Nov 23 01:36:26                                                                                        
                                                                                                           
    Command aborted. Not ready for mastership switch, try after 221 secs.                                       #<------
                                                                                                           
    {master}                                                                                               
    lab@alecto-re0> [Sun Nov 23 01:35:52 EST 2014::..[script:ops! sorry...will redo 221s later then...]..]      #<------
    Nov 23 01:42:50                                                                            
                                                                                               
    {master}                                                                                   
    lab@alecto-re0> request chassis routing-engine master switch                                                #<------
    Nov 23 01:42:50                                                                            
    Toggle mastership between routing engines ? [yes,no] (no) yes                              
    Nov 23 01:42:51                                                                            
                                                                                               
    Resolving mastership...                                                                    
    Complete. The other routing engine becomes the master.                                     
                                                                                               
    {backup}                                                                                   
    lab@alecto-re0> [Sun Nov 23 01:42:18 EST 2014::..[great! will exit current session...]..]  
    exit                                                                                       
    Nov 23 01:42:51                                                                            
    Connection closed by foreign host.                                                         
    [Sun Nov 23 01:42:18 EST 2014::..will reconnect in 30s..]                                  
    ping@ubuntu1404:~$ telnet alecto.jtac-east.jnpr.net                                        
    Trying 172.19.161.100...                                                                   
    Connected to alecto.jtac-east.jnpr.net.                                                    
    Escape character is '^]'.                                                                  
                                                                                               
    alecto-re1 (ttyp0)                                                                         
                                                                                               
    login: lab                                                                                 
    Password:                                                                                  
                                                                                               
    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC                      
    {master}                                                                                   
    lab@alecto-re1> set cli screen-width 300                                                   
    Screen width set to 300                                                                    
                                                                                               
    {master}                                                                                   
    lab@alecto-re1> set cli timestamp                                                          
                                                                                               
    Nov 23 01:43:24                                                                            
    CLI timestamp set to: %b %d %T                                                             
                                                                                               
    {master}                                                                                   
    lab@alecto-re1> [Sun Nov 23 01:42:50 EST 2014::..it's all yours now!:)..]                  
                                                                                               
    Nov 23 01:43:24                                                                            
                                                                                               
    {master}                                                                                   
    lab@alecto-re1>                                                                            

=====================================================

[NOTE]
====
* if you are trying to switchover too frequently (less than 4min?) , by default
Juniper router will reject the GRES request and pop out a complaining message.
the "GRES" procedure in this script currently can parse the message and find
out the right seconds to hold before next attemp.

====

==== repeat the "GRES" operation multiple iterations

the following command will perform GRES 10 times, with 300s intervals and
reconnect after 10s after each switchover.

=====================================================

    crtc -c GRES -pn3 -i 10 -r 10 alecto@jtac

=====================================================

this might be useful in the GRES related test/bug replications.

==== SLEEP command (platform-independent)

.config

NONE

.`SLEEP` command: set intervals between each command
=====================================================

sending a `SLEEP 10` "pseudo" command will actually send nothing to the remote
device, but just to slow down the next command with the specified number of
second. So this is useful when different intervals need to be set between each
command.

    ping@ubuntu1404:~/bin$ crtc -c "show system alarm" -c "show system uptime" -c "SLEEP 10" -c "show system alarm" alecto

    current log file ~/logs/alecto.log
    telnet alecto.jtac-east.jnpr.net
    ping@ubuntu1404:~/bin$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re1 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re1> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re1> set cli timestamp

    Dec 01 22:15:20
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re1> show system alarm
    Dec 01 22:15:21s
    3 alarms currently active
    Alarm time               Class  Description
    2014-12-01 19:42:06 EST  Minor  PEM 1 Absent
    2014-12-01 19:42:06 EST  Minor  PEM 0 Absent
    2014-12-01 19:41:55 EST  Minor  Backup RE Active

    {master}
    lab@alecto-re1> show system uptime
    Dec 01 22:15:21
    Current time: 2014-12-01 22:15:21 EST
    System booted: 2014-08-24 17:55:41 EDT (14w1d 05:19 ago)
    Protocols started: 2014-12-01 19:41:51 EST (02:33:30 ago)
    Last configured: 2014-12-01 19:37:52 EST (02:37:29 ago) by root
    10:15PM  up 99 days,  5:20, 1 user, load averages: 0.32, 0.22, 0.09

    {master}
    lab@alecto-re1> it's all yours now!:)
    show system alarm
    Dec 01 22:15:31s
    3 alarms currently active
    Alarm time               Class  Description
    2014-12-01 19:42:06 EST  Minor  PEM 1 Absent
    2014-12-01 19:42:06 EST  Minor  PEM 0 Absent
    2014-12-01 19:41:55 EST  Minor  Backup RE Active

    {master}
    lab@alecto-re1>
    Dec 01 22:15:31

    {master}
    lab@alecto-re1>

=====================================================

=== integration with other tools in shell

==== work with shell script

here is an example of using crtc in shell script:

.interact with remote device within shell
=================================================
file: printver.sh

    #!/bin/bash
    ver=`crtc -Hqc "show version | no-more" $1 | grep -i "base os boot" | awk '{print \$5}'`
    echo "router $1 is running software versoin: $ver"

run the shell script:

    ping@ubuntu1404:~/bin$ printver.sh alecto@jtac
    router alecto@jtac is running software versoin: [12.3-20140210_dev_x_123_att.0]

    ping@ubuntu1404:~/bin$ printver.sh tintin@jtac
    router alecto@jtac is running software versoin:  [12.3R3-S4.7]

=================================================

another good example is to "sweep" or "scan" a subnet and find out all
login-able junos device, then report the requested data:

.scan all routers in a subnet
=================================================
use `-w` to specify a relatively lower "patience" . This is the amount of time
the script is willing to wait for a response from the remote device after the
connnect attemp was started. with `-w 5`, if there is no response within 5s
(say, a prompt asking for username), the script will stop and quit.

this can be used to quickly probe all IPs within a subnet.

    for i in {1..254}
    do 
        echo "probing 172.19.161.$i ..."
        crtc -Hqac "show chassis hardware" -w 5 172.19.161.$i@jtac
    done

or a oneliner for short:

    for i in {1..254}; do crtc -Hqac "show chassis hardware" -w 5 172.19.161.$i@jtac; done;

=================================================


==== work with GNU screen/tmux

if you are familiar with any of the terminal "multiplexer" kind of tools (GNU
screen, tmux, byobu, etc), you can now run the crtc script within those tools.

with GNU screen/tmux , you can login to a server, run your program (crtc in
this case), then shutdown your own PC and go home, while the session/test keep
running in the server.

    [pings@svl-jtac-tool02 ~]$ screen -fn -t alecto ~pings/bin/crtc myrouter

now you don't need to remain the connection from your own PC (mostly windows)
to your unix server where `crtc` is running. 

`ctrl-d` to detech the screen session,

    [pings@svl-jtac-tool02 ~]$ screen -ls
    There is a screen on:
            11858.ttyp0.svl-jtac-tool02     (Detached)
    1 Socket in /tmp/screens/S-pings.

You can leverate all power from GNU screen to manage the sessons.


=== send log as email attachment

TODO

=== other misc options

* `-v`:     print version info

* `-l`:     print all currently configured hosts

* `-h`:     print some dummy help info

* `-d`:     enable `debug` option
+
in "debug mode" the script will throw out a lot more garbage info, and it's
mainly just for debugging/script developping purpose. 


=== activate features on the fly

login and start command automations (as shown earlier)

    pings@PINGS-X240:~$ crtc -c "show chassis alarms" -H -q -n 30 -i 10 alecto@jtac
    no file /etc/resolv.conf exists!
    log file: ~/logs/alecto@jtac.log
    start to login alecto@jtac ...

crtc will start sending commands (-c) right after successful login:

    execute cmds1 after login alecto@jtac!

          show chassis alarms
    Mar 13 23:24:39 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> iteration:2=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:39 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

after each iteration, it will count 10 seconds before proceeding:

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...


    timed out (10s) without user interuption...

       iteration:3=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:50 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
     you hit something ,escape sleep (10s)...
    iteration:4=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:52 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...

now, if you are monitoring the commands executions and wait to expedite it,
just hit SPACE or ENTER to interupt the waiting period, crtc will escape the
rest of the waiting seconds and execute the next iteration.

     you hit something ,escape sleep (10s)...
    iteration:5=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:53 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
    you hit something ,escape sleep (10s)...
    iteration:6=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:55 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
    you hit something ,escape sleep (10s)...
    iteration:7=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:56 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

then, if for some reason you want to temperarily suspend the automation and
type something else manually, just press ENTER 2 or 3 times quickly, the
automation will be "suspended" and the control will be returned back to you
now.

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
    you hit something ,escape sleep (10s)...
    iteration:8=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 13 23:24:57 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    you want to type sth here? go ahead...      #<------you typed quickly at least 2 ENTER
    it's all yours now!:)
     log file: ~/logs/alecto@jtac.log, ctrl-g to move to backgroud
     !t toggle local timestamp, !L peek log, !E to send log in email
     !h list configured hosts,  !? for more commands, !i for the full tutor doc

now the control is yours, you can type your other commands here:

    lab@alecto-re1> show system uptime  
    you have unfinished automations (stack 1)!
     press !R to continue, !r to restart, ^\ or !s to stop! 

    Mar 13 23:58:35 
    Current time: 2015-03-13 23:58:35 EDT
    System booted: 2014-08-24 17:55:24 EDT (28w5d 06:03 ago)
    Protocols started: 2015-03-01 23:50:33 EST (1w4d 23:08 ago)
    Last configured: 2015-03-08 22:54:00 EDT (5d 01:04 ago) by lab
    11:58PM  up 201 days,  6:03, 2 users, load averages: 0.02, 0.04, 0.00

after every commands or ENTER you typed, crtc will prompt you that you still
have some suspended automation task.

    lab@alecto-re1>  
    you have unfinished automations (stack 1)!
     press !R to continue, !r to restart, ^\ or !s to stop! 
    Mar 13 23:58:38 

    lab@alecto-re1>  

if you want to continue with whatever left off in the previous automation,
press `!R` to resume it:

    lab@alecto-re1> !R
      resuming previous automations ...
       show chassis alarms
    Mar 14 00:00:53 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
    timed out (10s) without user interuption...
    iteration:9=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 14 00:01:04 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

these operations can be repeated. For example, if you want to again suspend the
automation, quickly type some ENTER will do it:

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
    you hit something ,escape sleep (10s)...
    iteration:10=>alecto@jtac: 
    {show chassis alarms}
    you want to type sth here? go ahead...
    it's all yours now!:)
     log file: ~/logs/alecto@jtac.log, ctrl-g to move to backgroud
     !t toggle local timestamp, !L peek log, !E to send log in email
     !h list configured hosts,  !? for more commands, !i for the full tutor doc

    you have unfinished automations (stack 1)!
     press !R to continue, !r to restart, ^\ or !s to stop! chassis alarms
    Mar 14 00:01:06 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1>      
    you have unfinished automations (stack 1)!
     press !R to continue, !r to restart, ^\ or !s to stop! 
    Mar 14 00:05:17 

if you decide to stop the prevoius automation, press `!s` or `q`:

    lab@alecto-re1> !s - stop unfinished automations!
                                                   
    Mar 14 00:05:21 

    lab@alecto-re1>  
    Mar 14 00:05:22 

    lab@alecto-re1> !R
    there is no automation to continue...
                                                            
    Mar 14 00:05:27 

if you later want it back again, press `!r` to restart it:

    lab@alecto-re1> !r - to repeat the previous cmds1 executions ,right((y)es/(q)uit?[y])
    y
    show chassis alarms
    Mar 14 00:07:05 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> iteration:2=>alecto@jtac: 
    {show chassis alarms}
    show chassis alarms
    Mar 14 00:07:05 
    4 alarms currently active
    Alarm time               Class  Description
    2015-03-01 23:51:41 EST  Major  FPC 4 PIC 0 Failure
    2015-03-01 23:50:34 EST  Minor  PEM 1 Absent
    2015-03-01 23:50:34 EST  Minor  PEM 0 Absent
    2015-03-01 23:50:33 EST  Minor  Backup RE Active

    lab@alecto-re1> count {10}s before proceeding...
    type anything to interupt...
     
or if you want to start another new automation (with a different command), press `!n`
then answer the questions asked by `crtc`

    lab@alecto-re1> !n
    Enter the command array you configured:
        show system uptime
    Enter how many iterations you want to run: [current 1]    
        1000
    Enter intervals between each iteration: [current 0]
        5
    will iterate command [show system uptime] 1000 rounds with interval 5 between each iteration, (y)es/(n)o/(q)uit?
        y
    show system uptime

new automation will start then:

    Mar 15 00:52:24
    Current time: 2015-03-15 00:52:24 EDT
    System booted: 2014-08-24 17:55:24 EDT (28w6d 06:57 ago)
    Protocols started: 2015-03-01 23:50:33 EST (1w6d 00:01 ago)
    Last configured: 2015-03-08 22:54:00 EDT (6d 01:58 ago) by lab
    12:52AM  up 202 days,  6:57, 2 users, load averages: 0.00, 0.11, 0.13

    lab@alecto-re1> iteration:2=>alecto@jtac:
    {show system uptime}
    show system uptime
    Mar 15 00:52:25
    Current time: 2015-03-15 00:52:25 EDT
    System booted: 2014-08-24 17:55:24 EDT (28w6d 06:57 ago)
    Protocols started: 2015-03-01 23:50:33 EST (1w6d 00:01 ago)
    Last configured: 2015-03-08 22:54:00 EDT (6d 01:58 ago) by lab
    12:52AM  up 202 days,  6:57, 2 users, load averages: 0.00, 0.11, 0.13

    lab@alecto-re1> count {5}s before proceeding...
    type anything to interupt...
    qyou stopped the automation!

    Mar 15 00:52:30


=== running `crtc` without options/parameters

even if you don't need to login to any remote device, just run `crtc` without
any parameter will provide you some useful features. essentially what `crtc`
does is to spawn a local bash according to your server's default setting
(whatever in environment variable `$env(SHELL)`). you can work in it as if
nothing happened, plus you now get features like logging, command timestamping,
listing configured hosts, etc. whenever needed, you can start another crtc
instance to login to a remote device. the worst case - you won't lose anything.

    [pings@svl-jtac-tool02 ~]$ crtc
    current log file ~/logs/LOCALHOST.log
    it's all yours now!:)
    date

    pings@svl-jtac-tool02:~$ date
    Wed Dec  3 05:56:46 PST 2014
    pings@svl-jtac-tool02:~$            #<------you are now in a new shell

    pings@svl-jtac-tool02:~$ !l         #<------list hosts configured in your config file
    host list: qfx10@att LOCALHOST qfx11@att LOCALHOST@jtac vmx DT405JVPE alecto dt401-host vmx-vpfe qfx9@att LOCALHOST@att att-term-server2 e320-svl tintin myrouter

    pings@svl-jtac-tool02:~$
    [pings@svl-jtac-tool02 ~]$ pwd      #<------type your commands like normal
    /homes/pings
    pings@svl-jtac-tool02:~$ !t         #<------add timestamps
    timestamp 1
    pings@svl-jtac-tool02:~$ pwd
    Dec 03 06:12:23 2014(local)
    /homes/pings
    pings@svl-jtac-tool02:~$ crtc alecto@jtac   #<------call another crtc to login to a router
    Dec 03 06:00:51 2014(local)
    current log file ~/logs/alecto.log
    telnet -K  alecto.jtac-east.jnpr.net
    pings@svl-jtac-tool02:~$ telnet -K  alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login: lab
    Password:

    --- JUNOS 12.3-20140210_dev_x_123_att.0 built 2014-02-10 20:40:15 UTC
    {master}
    lab@alecto-re0> set cli screen-width 300
    Screen width set to 300

    {master}
    lab@alecto-re0> it's all yours now!:)
    set cli timestamp

    Dec 03 09:00:54
    CLI timestamp set to: %b %d %T

    {master}
    lab@alecto-re0> exit                #<------exit the remote router, this will also exit the locally spawned new shell
    Dec 03 09:14:26
    Connection closed
                     [pings@svl-jtac-tool02 ~]$

the last `exit` command showed above will end up with two things happen:
* exit the telnet session (initiated by the 2nd crtc instance)
* exit the the first spawned shell (initiated by the 1st crtc instance)

this is because by default the option `exit_sync` was set.
with this option the crtc script is able to detect the connection close event
and exit local script. if this is not what you prefered, this can be changed at
least with 3 ways:

* setting the config option `exit_sync` to `0`
* use `-K` command option when running crtc
* hit `!k` key after run crtc

==== start new automations AFTER the initial automation finishs

one possible scenario that this might be useful is that, if you can't automate
the login process, but would like to automate the command sending after
successful login. In a lot of (most) production networks a random token need to
be typed in (possibly along with some "PIN" numbers) in the middle of the login
steps.  You can start crtc without options, login remote routers manually, then
start the automation by typing one of the supported keystrokes (will add this
later), which will trigger the new automation process.

t.b.c. examples:

* start crtc without options

    ~pings/bin/crtc

* login to the device manually (start ssh, input token, pin, password, etc)

    $ssh user@sprintboard
    please read your token and input number displayed on it: *******
    please input your personal PIN: ****
    ###############
    ###!welcome!###
    ###############
    secured-router > 

* configure your new automations in the config file like this

    set cmds3(LOCALHOST)         [list      \
            ospf_commands                   \
            bgp_commands                    \
    ]

    set cmds3(LOCALHOST)         [list      \
            ospf_commands                   \
            system_commands                    \
    ]

    set ospf_commands(LOCALHOST) [list      \
        "show ospf neighbor"                \
        "show ospf interfaces"              \
    ]

    set system_commands(LOCALHOST) [list       \
        "show system uptime"                  \
        "show system alarms"             \
    ]

* now after you've worked on the session for a while and decide to start
  another automation to collect OSPF and BGP info from the router, just type
  `!c!`, literally . once crtc script see that, it will check `cmds3` and start
  to execute all configured commands in it.

[NOTE]
====
in this example the script was started without a session/host name, so a
"LOCALHOST" was used as the key (or "index") when composing the `cmds3` array.
for a session logged into a remote host, use the session name (always same as
host name) to compose the array.

command:

    ~pings/bin/crtc alecto@jtac

configuration:

    set cmds3(alecto@jtac)         [list      \
            ospf_commands                   \
            system_commands                    \
    ]

    set ospf_commands(alecto@jtac) [list      \
        "show ospf neighbor"                \
        "show ospf interfaces"              \
    ]

    set system_commands(alecto@jtac) [list       \
        "show system uptime"                  \
        "show system alarms"             \
    ]

to execute above configured commands, type:
    
    !c!

====

[[X6]]
==== organize commands with nested command groups

as you maybe already noticed, the commands can be defined in a "nested" fasion,
this make it easier to group your commands according to whatever criterias you
prefered. You can organize your commands to different "levels" or "layers" like
this:

    set cmds3(yourdevice)   [list   \
            routing_protocols       \
            hardware_check          \
            snmp_check              \
            traffic_monitor         \
    ]

    set routing_protocols   [list   \
            ospf_check              \
            bgp_check               \
            multicast_check         \
    ]

    set hardware_check [list        \
        show chassis hardware       \
        show chassis alarms         \
    ]                               \
                                     
    set snmp_check [list            \
        ......                      \
    ]                               \
                                     
    set traffic_monitor [list       \
        ......                      \
    ]                               \
                                     
    set ospf_check      [list       \
            show ospf neighbors     \
            show ospf interfaces    \
    ]                               \
                                     
    set bgp_check   [list           \
            show bgp summary        \
            show bgp neighbors      \
    ]                               \
                                     
    set multicast_check [list       \
            ...                     \
    ]                               \

    ......

=== commands "on the fly" (keymaps)

keystroke commands are the keys you typed in after the session control was
returned back to you, after the login and commands automations. Not like the in
GUI tools, where you can select the menus and buttons anytime to activate or
start some specific features, in crtc everything you typed in was read and
matched by the script itself, to compare with the pre-defined internal keybinds
,which was used as triggers to some features. footnote:[yes,the characters will
be delivered to the remote device only when the input does not match with any
of the pre-defined keybinding]

here are some of the useful keybings.

==== `!t` `!T` timestamp every commands

when hitting <<X5,`!t`>>, the timestamp feature can be toggled. when turned on,
this feature will send a local timestamp on every carrige return, same as what
Junos `set cli timestamp` knob does.

==== `!c!`, `!e!`

these keybinds can be used to <<X6,start new automations>>.


==== other key commands (t.b.c)

* `!a`:     toggle `auto_paging` option

* `!e`      edit config file

* `!l`      list currently configured hosts

* `!k`:     toggle `exit_sync` option

* `!o`:     toggle `login_only` option

* `!p`:     toggle `persistent` option

* `!q`:     toggle `nointeract` option (quick mode)

* `!t`:     toggle `timestamp` option

* `!u` or `!H`  toggle `hideinfo` option

* `!n`     t.b.c

the full list of currently supported keybindings can be displayed by `!?`:

    {master}
    lab@alecto-re0>
    Mar 13 23:24:57

    {master}
    lab@alecto-re0> !?key commands:
        !?  list all keystoke commands"
        !a  toggle auto_paging option"
        !c!  start commands group automation"
        !c?  same, but asking for array's name"
        !C  editing current config file"
        !d  toggle debug option"
        !e!  start expect-command pair group automation"
        !e?  same, but asking for array's name"
        !E  attach log in email and send to user"
        !h  print usage"
        !i  print detail tutor docs"
        !H  toggle hideinfo option"
        !k  toggle exit_sync option"
        !l  print host list"
        !L  peek the log file"
        !n  specify and repeat a command"
        !o  toggle login_only option"
        !p  toggle persistent option"
        !P  go to tclsh"
        !q  toggle nointeract option"
        !r  repeat the previous cmds1 cmds execution"
        !s  stop the remaining of previous automations"
        !t  toggle timestamp option"
        !T  toggle timestamp option"
        !v  print version/license info"
    Ctrl-g suspend the script"
    Ctrl-\(SIGQUIT) to stop the cmds1 automations"

    {master}
    lab@alecto-re0>

== license

This program is free software; you can redistribute it and/or     
modify it under the terms of the GNU General Public License       
as published by the Free Software Foundation; either version 2    
of the License, or (at your option) any later version.            
                                                                  
This program is distributed in the hope that it will be useful,   
but WITHOUT ANY WARRANTY; without even the implied warranty of    
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the     
GNU General Public License for more details.                      
                                                                  
You should have received a copy of the GNU General Public License 
along with this program; if not, write to the Free Software       
Foundation, Inc., 675 Mass Ave, Cambridge, MA 02139, USA.

ifdef::workflow[]

== script work flows

=== simple diagram
                                         
     +--------+      +-----+------+           +-----+------+
     |`start` +----->|login_info()+---------->|send cmdsN  |<---+
     +----+---+      +-----+------+           +-----+------+    |
                    (login to router)        /      .           |
                                       ------       .           |
             ........................./...          .           |
             .check issues:          v   .          .           |
             .       +------------+      .          .           |
             .       |regex_info  | \    .          .           |
             .       +-----+------+  |   .          .           |
             .             |         |   .          .           |
             .             |         \   .          .           |
             .             |          >  .          .           |
             .             |         /   .          v           |
             .             v         |   .          .           |
             .       +------------+  |   .          .           |
             .       |issue_info  |  /   .          .           |
             .       +------------+ /    .          .           |
             .                           .          .      +----------+
             ........................\....          .      |send testN|
                                      \_____        .      +----------+
                                            \       .           |
                                             \      .           |
                                              \     .           |
                                               v    .           |
                                                    v     'loop'|
                                                  -----         |
                                                 /`is` \        |
     +--------+      +-----+------+             / issue \       |
     |   end  +<-----|send collect+<-----------X detectedX------+
     +----+---+      +-----+------+       yes   \-------/  no


=== more stuff

     +--------+               +-----+------+    N         +-----+------+
     |`start` +-------------->|login_info  +------------->|exec_cmds   |<---+
     +----+---+               +-----+------+              +-----+-----++    |
                            /(login_only?)               /(regex_info)|     |
                          Y/                          Y /   exist?    |N    |
            myinteract<----                            /              |     |
                                                      /               v     |
                        "active_monitor"        ------         myinteract   |
                        ......................./...                         |
                        .check_flag           v   .                         |
                        .     +------------+      .                         |
                        .     |regex_info  | \    .                         |
                        .     +-----+------+  |   .                         |
                        .           |         |   .                         |
                        .           |         \   .                         |
                        .           |          >  .                         |
                        .           |         /   .                         |
                        .           v         |   .                         |
                        .     +------------+  |   .                         |
                        .     |issue_info  |  /   .                         |
                        .     +------------+ /    .                         |
                        .                         .               +----------+
                        ......................\....               |send testN|
                                               \                  +----------+
                                                \                           |
                                                 \                          |
                                                  \                         |
                                                   \                        |
                                                    \                       |
                                                     \                      |
                                                      \                     |
                                                       \                    |
                                                        \                   |
                                                         \                  |
                                                          \                 |
                                                           v          'loop'|
                                                              -----         |
                                                             /`is` \        |
     +--------+                     +-----+------+          / issue \       |
     |   end  +<------myinteract-<--|send collect+<--------X detectedX------+
     +----+---+                     +-----+------+    yes   \---?---/  no



=== main flow

    |main
    |
    |static code template:
    |
    |    init_template: general code
    |
    |    code_template: a general template for dynamic code: `code_alecto@jtac_2`
    |
    |update_code_template:
    |
    |    goal: base on code_template , using current regex_info/issue_info
    |       for each cmd, generate code vars :  
    |       ......
    |       `code_alecto@jtac_2`
    |       `code_alecto@jtac_3`
    |       ......
    |
    |    . substitute `code_template` based on:
    |
    |       regex_info
    |       issue_info
    |
    |    . assign code vars: `code_alecto@jtac_2`
    |
    |       this is not the complete code, 
    |       but will be the main part of code that do the real work
    |
    |login
    |
    |    foreach host in list
    |       spawn_login
    |
    |exec_cmds $host cmds5
    |
    |
    |    post-commands
    |
    |
    v
     
=== spawn_login

    |spawn
    |calculate hideinfo
    |do_pag
    |expect_compensation
    |calculate reconnect
    |return code process
    |
    |
    |
    v
     

=== exec_cmds

    |    calculate pa_pair
    |
    |    calc hideinfo
    |
    |    calc interval_cmds/min/max
    |
    |    sub proc to process return values
    |
    |    calc active_monitor
    |
    |    actions_code
    |        run EXEC_CMDGROUP
    |          calc num of cmd arr
    |          "groupN" if defined
    |          "group" otherwise
    |        EMAIL
    |        EMAIL_MAX
    |        EXIT_MAX
    |        EXIT
    |        EXIT_ALL
    |
    |    pre_cmds group
    |
    |    loop cmds groups
    |
    |        calc baseline
    |        formal iterations
    |         |
    |         |  calc sig STOP
    |         |
    |         |  do_pag
    |         |
    |         |
    |         |  calculate reconnnect
    |         |
    |         |  active_monitor
    |         |
    |         |  actions on hit
    |         |    |
    |         |    |  check_flag, foreach cmd
    |         |    |     |
    |         |    |     |global code var `code_alecto@jtac_2`
    |         |    |     |
    |         |    |     |static proc `cmds5_alecto_2` exist?
    |         |    |     |
    |         |    |     |     call hander to call proc, return 0 or 1
    |         |    |     |     save in result_flag(alecto@jtac,2)
    |         |    |     |
    |         |    |     |"dynamic" code var `code_alecto@jtac_2` defined?
    |         |    |     |
    |         |    |     |     substitute `init_template` and get `init_cmds5`
    |         |    |     |
    |         |    |     |         substitute `cmds1` with sth proper: `cmds5`
    |         |    |     |
    |         |    |     |     compose real proc definition string "handle_proc"
    |         |    |     |
    |         |    |     |         put `proc cmds5_alecto@jtac_2 {router cmd}`
    |         |    |     |         put `global ...`
    |         |    |     |         put `init_cmds5`
    |         |    |     |         put `code_alecto@jtac_2`
    |         |    |     |
    |         |    |     |     eval hander_proc 
    |         |    |     |         this will define proc `cmds5_alecto@jtac_2` 
    |         |    |     |
    |         |    |     |     call hander: `code_alecto@jtac_2`
    |         |    |     |
    |         |    |     |         this will execute the proc, return 0 or 1
    |         |    |     |
    |         |    |     |     save result in result_flag(alecto@jtac,2)
    |         |    |     |    
    |         |    |     |evaluate result_flag, return 0 or 1
    |         |    |     |actions on not hit
    |         |    |     |not active_monitor
    |         |    |     |reload data if needed
    |         |    |     |inter-cmds-mysleep

=== do_pag


    |#initial cr
    |#verify login_info, RETURN_DO_PA5_NO_PA
    |#sigtect: SLEEP_ALL/PEND_ALL/CONT_ALL
    |#evaluate/expand cmds list
    |#start iteration
    | |#addclock
    | |#get pattern/data
    | |#nested
    | |#non-nested
    | . |#GRES
    | . |#SLEEP
    | . |#UPGRADE
    | . |#SLEEP_ALL X
    | . |#GOBACKGROUND
    | . |#PEND_ALL
    | . |#CONT_ALL
    | . |#normal cmds
    | . . |#handle interupt
    | . . |#RETURN_EXPECT_USER_INTERUPT
    | . . |#normal returns
    | . . | else  ;
    |
    |


=== myexpect

    |#return:
    |#parallel
    |#sendfirst mode
    |#expect
    . |#if enable_user_patterns set
    . . |#generate dynamic expect code snippet
    . . . |#non empty
    . . . . |#only pattern exists
    . . . . |#other strings exists
    . . . . . |"RECONNECT" {   ;
    . . . . . |default {       ;
    . . . |#empty
    . . |#final expect code
    . . |#eval after subst final code
    . |#if enable_user_patterns not set
    . . |i $user_spawn_id                                       ;
    . . . |re [CONST $key_interact] {                                 ;
    . . |i $session                                              ;
    . . . |#user_patterns
    . . . . |nocase -re $pattern_broken_pipe {                      ;
    . . . . |nocase -re $pattern_connection_close_msg {             ;
    . . . . |re $pattern_not_resolve_msg {                          ;
    . . . . |re $pattern_console_msg {                              ;
    . . . . |nocase -re $pattern_connection_unable {                ;
    . . . |re "$pattern" {                                        ;
    . . . |re $pattern_more {                                      ;
    . . . |timeout {                                       ;
    . . . |eof {                                           ;
    . . . |full_buffer {                                   ;

=== where to add the reconnect features?

there are at least 4 places where the extra features (e.g, reliabitity -
persistent, reconnect, retry, etc) can be added:

* spawn_login
* exec_cmds
* do_pag
* myexpect

where is the best place to go?

.in spawn_login and exec_cmds?

the pros:

* treat login and command sending process differently
* keep low layer code simple ,neat, so fast and stable

the cons:

* need to control and manage return values well
* no good way to continue from where left off
* duplicate codes in two places

.in do_pag?

the pros:

* simplified implementation, centralized control of these features
* be able to continue from where left off

the cons:

* messy flows, may contain endless loops?: 

    do_pags -> spawn_login (-> do_pags)
    
.in myexpect?

the pros:

* solve issue from low level, more precise control

the cons:

* performance overhead


.the best compromise

* do_pag
* spawn_login(login) + do_pag(command sending)


=== pattern(event)/action(processor) mapping

.patterns(events)

* expect timeout
* pattern_connection_close_msg: 
* etc

.actions(processors)

* RECONNECT
* RETRY
* EXIT
* NONE
* CTRLC
* CLOSE
 
////
==== actions_on_timeout_cmd knob

knob to process expect timeout actions

.implementation

. myexpect to return timeout event
. do_pag to process it:

    |#proc do_pag                                        
    . |#initial cr                                       
    . |#verify login_info, RETURN_DO_PA5_NO_PA           
    . |#sigtect: SLEEP_ALL/PEND_ALL/CONT_ALL             
    . |#evaluate/expand cmds list                        
    . |#start walk-through                               
    . . |#addclock                                       
    . . |#get pattern/data                               
    . . |#nested cmd                                     
    . . |#non-nested cmd                                 
    . . . |#GRES                                         
    . . . |#SLEEP                                        
    . . . |#UPGRADE                                      
    . . . |#SLEEP_ALL X                                  
    . . . |#GOBACKGROUND                                 
    . . . |#PEND_ALL                                     
    . . . |#CONT_ALL                                     
    . . . |#normal cmds                                  
    . . . . |#myexpect in 1 while                        
    . . . . |#process return                             
    . . . . . |#interupt from expect                     
    . . . . . |#RECONNECT in user_patterns               
    . . . . . |#"success" returns                        
    . . . . . |#"RETURN_EXPECT_TIMEOUT                          #<------
    . . . . . . |if $during_login {      ;               
    . . . . . . |} elseif $during_cmd {  ;               
    . . . . . . . |#actions_on_timeout_cmd=="RETRY"             #<------
    . . . . . . . |#actions_on_timeout_cmd=="CTRLC"      
    . . . . . . . |#actions_on_timeout_cmd=="EXIT"       
    . . . . . . . |#actions_on_timeout_cmd=="CLOSE"      
    . . . . . . . |#actions_on_timeout_cmd=="CONTINUE"   
    . . . . . . . |#actions_on_timeout_cmd=="NONE"       
    . . . . . . . |#else                                 
    . . . . . |#all other conditions                     
    . . . . |#fill cmd_output_array                      

.algorithm

. send double ctrl-c to escape from current hanging status (by interupting
  previous command)
. check escape status by expecting the prompt pattern
. if escaped successfully (indicated by seeing the prompt pattern)
  - repeat the same cmd (by continue the myexpect that was in a while loop) 
. otherwise (timeout again)
  - send ctrl-c again and repeat 2

    if {$actions_on_timeout_cmd=="RETRY"} {  
        
        #press a double "ctrl-c" to interupt
        exp_send -i $session "[CONST CTRL_C]"
        exp_send -i $session "[CONST CTRL_C]"

        #if not under pa pair mode (just send cmds)
        #wait for the prompt to appear before proceeding
        #if get timeout again, then send ctrl-c again,
        #and repeat this until we get a prompt!
        if !$pa_pair {
            expect {
                -i $session $pattern {
                    continue
                }
                timeout {
                    exp_send -i $session "[CONST CTRL_C]"
                    exp_continue
                }
            }
        }

.scope

per current implementation this only applies to during command sending
automations ("during_cmd"), not during login.

////

=== kibitz































































=== myinteractcmd

.structure

    proc myinteract -> 
    eval $myinteractcmd ->
    subst -nocommands
        $myinteract_user_input_patterns
            set myinteract_user_input_patterns {       
                #seems better to monitor ingress flows instead of user typing?
                #timeout $anti_idle_timeout {       ;#{{{4}}}
                #    interact_timeout 
                #}
                -echo "!?" usage_inline             ;#{{{4}}}
                "!!" {send -i $process "!"}         ;#{{{4}}}
                {\\} {send -i $process "\\"}        ;#{{{4}}}
                -echo -reset -re "!a" { eval $interact_a }  ;#{{{4}}}
                ......
            }
        $myinteract_process_input_user_patterns
            append myinteract_process_input_user_patterns [subst -nocommands {
                ......
            }
        $myinteract_process_input_patterns_misc
        $myinteract_process_input_patterns_static
            set myinteract_process_input_patterns_static {
                timeout $anti_idle_timeout {        ;#{{{4}}}
                    #interact_timeout 
                    eval $interact_timeout
                }
                -nobuffer -re $pattern_connection_close_msg {  ;#{{{4}}}
                    #don't detect connection close msg in interact mode,to prevent
                    #these text from "show log message" invoke an reconnection
                    interact_connection_close; 
                    #puts "pattern_more now looks: $pattern_more"
                    return
                }
            }
        $myinteract_user_output_patterns

.source code:

    set myinteractcmd [subst -nocommands {\n\
        interact {\n\

            -input $user_spawn_id\n\ 
                $myinteract_user_input_patterns\n

            -output process_output\n\

            -input kibitz_list

                #for invitelocal, just detect eof(the other user exit)
                #directly, then clear kibitz_list (not to close it) 
                #this is to prevent spawn_id being closed and script will
                #bail out
                -iwrite eof {
                    myputs "detected eof from [set interact_out(spawn_id)]"
                    #lappend shell_list  [set interact_out(spawn_id)]
                    ldelete kibitz_list [set interact_out(spawn_id)]
                    ldelete kibitz_user_list\
                        [set sid2user([set interact_out(spawn_id)])]
                    set kibitz_list_ori [set kibitz_list]

                    send -i "[set user_spawn_id] [set kibitz_list]"\
                        "user [\
                            set sid2user([set interact_out(spawn_id)])\
                        ] disconnected!"
                }

                #for inviteremote, kibitz running in a new shell, so no way
                #to detect eof, so just monitor the kibitz shell prompt,
                #and determine if other user exit
                -iwrite -nobuffer -exact "$kibitz_shell_prompt" {

                    puts "detected -[set kibitz_shell_prompt]-"

                    #move back (from kibitz_shell list) into shell_list
                    lappend shell_list  [set interact_out(spawn_id)]
                    ldelete kibitz_list [set interact_out(spawn_id)]
                    ldelete kibitz_user_list\
                        [set sid2user([set interact_out(spawn_id)])]
                    set kibitz_list_ori [set kibitz_list]

                    send -i "[set user_spawn_id] [set kibitz_list]"\
                        "user [\
                            set sid2user([set interact_out(spawn_id)])\
                        ] disconnected!"

                    #kill shell if remote user exit, seems better to keep
                    #it?
                    #
                    #set a [set kibitz_list]
                    #exec kill -9 [exp_pid -i [set kibitz_list]]
                    #close -i [set a]
                    #wait -i [set a]

                }

                #during kibitz, whenever another folk start typing, send a
                #notice to others indicating who is typing
                -iwrite -nobuffer -re "\r" {
                    set kibitz_list_backup [set kibitz_list]
                    set curr_sid [set interact_out(spawn_id)]
                    set curr_user [set sid2user([set curr_sid])]

                    if [set sid2typing([set curr_sid])] {
                    } else {
                        send -i\
                            "[set user_spawn_id]\
                            [ldelete kibitz_list_backup [set curr_sid]]"\
                            "\r\n<---[set curr_user] is typing ...\r\n"
                        set ownertyping 0
                        foreach asid [array name sid2typing] {
                            set sid2typing([set asid]) 0
                        }
                        set sid2typing([set curr_sid]) 1
                    }
                }

            -output process_output\n\


            -input process_input\n\
                $myinteract_process_input_user_patterns\n\
                $myinteract_process_input_patterns_misc\n\
                $myinteract_process_input_patterns_static\n\
            -output $user_spawn_id\n\
                $myinteract_user_output_patterns\n\
            -output kibitz_list\n\
        }\n\
    }]

=== NewHeadline

    #set user_patterns(login_retry_diff_account) 
    #   [list "ogin: $" "RECONNECT $reconnect_eval" interact_only]
    #          -------   -------------------------
    #            ^       ^ ------    -------------
    #            |       |      ^       ^
    #            |       |      |       |
    #            |       |      |       |
    #            |       | action_name user_action_eval
    #      user_pattern user_action  
    #      (event) 



example

    #yes/no {{{2}}}
    set user_patterns(answer_yes)   [list {\(no\)} yes]

    set user_patterns(ssh_yesno)    [list \
        {Are you sure you want to continue connecting \(yes/no\)?} yes \
    ]

    set user_patterns(auto_paging)  [list \
        {(--\(more \d+%\)--|--\(more\)--|--More--)}     \
        " "
    ]

== timeout RETRY code flow

=== do_pag

    while {1} {
        ......
        set myexpect_return [myexpect $router $pattern     \    #<------call myexpect first
            $datasent $pattern_timeout [expr !$pa_pair] 0   \
        ]       #<------"RETRY"

        ......

        if [info exists action_handler($myexpect_return)] {

            eval [set $action_handler($myexpect_return)]        #<------then excute action code

        } else {

            myputs "action not defined"

            break
        }

    }

=== action_handler(RETRY)

    set action_handler(RETRY) "code_return_retry" 

==== action code: code_return_retry

    set timeout 1
    expect -i $session -re ".+" {exp_continue -continue_timer}
    exp_send -i $session "[CONST CTRL_C]\r"
    if !$pa_pair {
        set retry_count 0
        expect {
            -i $session -re $pattern {
                myputs "expected pattern -$pattern- matched!"
                mysleep $reconnect_interval
                puts "retry now..."
                continue
            }
            timeout {
                if {$retry_count < $retry_max } {
                    incr retry_count
                    puts "press ctrl-c again to escape ..."
                    #just repeat sending ctrlc if got timeout
                    exp_send -i $session "[CONST CTRL_C]\r"
                    exp_continue
                } else {
                    puts "reach retry_max $retry_max! will exit"
                    exit
                }
            }
        }
    } else {
        exit
    }

retry code will be executed under context of do_pag


==== action code: code_return_reconnect

    global session

    if $persistent {

        puts "persistent set, will reconnect after ${reconnect_interval}s!"
        mysleep $reconnect_interval

        close ;wait 

        spawn_login $router     #<------call myexpect again

    } else {
        puts "persistent not set, not to reconnect !"
    }

reconnect code will be executed under context of do_pag

=== myexpect

    eval [subst $myexpectcmd]

==== myexpectcmd(_matchall) (global var)

    |set myexpectcmd_matchall {    ;
    . |i "$user_spawn_id" ;
    . . |re [CONST $key_interact] {   ;
    . . |re ".+" { ;
    . |i $session ;
    . . |re ".+" {  ;
    . . . |{$pattern} {        ;
    . . . . |if $isSendFirst {   ;
    . . . . |} else {    ;
    . . . |{$pattern_more} {   ;
    . . . |#expect_user_patterns
    . . . |default {           ;
    . . |#timeout_clause
    . . |eof {           ;
    . . |full_buffer {   ;

==== expect_user_patterns (build from main)
 
    $expect_user_patterns   ;#{{{5}}}

        "RETRY" {       ;#{{{6}}}

            myputs "will compose expect clause to return\
                user_action the value $user_action"
            #repeat the previous cmd and exp_continue
            append expect_user_patterns [subst {
                $pat_flag {$user_pattern} {
                    puts "detected -$user_pattern-"
                    puts "user_action configured as RETRY"
                    puts "will repeat same command and continue..."
                    return $user_action         #<------return "RETRY"

                }
            }]
        }

        "RECONNECT" {   ;#{{{6}}}

            #return "RECONNECT" literally, and the reconnection
            #action will be processed by upper caller
            myputs "will compose expect clause to return\
                user_action the value $user_action"
            append expect_user_patterns [subst {
                $pat_flag {$user_pattern} {
                    puts "detected -$user_pattern-"
                    puts "now eval user_action_eval..."
                    eval {$user_action_eval}    #<------
                    myputs "will return user_action $user_action"
                    return $user_action
                }
            }]
        }

endif::workflow[]

== appendix

.telnet client settings

as an expect/tcl script, crtc just call external client tools to login remote
device.  but there might be some subtle difference between host to host in
terms of the default settings of these clients software, and this might cause
some potential issues to the login automation. 

for example if the telnet protocol was configured to login, in some jtac server
you might notice this:

    [pings@svl-jtac-tool02 ~]$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp1)

    Password:

while typing the exact same commands from another server print generate
different interactions:

    ping@ubuntu1404:~$ telnet alecto.jtac-east.jnpr.net
    Trying 172.19.161.100...
    Connected to alecto.jtac-east.jnpr.net.
    Escape character is '^]'.

    alecto-re0 (ttyp0)

    login:

this makes the same login steps configured for same remote machine works from
one server, but may fail from the other.

    set login_info(myrouter)       [list               \
        "$"        "telnet alecto.$domain_suffix"    \  
        "login: "      "lab"                        \#<------expect a "login:" string
        "Password:"    "lab123"                   \
        ">"            "set cli screen-width 300"   \
        ">"            "set cli timestamp"          \
    ]

checking the telnet client default settings in both server reveals the cause:

    svl server                                                          my server
    ========================================================================================================================
    telnet> display                                                     telnet> display
    will flush output when sending interrupt characters.                will flush output when sending interrupt characters.
    won't send interrupt characters in urgent mode.                     won't send interrupt characters in urgent mode.
    will send login name and/or authentication information.             won't read the telnetrc files.
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    won't skip reading of ~/.telnetrc file.                             won't map carriage return on output.
    won't map carriage return on output.                                will recognize certain control characters.
    will recognize certain control characters.                          won't turn on socket level debugging.
    won't turn on socket level debugging.                               won't print hexadecimal representation of network traffic.
    won't print hexadecimal representation of network traffic.          won't print user readable output for "netdata".
    won't print user readable output for "netdata".                     won't show option processing.
    won't show option processing.                                       won't print hexadecimal representation of terminal traffic.
    won't print hexadecimal representation of terminal traffic.                                                                     
                                                                        
    echo            [^E]                                                echo            [^E]
    escape          [^]]                                                escape          [^]]
    rlogin          [off]                                               rlogin          [off]
    tracefile       "(standard output)"                                 tracefile       "(standard output)"
    flushoutput     [^O]                                                flushoutput     [^O]
    interrupt       [^C]                                                interrupt       [^C]
    quit            [^\]                                                quit            [^\]
    eof             [^D]                                                eof             [^D]
    erase           [^?]                                                erase           [^?]
    kill            [^U]                                                kill            [^U]
    lnext           [^V]                                                lnext           [^V]
    susp            [^Z]                                                susp            [^Z]
    reprint         [^R]                                                reprint         [^R]
    worderase       [^W]                                                worderase       [^W]
    start           [^Q]                                                start           [^Q]
    stop            [^S]                                                stop            [^S]
    forw1           [off]                                               forw1           [\377]
    forw2           [off]                                               forw2           [\377]
    ayt             [^T]                                                ayt             [^T]
                                                                        telnet>

the reason is that this svl server by default have the telnet `autologin`
feature enabled, and what that does is to send user-id of current user as the
telnet login name right after the initial telnet negotiation. this ends up with
only the password prompt given by the telnet server. `man telnet` tells more
detail crabs about this behavior:

    autologin     If the remote side supports the TELNET
                  AUTHENTICATION option telnet attempts to use it
                  to perform automatic authentication.  If the
                  AUTHENTICATION option is not supported, the
                  user's login name are propagated through the
                  TELNET ENVIRON option.  This command is the same
                  as specifying -a option on the open command.

`crtc` simply avoid this issue by always turning off the auto-login feature
(`telnet -K`), if the telnet was configured in the login_info array.

=== add an option

adding a new option or command line flag takes 3 steps:

* add a default value in `config_default`

    set config_default { 
        ......
        set options(verbose)                1
        ......
    }

* update optlist and optmap

    set optlist "-(a|A|b|B|c|C|d|D|e|E|f|F|g        \
                  |h|H|i|I|j|J|k|K|l|L|m|M|n|N      \
                  |o|O|p|P|q|Q|r|R|s|S|t|T|u|U      \
                  |V|v|w|W|x|X|y|Y|z|Z)"
                     ^
                     |
                     |

    array set optmap {                  \
        ......                          \
        "-v" verbose                    \
        ......                          \
    }

* update usage

    proc usage {} { 
        ......
        send_user "     -v  :verbose\n"
        ......
    }

=== update notes and todo list

.updates:

* (2014-12-06) updates: 
  - merge everything in one file: code, docs, config template. 
  - config file is now optional for the script.
  - `-H`, `!i` to display this tutor
* (2014-12-7) removed sensitive info and pushed to github
* (2014-12-12) updates:
  - auto_paging mode works, no need "show ...|no-more" when turned on
* install signal intercepters: to stop expect when needed
* to support auto detection to the user-defined "issues"

.todo:
* to provide a menu, easier hosts selection
* to write a unix man page
* add kibitz mode
* add automatic config scripts generation

[[X2]]
=== some interesting references:

* http://pubs.opengroup.org/onlinepubs/009695399/basedefs/xbd_chap12.html[utility conventions]
* https://www.gnu.org/prep/standards/standards.html#Command_002dLine-Interfaces[GNU standard]
* https://www.cs.tut.fi/~jkorpela/chars/c0.html[ascii control codes]

=== test set (for testing crtc dev only)

    crtc -c "show chassis alarms" -H -q -n 30 -i 2 alecto@jtac
    crtc -Hqi2c "show chassis alarms" -n 30 alecto@jtac
    crtc -b "configure" \
    crtc -E ">" -S "start shell" -E "%" -S "su" -E "sword" -S "Juniper" \
    crtc -b ">" -b "start shell" -b "%" -b "su" -b "Password:" -b "lab123" \
    crtc alecto@jtac
    crtc -n 3 -i 5 alecto@jtac
    crtc -c "show system uptime" -c "SLEEP 15"        \
    crtc -c "GRES" -Hqn 30 -i 300 alecto@jtac
    ver=`~pings/bin/crtc -Hqc "show version | no-more" alecto@jtac | grep -i "base os boot" | awk '{print \$5}'`
    ver=`~pings/bin/crtc -Hqc "show version | no-more" $1 | grep -i "base os boot" | awk '{print \$5}'`
    for i in {2..254}; do crtc -Hqac "show chassis hardware" -w 5 172.19.161.$i@jtac; done;
    crtc -Ht e320-svl

    crtc -yn 10 -i 20 \
      -E "$" -S "telnet alecto.jtac-east.jnpr.net" -E "login: " -S "lab" \
      -E "Password:" -S "lab123" -E ">" \
      -c "show interfaces ge-1/3/0 extensive | match pps" \
      -c "show vpls connections instance 13979:333601" \
      -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" \
      -R "2@@2601\s+rmt\s+LD" \
      -I "1@pps!=pps_prev" \
      -Y "show interfaces ge-1/3/0.601 extensive | no-more" \
      -N "configure" -N "run show system uptime" -N "exit" \
      a_new_router

    crtc -yn 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" 
        -I "1@pps!=pps_prev" anewrouter

    crtc -n 10 -i 20 -R "2@@2601\s+rmt\s+LD" anewrouter


    crtc -n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps"
    crtc -yn 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" 
    crtc -n 10 -i 20 -R "2@@2601\s+rmt\s+LD" anewrouter

    screen -fn -t alecto ~pings/bin/crtc alecto@jtac
    dislocate crtc -H alecto@jtac
     1  29571  Sat Apr 11 11:11:28  crtc -H alecto@jtac
     2  27225  Sat Apr 11 10:29:56  crtc rams@jtac

    crtc -c "show chassis alarms" -H -q -n 30 -i 10 alecto@jtac
    crtc -h cenos-vm1 centos-vm2 centos-vm3
    crtc -h cenos-vm{1..3}
    crtc -n3i5Pc "show system uptime" alecto@jtac tintin@jtac
    crtc -pr3tiUn 10000 -b "start shell" sonata@jtac 
    crtc -pc "show system uptime" -n 20 -i 5 -r 5 alecto@jtac
    crtc -d3n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps"
    crtc -E "$" -S "telnet alecto.jtac-east.jnpr.net" -E
    crtc -n 10 -i 20 -R "1@@Input  packets:\s+(\d+)\s+(\d+) pps@packets@pps" 
    crtc -n 10 -i 20 -R "2@2601\s+rmt\s+LD" anewrouter

ifdef::expectnote[]

=== TODO and DONE

////
#docs {{{1}}}
#TODO: {{{2}}}
#
#!!!STOP working on this!!! deadline Mar. 2016
#need to:
#1) finish a few important change/enhance
#2) document whatever 
#   implemented, 
#   learned tricks of expect, tcl
#then start to learn python/shell!
#       #<-----(Sat 20 Feb 2016 08:59:40 AM EST) -
#
#

to add: maintain CLI history, ctrl-P/N to call old/new cmd. handy when spanning
sessions

to fix:
same script, in lnx03 very slow, good on lnx01:

on lnx03(bad):

    ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    [Fri Jan 20 09:06:16 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    , 57<<<..
    [Fri Jan 20 09:06:16 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..
    [Fri Jan 20 09:06:16 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>, 25<<<.. 	#<------the problem
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    , 188<<<..
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..
    [Fri Jan 20 09:06:51 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>Warning: Permanently added ', 78<<<..0.141' (RSA) to the list of known hosts.
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..ng: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..

    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>
    , 2<<<..
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    <<<..
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    Password: [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>Password: , 10<<<..
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx03:~$
    pings@svl-jtac-lnx03:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    Password: <<<..
    [Fri Jan 20 09:06:53 PST 2017]:[netpbc@att]:spawn_login:..expected pattern -sword- captured..

on lnx01(good):

    ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    [Fri Jan 20 09:06:11 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    , 57<<<..
    [Fri Jan 20 09:06:11 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..
    [Fri Jan 20 09:06:11 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    pings@svl-jtac-lnx01:~$
    pings@svl-jtac-lnx01:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    [Fri Jan 20 09:06:12 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>pings@svl-jtac-lnx01:~$ 	#<------this is when it works
    pings@svl-jtac-lnx01:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    , 213<<<..
    [Fri Jan 20 09:06:12 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx01:~$
    pings@svl-jtac-lnx01:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    <<<..
    [Fri Jan 20 09:06:12 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    , 80<<<..
    [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx01:~$
    pings@svl-jtac-lnx01:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    <<<..
    [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..exp_continue on no match of ..+..
    Password: [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..get new strings:>>>Password: , 10<<<..
    [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..this make the buf looks:
    >>>ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    pings@svl-jtac-lnx01:~$
    pings@svl-jtac-lnx01:~$ ssh -o "StrictHostKeyChecking no" ps249h@155.174.70.141
    Warning: Permanently added '155.174.70.141' (RSA) to the list of known hosts.
    Password: <<<..
    [Fri Jan 20 09:06:13 PST 2017]:[netpbc@att]:spawn_login:..expected pattern -sword- captured..


to fix: kibitz local: when sending using -tty, still sent to wrong one
fixed.

to fix: under kibitz, no way to detect keystroke of remote user, how to switch to fast mode?

    * fix: detect keystroke -output process, then switch to fast mode
    * workaround: run crtc -Q0 under kibitz, to disable features.

to fix: currently with remoteinvite, when remote kibitz disconnect, no good way for
crtc to detect this via kibitz_list sid - the parent shell is still there, but
since no it starts to echo, there will be loops:

        user_spawn_id -> process_out  --> 
                        >                \
      +----------------/                  \
      |                                   /
      |  /kibitz_list <-  process-in   <-- 
      | / 
      ||user_spawn_id
      |\
      | ---> now is the new shell, will echo (along with error msg) \
      |                                                              \
      |                                                              /
      +-------------------------------------------------------------- 

the key is to detect disconnection:

* detect if old prompt appear?
* modify kibitz, print a message when/before close

to add: support glob, and multiple file operation, when uploading/download via !c

to fix: !c after that double spawn_id left, confusing
        #<------done

to automate:  (2016-10-06) 

    ping@ubuntu4:~$ ssh -o "StrictHostKeyChecking no" labroot@10.85.4.102
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the ECDSA key sent by the remote host is
    40:0b:7e:a7:30:a6:4f:f7:98:1d:ff:06:77:6a:08:3f.
    Please contact your system administrator.
    Add correct host key in /home/ping/.ssh/known_hosts to get rid of this message.
    Offending ECDSA key in /home/ping/.ssh/known_hosts:1
      remove with: ssh-keygen -f "/home/ping/.ssh/known_hosts" -R 10.85.4.102
    Password authentication is disabled to avoid man-in-the-middle attacks.
    Keyboard-interactive authentication is disabled to avoid man-in-the-middle attacks.
     Warning Notice

     contact CFTS ATT LAB team before using the setup. Purpose: AVPN-vPE
     Message: This is for vAVPN support ONLY!
    Permission denied (publickey,password,keyboard-interactive).

to add: send email when my account is in used by other folks

to enhance: when the invitation receiver is also running crtc (or other apps)
the invitation won't be seen. resolve by spawn -console ?

to fix: !K kibitz seems not working well when sharing with hongpeng 
    works for a while, the broke, but not detected
    need to improve disconnection detection!

to add: to allow user_patterns to be defined (and removed?) "on the fly"
        #<------a simpler/general workaround is to use !O - reload config?

    !u
    to add press 1/ delete press 2 ? 1
    input the event
    input the command/action when event triggered!

    to add 1/delete 2 ? 2
    input the event number: 5
    event deleted!

    !U
    to undo latest event (easy for internal pattern testing)

to enhance: add RETRY/RECONNECT count, currently endless not make sense

to enhance: running in background

to add: run@certain time - like simple crontab

to produce new usage case:
* scan attlab routers, daily, backup config/HW/SW version info

to add: per host attributes?
1. anti_idle_timeout/string for server/router are diff, 
2. etc

to enhance: for semi-automation.
in do_pag, if pattern matched, and action is "INTERACT", then call interact. 
and press !D to indicate done interaction and exit, return back to expect.
this will be useful for login process where a dynamic pass is needed.


to enhance: "pattern_more" , should also be configurable/customizable in
user_pattern

to enhance: protect some poor dynamic code from exit?

to improve: be compatible to expect 4.45 in jtac server

to add: support for multiple sessions to same router, in -h
    crtc -h a b a


to improve: add(or change to) router-independent action, so ukern_trace below
can be re-used by other router. currently need to define for each router

    #PIM log:
    set event_pim {rpd\[\d+\]: %DAEMON-5-RPD_PIM_NBRDOWN: Instance PIM\S+ PIM neighbor[^&]+removed due to:[^&]+ hold-time period}
    #BGP log:
    set event_bgp {rpd\[\d+\]: %DAEMON-4: bgp_hold_timeout:\d+: NOTIFICATION sent to [^&]+Hold Timer Expired Error}

    set ukern_trace(attga301me6@attlab)            {
        "start shell pfe network fpc0"
        "sh ukern_trace 45"           
        "sh ukern_trace 46"           
        "sh ukern_trace 47"           
        "sh ukern_trace 65"           
        "exit"
    }

    set eventscript(attga301me6@attlab)       [list      \
        "$event_pim"        "ukern_trace"      \
        "$event_bgp"        "ukern_trace"      \
    ]

to add: turn crtc as a "filter" if possible?

    echo "show version" | crtc myrouter | grep "junos"

current issue:

    error writing "stdout": bad file number
        while executing
    "puts "spawned process exited!" "

    you typed something here...type ESC if you want to interupt crtc...    
    : spawn id exp0 not open                                               
        while executing                                                    
    "expect {                                                              
            -i "exp0" ;#{{{4}}}                                            
                -re [CONST ESC] {   ;#{{{5}}}                              
                    myputs2 "session:\[myrouter\]:you typed ..."           
        ("eval" body line 5)                                               

    ping@ubuntu47-3:~$ crtc myrouter
    <<<CRTC:myrouter:start to login, please wait ...
    <<<CRTC:myrouter:to interupt the login process, press <ESC>!
    <<<CRTC:myrouter:to exit script(kill): press <ESC> and !Q
    can't read "tty_spawn_id": no such variable


to fix: coredump when:
    pings@svl-jtac-tool01:~$ crtc 10.144.10.9  

to add: an option to select message printing cmd: send_user/send_error/puts

to add: an option to logging:  (2016-05-08) 
* 0 (raw)       log_user
* 1 (clean)     expect_out and 
                interact_out

to add:
implement a vim-like keystroke commands: 
. interact mode is like "insert" mode, under interact/insert mode:
* press <esc> once, will send it to process as usual
psducode:

    interact 
        "esc"   #<------make it hold and not to send
            set timeout 0.3
            expect_user {
                "esc" {
                    set normal mode"            #<------another esc will move to normal mode
                }
                timeout {
                    send -i $process "esc"      #<------send if no further esc
                }
            }

. under normal mode, crtc hold any keys pressed by user, and use them as cmds
to CRTC itself:
e: send email
    s: edit email subject
    b: edit email body
    t: edit to list
    c: edit cc list
    B: edit bcc list
    ...

l: logging
    n: start a new log file
    s: ...
i: going back to "insert/interact" mode. keystrokes sent to process

but, maybe same can be done simply by !e !l , then continue with submenu, just
as of now?


to enhance:
force all "options" name to CRTC_xxx ?
this will bring a better control of what options we have...


#to start introducing crtc: script of the week - every week publish a crtc usage tips
in progress. considered as done.

#to complete doc about expect/tcl usage, for later reference
done



to fix version issue in jtac server

    working under:

        pings@svl-jtac-tool01:~$ /usr/local/bin/expect -v
        expect version 5.44.1.15

    not working under:

        pings@svl-jtac-tool01:~$ /volume/buildtools/bin/expect -v
        expect version 5.45

    1. 

        svl-jtac-tool01~[1] % ~pings/bin/crtc attga301me6@attlab
        <<<CRTC:attga301me6@attlab:start to login, please wait ...
        <<<CRTC:attga301me6@attlab:to interupt the login process, press <ESC>!
        <<<CRTC:attga301me6@attlab:to exit script(kill): press <ESC> and !Q
        bad index "0-2": must be integer or end?-integer?
            while executing
        "lindex $cmd_list $i-2"
            invoked from within
        "if $pa_pair {
                    set pattern_last    [lindex $cmd_list $i-2]
                    set pattern         [lindex $cmd_list $i]
                    set pattern_..."
            (procedure "do_pag" line 53)
            invoked from within
        "do_pag $login_index  login_info cmd_output_array_login_info  $interval_cmd $waittime_login"
            (procedure "spawn_login" line 21)
            invoked from within
        "spawn_login $login_index"
            ("foreach" body line 5)
            invoked from within
        "foreach login_index $hostlist_full {
            myputs2 "<<<CRTC:$login_index:start to login, please wait ...\n"
            myputs2 "<<<CRTC:$login_index:to interup..."
            (file "/homes/pings/bin/crtc" line 5259) 
        svl-jtac-tool01~[2] % echo $SHELL
        /bin/tcsh 

    2. 

        bad option "-matchvar": must be -exact, -glob, -regexp, or --
            while executing
        "switch -regexp -matchvar match -- [set buf] {
                            {} {        ;#{{{6}}}
                                myputs "expected pattern -- captured"
        ..."

    #<------done, seems resolved by forcing this in 1st line:
         #!/usr/local/bin/expect



to enhance:
    for all file open/puts, check if succeed, don't simply exit.
    a common issue is that script will error out when disk full
        #<------done (2016-06-19) 

to enhance:
    when user typing, use nofeature mode, 
    when user timeout, use feature mode
        #<------done (2016-06-16) 

to fix: configparser bug, below doesn't work:
    set nofeature 1
    if !$in_jserver {
        set nofeature 0
    }
the nofeature is always set to 0. need to make configparser only apply to
specific set of parameters, not all, e.g. to all CRTC_xxx, like CRTC_nofeature.
        #<------done (2016-06-16) 

kibitz related: (done)

    to add:integrate kibitz in
    need to read through kibitz source, then make it work in crtc

    to enhance: inviteremote:
        need to enhance session management
            1. dynamically create new shells
            2. dynamically switch between shells and original session(make it index 0?)
            3. after switch to a shell, able to start new kibitz
            #<------done (2016-06-12) 

    to enhance: use indirect spawn_id in interact

    to add: added kibitz!
            #<------basic done (2016-06-05) 

to add: allow user to define a key-binding mapping to a long command:
    !k
    input the key: \x
    input the command to map to: show abc def 123 a b c
    mapping created: !x ---- "show abc def 123 a b c"

--
to add: make crtc support semi-automation

    * when to stop automation, but ask the user's input to perform manual login, and
    * when to continue the automation

solution: use a special pattern or action as indicator that, when seen, 

    * stop doing from do_pag the normal myexpect, 
    * but to print the "pattern" as a question, and expect_user to answer as the
      action
    * match when user hint return, and continue to the rest of automation
        #<------done (2016-05-08) 
--

to enhance:
auto reattempt login with diff account
        #<------done (Fri 29 Apr 2016 03:28:24 PM EDT) 

to enhance:
"SLEEP" not working in pa_pair mode...
        #<------done (Fri 29 Apr 2016 03:27:43 PM EDT) 

to enhance:
iterate some commands , but have to change some parameters in it each time.

* generate some logs, need to use unique filename each time
  - use timestamp       #<------done
  - use a random num
  - use an increasing num


to enhance:
current with subst myexpectcmd, there could be an issue, when password include
\n, or any other \X, it will be substituted and wrong info will be sent to
device, need to find a way to avoid that.       #<------done


to enhance: able to insert router name or whatever, in each line of command output...
this make the log file grep friendly.
(Wed 20 Apr 2016 03:12:36 PM EDT) 
#<------done(Thu 21 Apr 2016 04:37:03 PM EDT)       

to enhance:

    pings@svl-jtac-tool01:~$ ssh -o "StrictHostKeyChecking no" jtac@12.3.167.13
    detected -timeout-
                      user_action configured as RETRY
                                                     will repeat same command and continue...
                                                                                             pressed a ctrl-c to escape just in case...
                              then expecting a prompt -sword-
                                                             ^C
    pings@svl-jtac-tool01:~$ detected -timeout-
                                               user_action configured as RETRY
                                                                              will repeat same command and continue...
             pressed a ctrl-c to escape just in case...
                                                       then expecting a prompt -sword-
                                                                                      ^C
    pings@svl-jtac-tool01:~$ detected -timeout-
                                               user_action configured as RETRY
                                                                              will repeat same command and continue...
             pressed a ctrl-c to escape just in case...

    solution: just retry a couple of time and exit or continue, instead
    of repeat endless

        #<------done (Wed 20 Apr 2016 03:13:09 PM EDT) 
        resolved in 2:
        1) add retry_max, by default just retry 3 times
        2) in pa_pair, repeat the datasent_prev and expect pattern_prev,
        instead of current
        not fully tested though.

to enhance: make the warning msg popping up only one time , or everything 10
return, not every time.

    jtac<<<< [session frpari2002me6@attlab]:you have unfinished  automations (stack 1)!
        press !R to  continue, ^\ or !s to stop,  !Q to quit script

to improve: make it work in crontab (2016-04-17) 
#crontab issue {{{2
##Date: Mon,  5 Oct 2015 01:51:02 -0400 (EDT)

#<<<start to login alecto@jtac ...
#<<<please wait...any keystroke will interupt the login process!
#stty: impossible in this context
#are you disconnected or in a batch, at, or cron script?Something seems to have gone wrong:
#Information about it: stty: ioctl(user): bad file number
#
#    while executing
#"stty raw"
#    ("eval" body line 1)
#    invoked from within
#"eval $cmd"
#stty: impossible in this context
#are you disconnected or in a batch, at, or cron script?Something seems to have gone wrong:
#Information about it: stty: ioctl(user): bad file number
#
#    while executing
#"stty raw"
#    ("eval" body line 1)
#    invoked from within
#"eval $cmd"
#ssh -o "StrictHostKeyChecking no " labroot@172.19.161.101

to fix: issue caused by recursive proc (spawn_login)

    login: Login attempt timed out after 120 seconds

    [Connection to frpari2002me6 closed by foreign host]
    |Connection to (d{1,3}.){3}d{1,3} closed|Connection to S+ closed|Connection reset by peer-

    <<<<count {10}s before proceeding...
    <<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
    too many nested evaluations (infinite loop?)
        while executing
    "getenv TCL_TZ"
        (procedure "GetSystemTimeZone" line 5)
        invoked from within
    "GetSystemTimeZone"
        (procedure "::tcl::clock::format" line 12)
        invoked from within
    "clock format [clock seconds]"
        (procedure "myputs" line 9)
        invoked from within
    "myputs "file $filename found for signalling""
        invoked from within
    "if [info exists filename] {
            myputs "file $filename found for signalling"
            return $filename
        } else {
            myputs "Cannot find a f..."
        (procedure "sigdetect" line 3)
        invoked from within
    "sigdetect "SLEEP_ALL""
        (procedure "do_pag" line 52)
        invoked from within
    "do_pag $login_index  login_info cmd_output_array_login_info  $interval_cmd $waittime_login"
        (procedure "spawn_login" line 53)
        invoked from within
    "spawn_login $router"
        invoked from within
    "if $persistent {

            mysleep $reconnect_interval

            #$session is a must, other wise have issues
            #
            #but this now report an e..."
        ("eval" body line 22)
        invoked from within
    "eval [set $action_handler($myexpect_return)]"
        invoked from within
    "if [info exists action_handler($myexpect_return)] {
                            myputs "action defined as  $action_handler($myexpect_return), eval it"

      ..."
        invoked from within
    "if {[regexp {GRES\s*(\d*)} $datasent -> interval_gres]} {
                    myputs "GRES command detected!"
                    if {[string equal $interval_..."
        (procedure "do_pag" line 166)
        invoked from within
    "do_pag $login_index  login_info cmd_output_array_login_info  $interval_cmd $waittime_login"
        (procedure "spawn_login" line 53)
        invoked from within
    "spawn_login $router"
        invoked from within
    "if $persistent {

            mysleep $reconnect_interval

            #$session is a must, other wise have issues
            #
            #but this now report an e..."
        ("eval" body line 22)
        invoked from within
    "eval [set $action_handler($myexpect_return)]"
        invoked from within
    "if [info exists action_handler($myexpect_return)] {
                            myputs "action defined as  $action_handler($myexpect_return), eval it"

      ..."
        invoked from within
    "if {[regexp {GRES\s*(\d*)} $datasent -> interval_gres]} {
                    myputs "GRES command detected!"
                    if {[string equal $interval_..."
        (procedure "do_pag" line 166)
        invoked from within
    "do_pag $login_index  login_info cmd_output_array_login_info  $interval_cmd $waittime_login"
        (procedure "spawn_login" line 53)
        invoked from within
    "spawn_login $router"
        invoked from within
    "if $persistent {



to fix: during `exec_cmd` automation, 
        press q to go to `myinteract`, 
        if matching `user_patterns` and needs a "RECONNECT" (login_spawn), 
        will fail...

exec_cmd 
    mysleep 
        expect, press q
            myinteract
                spawn_login

        juniper@chgil303ia2-PE24>                 
        [Connection to 192.168.112.24 closed
        |Connection to (d{1,3}.){3}d{1,3} closed|Connection to S+ closed|Connection reset by peer-!!
        now execute action group -RECONNECT-!!
        action is RECONNECT
        persistent mode set,  will reconnect in 10s

        <<<<count {10}s before proceeding...
        <<<<  "q":break(quit automation) "Q":exit the script " |\r" (or anything else): continue(escape sleeping)
        expect: spawn id exp9 not open from indirect variable (host2session(pe24@attlab))
            (write trace on "host2session(pe24@attlab)")
            invoked from within
        "set host2session($login_index) $spawn_id"
            (procedure "spawn_login" line 29)
            invoked from within
        "spawn_login $router"
            invoked from within
        "if $is_respawn {
                    set is_respawn 0
                    set sigquit 0
                    spawn_login $router
                }"
            (procedure "myinteract" line 372)
            invoked from within
        "myinteract $login_index"
            invoked from within
        "expect {

                #pattern_continue_automation {{{4}}}
                -i $user_spawn_id -re $pattern_continue_automation {
                    myputs2 "you hit somet..."
            (procedure "mysleep" line 31)
            invoked from within
        "mysleep $reconnect_interval"
            invoked from within
        "expect {
                        -i $session -re $pattern {
                            myputs "expected pattern -$pattern- matched!"
                            mysleep $rec..."
            invoked from within
        "if !$pa_pair {
                    expect {
                        -i $session -re $pattern {
                            myputs "expected pattern -$pattern- matched!"
             ..."
            invoked from within
        "if $during_login {      ;#{{{4}}}

                #this should have been covered in spawn_login
                #if reconnect_on_event==RETURN_EXPECT_TIMEOUT
               ..."
            ("eval" body line 10)
            invoked from within
        "eval [set $action_handler($myexpect_return)]"
            invoked from within
        "if [info exists action_handler($myexpect_return)] {
                                myputs "action defined as  $action_handler($myexpect_return), eval it"

          ..."
            invoked from within
        "if {[regexp {GRES\s*(\d*)} $datasent -> interval_gres]} {
                        myputs "GRES command detected!"
                        if {[string equal $interval_..."
            (procedure "do_pag" line 166)
            invoked from within
        "do_pag $login_index  $cmds cmd_output_array_$cmds  $interval_cmd $waittime_cmd $pa_pair"
            invoked from within
        "if [string equal [set ${cmds}($login_index)] ""] {
                    myputs "no $cmds!"
            } else {

                #calc baseline{{{4}}}
                #  only if issue..."
            (procedure "exec_cmds" line 300)
            invoked from within
        "exec_cmds $login_index $cmds"
            invoked from within
        "if !$auto_resolve {
                exec_cmds $login_index $cmds
            } else {
                #auto_resolve feature {{{3}}}
                myputs "auto_resolve set"

               ..."
            (procedure "exec_cmds2" line 16)
            invoked from within
        "exec_cmds2 $login_index $cmds"
            invoked from within
        "if ![string equal [set ${cmds}($login_index)] ""] {
                    #if ![string equal $cmds1($login_index) ""] {}
                        myputs "execute $cmds f..."
            ("foreach" body line 3)
            invoked from within
        "foreach login_index $hostlist_full {

                    if ![string equal [set ${cmds}($login_index)] ""] {
                    #if ![string equal $cmds1($login_ind..."
            invoked from within
        "if $login_only {
            myputs "login only!"
        } else {

            if {$parallel} {

                #print a prompt message to differentiate between multiple hosts and ..."
            (file "/home/ping/bin/crtc/crtc" line 8091)
        ping@ubuntu47-3:~/Dropbox/juniperblog$  
        ping@ubuntu47-3:~/Dropbox/juniperblog$  

to add: how to change "any" options on the fly?

    * !X : currently supported list, press "!?"
    * !a and then input "set whatever x", that option will be set
    * !I going to interpreter, display and set any options and then going back!

to fix: 
crtc -c "show config | no-more" myrouter will report issue and exit
        #<------done (2016-05-06) 

to add:
make use of send_tty + unix redirect, so when redirected to another user, a
local print is also available, very useful for team collabration.
an option "double_echo" can be added to support this.
        #<------done (Sat 07 May 2016 02:33:14 PM EDT) 


to add: start and end time.  (2016-04-09) 

    crtc -a "set startfrom 1am" -a "set endat 6am" -C ISSU abc@attlab

#to add: crtc in "reading mode" 
#- to read docs and substitute all new words
#- to print some lines on each return
#to add: reading-in-crtc mode. insert a line of texts for every <return>.
#
#
#to add:
#
#       support "user_pattern" in all p-a pairs: 
#
#       * login_info
#       * cmdsN
#
#       set login_info(myrouter) [list          \
#               "$"             "uptime"        \
#               "| \(no)\"      "yes"           \
#               ">"             "blabla"        \
#       ]
#       
#       in myexpect, when seeing "| xxx", will include xxx in dynamic expect
#       code
#
#       this will solve the issue of programming unpredictable event sequence.
#
#       #<------(Sat, Mar 19, 2016  6:37:14 PM) @cancun
#
#to add: 
#       support the user_patterns also in myinterfact - merge with eventscript
#
#to improve:
#   remove all extra cmdNs, 
#
#to improve: clarify/merge these knobs:
#   set exit_sync 0
#   set no_reconnect_on_interact 1
#
#to fix:
#
# unable to realloc 600000002 bytes and coredump
#
#to add: develope -h option, to run crtc as a syslog server
#       crtc -h router1@jtac router2@attlab router3@hoho
#
#to fix: script hangs while expect sth, can't terminate script
#
#
#to fix !c, currently buggy! make it useful and stable!
#  consider to use expect_user, instead of [gets stdin]
#
#to add: auto try with diff account. 
#  * looks very useful with att login
#  * no need to change config back and forth
#  * make use of env var?
#
#to add: windows support
#
#to add: able to kill all other crtc when an issue detected
#
#to add: detect file change, and read the cmd from it, write the output into it
#  this way I can start to work on python or shell, but use crtc to handle
#  interaction
#  one way is to use the timeout under interact, to:
#  0. new option: server_mode
#  1. check file mtime
#  2. if changed, read the file content and start new spawn
#  3. send file content to the new spawned process
#  4. return back and monitor again
#
#to add: LOAD command, to :
#  read from a junos config file, 
#  compose to a cmds
#  update the cmd no in regex_info and issue_info
#
#to add usage example:        
#   tcp port attacking test: in a loop spawn 10k http processes in 1s?
#
#
#to add: a special pattern "WIDE", to match common prompts in cmds or login_info
#
#
#to add: long options: --option1 -t --option2 ..
#  run out of options chars
#
#to add: quick stroke to disable anti_idle. this is useful when need to scroll
#up for a while
#
#to add: hide the pings@juniper.net
#       #<------done (Sun 24 Apr 2016 07:49:56 PM EDT) 
#       #use scan and format, ascii transforming +1
#
#to add: when retry, reload the config file
#
#to enhance doc: draw more flowchats:
#
#   spawn_login    -> do_pag
#   exec_cmds      -> do_pag
#   
#   do_pag -> myexpect
#   
#to add enhance doc: integration with screen script:
#   from shell, call screen, to start some crtc sessions
#   whenever need data, direct cmd to screen session, get the output, then detach
#   this will be nice to have
#
#
#to enhance:
#   enhance myputs to make it compatible with puts, and use it whenever needed
#
#to add: unique log files for multiple sessions to same host
#
#to add: don't display below during login process?
#  you have unfinished automations (stack  1)! press !R to continue, ^\ or  !s to stop
#
#to add: run from cron? how ? (2015-09-29) 
#
#to add: an options to ignore the commands in config file, but just use
#  commands configured in options?
#
#to add: make a good device selection list, group-able , like securecrt,
#otherwise too hard to select and can only search in config file
#
#to add: per device options
#
#to change: remove uname , use tcl internal vars to detect os, otherwise in
#  cygwin there might be crash
#to fix: reconnection on timeout in data doesn't work
#to fix: after reconnect from data, !R won't work well
#
#to fix: !c and input command group name, triple cmds were input
#
#to add: merge interact_c and interact_n, in a same wizard
#
#to test: the data capture feature, for graphic in third party soft
#
#to add: the captured vars, needs to be initialized, otherwise if no match in
#first hit, will exit out
#
#
# to add(good) : implement a "multiple instance" feature when you login to
# router abc, via telnet, you often want to upload/download some files to/from
# it, so enhance the "multiple session" feature, press \u to start another ftp
# session , to the same router, to automatically upload a file, or \d to
# download a file.
#
#      login_info1(abc) : telnet abc user lab password lab123 ...
#      login_info2(abc) : lftp abc user lab password lab123 put $filename ... 
#
# even better, you continue to work on your current session, and when the new
# session done file upload/download, you will be noticed by a message saying
# upload done, etc.
# 
# to add: timeout need to be bidirectional. only send anti_idle_string when
# there is ongoing no cmd output 
#
# to add: detect the actual timeout (sent, no response) and reconnect
#
#* to add: in login_info: (2015-04-23) 
#  "sword" "abc_IR"
#  "ogin" "jtac"
#  "sword" "abc"
# user the _IR keyword to return the control to interact
# then in interact, use some key (ctrl-q) ? to return back to where left off 
# in previous automation
#
#* todo: make good plans to all constant return/exit code
#
#* to add: "Write failed: Broken pipe" message when connection broken
#* to add: "Timeout, server 172.22.146.249 not responding"

#* to add, make -c work with -e/-s
#* to add, recursive crtc running: when pattern match, run another crtc -C
#* to add: in login_info, when put "ASK_password", will ask user to input
#  password, then proceed automations
#
#
#* more about cmd output manipulation: delta mode (display all increasement for
#digits chars?)
#* multi-session managment: maintain an "active" session list, detect which
#       session has problem and delete from "active" session list
#* to optimize/clean up code, option name, flags, exit code, return code, etc
#* to merge do_pags(recursive with send_cmds, or maybe do_pag?)
#* to fix: make -E/S/C and -e/s/c work together, (fixed)
#* put all global vars in an array, make code looks better, or use :: notation

#* to add: inline session management:
#       \s to add a new session
#       \q to quit current session
#       !Q to exit script               done
#        
#* to add: inline match and action
#* to add: add inline text filter: "private pipe" -> "cmd.. || match key"
#* to add: currently too much global, to merge into 1 array (options_cli, e.g)
#* to remove some trival debug info
#* to add: repeat the last (N) cmds
#* to add: autopage: make it to also work in automation mode
#* to add: when connecting, fast auto-retry , if timed-out with a configured interval
#          useful when doing switchover/rebooting testing
#* to add: built-in kibitz
#       doesn't work so far (2015-04-15) 
#* to fix: GRES function was broken now "crtc -pc GRES alecto"
#* to add: always give a message as a reminding when sleeping before next
#iteration.
#* to find a way to escape command like this:
#   cprod -A fpc0 -c 'set dc bc "getreg chg rdbgc0"'
#* to fix: persistent mode: need to clean previous (close wait?) before open new one
#
#* to add: generate config per template , and send (slowly) to the router
#       heard another script "script-it" did it for some specific config
#       templates,(2015-02-13) 
#
#* to fix: ctrl-c doesn't work under vim
#* to add: write a man page for/in the script: crtc man
#* to add: compose a message containing the test result when reproduced, also
#  consider to use as email body
#
#* (2014-12-25) add email sending(-E) with (zipped -Z) attachment, need to:
#       document
#       more consideration when to send when not
#* (2014-12-25) add UPGRADE fake command to support junos release upgrade, need to 
#       test (combine with all other normal cmds)
#       document
#(2014-12-26) added issuecheck functions, read from config
#(2014-12-26) issuecheck function is now parameterized, -R -S, 
#       to document 
#       -S currently auto attach $ sign to vars, need to make this optional:
#          otherwise -S "error=CRCERROR" will turn to "$error=$CRCERROR"
#    
#*(2014-12-28) to support "implicit" -R,
#     -c "show interfaces ...@regex1(\w+)regex2(\d)@var1@var2" -c "show vpls@var3@var4" -c ...
# also to extend it to support var substitution function?
#     -c "show interfaces ...@regex1(.*)regex2@snmpid" -c "show ...$snmpid" -c ...
#     some more considerations:
#     when no var name, use c1_1 c10_20: c_CMDNUM_VARNUM
#     var refer: \1 \2 /1 /2
#
#. parameterize host also       #<------hard, need to read and param config file
#                                       and read info like login_info, and eval
#                                       later , etc
#
#if some expect pattern not match (e.g. --more-- not matched), currently suspend forever...
#   need to handle "signal interception" and stop the expect , but without exit the script?
#. use persist_login
#. learn fork, multiple hosts support
#. find a good interact regexp for auto_paging
#. add alias
#. how to pipe to pager to print long messages?
#. extract ,structure, and eval exec config options
#. -H 2 : hide more
#. -t: add clock sync with remote
#* to support a pre-defined "stateful" cmd sequence
#* to provide a menu, easier known-host selection
#. interact a "eof" and call tclsh instead of exit script?

==== done: {{{2}}}

#
#to fix: (2016-04-04) exit won't trigger reconnect, but "request system logout
#.." will:
#  need to understand current implementation
#  and figure out how to debug interact
#       
#       #<------done (2016-04-09) 
#       #this was caused by an "open" regex: 
#
#       #this works reliably:
#       #   Connection closed.*\r|...
#       #this doesn't:
#       #   Connection closed.*|
#
#to improve: make crtc fully parameterized - whatever supported in config file,
#  can be done equally with command line options. this is important when later
#  called from other apps like in python. this is also how qemu/kvm works:
#  kvm CLI support tons of options, virsh use XML and conver to CLI options
#  maybe this can be done in crtc later...
#to improve: otherwise hard to debug
#   make user_pattern one time process, not in every myexpect
#       #<------done (2016-04-02) 
#
#to improve: 
#clarify/merge these knobs:
#   set reconnect_on_event     ""
#   set terminate_on_event     ""
#   set actions_on_timeout_login
#   set actions_on_timeout_cmd
#       #<------done, all deprecated. need more test (2016-04-02) 
#
#to add: make myexpect dynamic, currently static
#
#       -nocase -re $pattern_broken_pipe {
#           #"Write failed: Broken pipe"
#           myputs "detected -$pattern_broken_pipe- ... msg\r"
#           return "RETURN_EXPECT_BROKEN_PIPE"
#       }
#       -nocase -re $pattern_connection_close_msg {
#               ...
#       }
#       #<------partially done (2016-03-15) 
#       currently the feature is in myexpect, but need to add same dynamic
#       support in "myinterfact" code
#
#  
#to design a usage: with one crtc cmd, backup config files or any other files
#  from multiple devices, requires the random var
#  can be done via a shell script calling crtc?
#       <------ done
#to change: improve -t, make it configurable, to insert time output from
#target machine's command
#       #<------done?
#       
#to add: an arbitrary variable from command line that user can define their own
#usage, could be very useful!
# crtc -A "set filename myfile.conf" abc@upload
# crtc -A "filename=myfile.conf" abc@upload
#
#to add: default is not to run commands configured in config file, unless -M is
#  used    #<------done (2015-09-29) 
#
#diff output pattern match between 2 stages:
#  e.g: "Connection to nypjar2 closed by foreign host" should be expected only
#  in login stage, not during interactive mode when matched in "show log message"
#       #<------solved, added "no_reconnect_on_interact" option
#
#to add: a big feature: make crtc programmable, like below:
#       #<------not done, workaround by alternatives
#    set regex_info(nypjar2@attlab) {
#        {1@@Count: (\d+) lines@line1}
#        {2@@Count: (\d+) lines@line2}
#        {3@@On (primary)@yesno}
#    }
#    
#    set issue_info(nypjar2@attlab) {
#        {1@line1!=50}
#        {2@line2!=50}
#    }
#    
#    #based on whether rlsq is in primary or backup, determine which slot to restart
#    global yesno
#    if {$yesno=="primary"} {
#        set slot 3
#        puts "yesno is primary, so set slot 3 for the next iteration"
#    } else {
#        set slot 4
#        puts "yesno is not primary, so set slot 4 for the next iteration"
#    }

#    set test(nypjar2@attlab) [list  \
#        collectsomeinfo \
#        "request chassis fpc restart slot $slot"    \
#        "SLEEP 240" \
#    ]
#
#  maybe need to read the config multiple time, extract one varible each time
#
#     
#to add: to also update regex_info and issue_info in reload_data proc   #<--done
#  without this read config during iteration will be not completely work
#
#to add: random var substitution:
#  %H: hostnamebgfngb 
#  %S: sessionname
#  %T: time
#  ......
#       #<------(2016-03-31) done
#
#to add: use attribute as options: like in asciidoc(tor) good!
#  -a "waittime_login 30" -a "abc=123"
#       #<------done (2016-03-31)
#
#to add: command output file: 
#       crtc -c "show version" -f "version.txt" \
#               -c "show chassis hardware | no-more" -f "hardware.txt" \
#               -c "show configuration | no-more" -f "%SESSION%_%DATE%.txt"
#               router@att
#       #<------(2016-03-31) done
#
#to add: local timestamp on automated command send
#       #<------already supported
#
#to doc: add "crtc rsync-scooby" as example of shell friendliness
#       #<------basically done, may need improvement
#
#to fix: TELNET SSH substitution issue.
#       #<------(2016-03-29) workarounded by a trick
#       SSH will be replaced to "ssh     " (5 spaces)
#       normally "ssh abc" the number of spaces will be less than 5 - unless
#       the user is too insane to be qualified as a crtc user :)
#
#to improve the automation interuption, resume. 
#currently there is no good way to take control from an ongoing automation
#hit enter will just skip the sleep, but not taking over. this needs to be
#improved.
#       #<------ done (Mon 07 Mar 2016 01:30:49 AM EST) 
#       #copy user interupt handler from do_pag into mysleep
#       #add a extra "enter" for pa_pair
#
#to add an option for local spawn only, this is useful to automate local tasks
#currently crtc requires login_info to be configured, except only these 2:
#       crtc
#       crtc LOCALHOST
#       #<------ done (Sun 06 Mar 2016 05:23:51 PM EST) 
#
#to diff between crtc LOCALHOST and crtc
#       #<------done (Fri 04 Mar 2016 04:32:06 PM EST) 
#       #global argc check if argc>1:
#       #1) in spawn_login before doing do_pag and 
#       #2) when calling exec_cmd
#
#to enhance: add a prefix to crtc specific shell environment var:
#
#   export CRTC_account_user=pings
#   export CRTC_account_pass=abc
#
#   this is to avoid confliction with existing shell env.
#       #<------done(Mon 29 Feb 2016 11:50:13 PM EST)  
#       #TODO: replace all other "source .." call, (only when there is an issue?)
#to add auto_resolve feature:
#  take an value from output of a privious command
#  and put in later command 
#       #<------basic done(Sat 20 Feb 2016 11:31:12 PM EST)  
#       use '%' to indicate a var, just to fool the config file (also tcl)
#       (will subst to $ internally)
#       no support "multiple captures" yet, seems hard and more work
#
#to fix: seems baseline not work even with regex exists. this leads to first
#issue detection always fail. because matchok1 will be no match, matchok2 match
#though.
#       #<------done(Sat 20 Feb 2016 11:28:41 PM EST)  
#       1. just copy cmd_output_cmds to _prev, AFTER sending cmd
#       2. loose the time_now and time_prev comparison
#
#to fix: 
#    #this doesn't works
#    ver=`crtc -Hqc "show version | no-more" myrouter | grep -i "base os boot" | awk '{print \$5}'`
#    #this works
#    ver=`./crtc.3 -Hqc "show version | no-more" myrouter | grep -i "base os boot" | awk '{print \$5}'`
#       #<------done
#
#       cause is sometime the output from login process were not
#       completely sucked , the sucking time is too short (1s). increase it to
#       2s looks the best compromision - all outputs got sucked, and no too much
#       delays. more sucking time will bring too much delays when login.
#
#   
#to add: command "REPEAT P N"
#       #<------done (Mon 01 Feb 2016 10:26:52 PM EST)  
#   so instead of:
#
#   set cmds1(NYCNY301ME7@attlab) [list                    \
#       "GRES"                                      \
#       "show bgp summary | match estab | count"    \
#       "SLEEP 4"                                   \
#       "show bgp summary | match estab | count"    \
#       "SLEEP 4"                                   \
#       ......220 more...
#       ]
#       
#   use:
#   
#   set cmds1(NYCNY301ME7@attlab) [list                    \
#       "GRES"                                      \
#       "show bgp summary | match estab | count"    \
#       "SLEEP 4"                                   \
#       "REPEAT 2 221"                             \
#       ]
#
#   the implementation is a little bit tricky: expand not only cmdsN, but also
#   expand issue_info and regex_info, and updates the cmd_no
#   also need to add global var cmdsN_exp, issue_info_exp, regex_info_exp to
#   save the expanded value and the rebuilt within reload_data ...
#       #<------done
#
#to add: merge junos event script mode into existing interact
#       #<------drop
#    maybe not a good idea: "monitor start message" may pop up anything
#    so a sperate interact dedicated to this maybe not bad
#
#to add: junos event script functions
#       #<------done(Sun 24 Jan 2016 11:55:06 PM EST)  
#
#to add: #make use of multiple session: 
#   crtc -h -c "configure" -c "save $routername-backup-$date.txt" -c "scp ..." xxx
#
#add start/stop/email logging feature(like in securecrt)
#       #<------done (2016-01-22) 
#
#implement consideration:
#   currently issue detection doesn't work with "expect-and-send" mode in
#   command sending (exec_cmds). it only works with "send-and-expect" mode. but
#   , instead of add that support, the use case is relatively less. and in case
#   needed, maybe changing the common prompt var (pattern_common_prompt) is an
#   much easier solution.
#       #<------worked around
#
#auto_resolve:
#  to test: work with "REPEAT" ?
#  to enhance: support and expand multiple captures? -- capture multiple and
#       matches in the output, like awk...
#
#  to enhance: auto_resolve to also support issue detection
#       #<------done (Sat 27 Feb 2016 11:35:07 PM EST) 
#
#  to enhance: what if not able to capture data for a var
#       #<------done (Sat 27 Feb 2016 11:35:36 PM EST) 
#       throw a warning, then 
#       exit or go myinteract
#
#improved user friendliness:    #<------(Sat 23 Jan 2016 08:49:41 PM EST)  
#   when a wrong(non-defined) session name was given, crtc will give more hint
#   and choose.
#
#to improve: -c now also depends on -j, same as cmdsN. need to make it
#independent of -j
#   done        #<------(2016-01-21) 
#
#to enhance: 
#   instead of hit "any" key to interupt the automation, hit a specified key,
#   like default an "escape \x1B" (looks the best)
#   or user defined -X "c", press c to interupt
#       #<------done
#
#
#to fix: the recursive, avoid doing it in login process.
#this doesn't work otherwise: 
# "test" is found to have another array defined with same name, go (wrongly)
# recursion...
#
#  set login_info(willi@jtac)       [list       \
#      "\\\$"         "ssh $juniperserver"           \
#      "sword"     $unixpass                       \
#      "\\\$"        "ssh test@willi.jtac-east.jnpr.net"    \
#      "Password:"    "test"              \
#      ">"            "set cli screen-width 300"   \
#      ">"            "set cli timestamp"          \
#       #<------fixed (2016-01-18) , by adding an exception in do_pags, not tested
#
#to add: only start log file after successful login? - to avoid generating
#useless log files
#       #<------done
#
#to add: detect and read environment if configured (username, password)
#   so some sensitive info can be configured in .profile also, no need in
#   crtc.conf
#   done: usage:
#   comment jtaclab_account , jtaclab_pass out from config file:
#       #set attlab_account "jtac"
#       #set attlab_pass "jnpr123"
#       #set attlab_account "abc"
#       #set attlab_pass "abc"
#, then assign the values from shell:
#       export attlab_account=jtac
#       export attlab_pass=abc
#       crtc nypjar2@attlab
#         #<------done
#
#to add: "request window title change" feature          
#   tested in at least secureCRT , tmux and cygwin
#       #<------(2015-09-29) done
#
#(Tue 18 Nov 2014 02:37:23 PM EST) :
#. add monitoring in "interact mode", once see "closed by remote host", #<------done
#   send "exit" or exit the script to close spawned process     #<------done
#. add jtaclab router console access data       #<------done
#. parameterize it (with cmdline?) (use in shell script)
#                                               #<------done (2014-11-19) 
#. allow compact option format: "-abcd VALUE"   #<------done (2014-11-30) 
#. inherate all default values in script (none config is allowed)       
#                                               #<------done
#. code review: core proc generic/small, create new proc to handle exceptions
#                                               #<------cancelled, same not easy
#. test it, make sure main features work        #<------mostly done
#. write a unix man page and a tutorial page    #<------partial done (no man page yet)
#* to support automating user-defined new data arrays from interact mode #<------done
#. add more quick keys under interactive mode, change config w/o exit   #<------done
#* to support compact options                   #<------done
#* github, license , etc        #<------done
#send_cmds (-c) might lost prompt chars for the 1st cmd #<------solved?
#* only use myexpect, not to use myexpect5 anymore      #<------done
#* (2014-12-25) add -f/-F to specify log file/folder name       #<------done
#
#full parameterization: -E "$" -S "telnet ..." -E "login" -S "lab"
#       -E "password" -S ...    #<------done (2014-12-27) 
#       to document
#
#* to remove the loop , and add line number support when match -R       #<------done
#* signal to interupt   #<------done
#* to add: need a key to repeat command line command tests      #<------done
#* to add: add timestamp triggered by cmd outputs returns?      #<------done
#* fixed -dn30 , now it is ok

# to add: hideinfo:     0 -> raw info
#                       1 -> hide login
#                       2 -> hide command output        #<------done
#   "cmd1 cmd2" -> hide output of these specific cmds?
# 
# to add: when to send email, a list    #<------done
# to do: fix -C issue, doesn't work well.       #<------done
# to do: draw a full flow picture , VERY IMPORTANT      #<------in progress
# to do: eval/reload from within a procedure does not work
# to add: better flow control about automation:
#   currently : q to skip waiting time
#   need to: c to go next iteration (skip rest cmds in the current iteration)
#            b to stop the automation
#            q to skip waiting time and go ahead
#            Q to exit script
#            ? and all other: print help
#
# to add: detecting "Write failed: Broken pipe"         #<------done
# to add: email notice whenever script is running, with the usage       #<------done
#
#* to add: issue triggered only if all condition met    #<------done, Feb 
#
#  config2 ... useful to automate conditions with crtc
#               #<------this is supported. done
#* to add, crtc -T task1 ..., run cmds configured in task1
#       #<------done, supported with -c ISSU
#* fix bug: this doesn't work when having 2 or more multi-option
#  crtc -dc "show system uptime" -PHh alecto@jtac rams@jtac 
#       #<------done
#
#* to add: session hold/release in myexpect for multihost support, done
#* fix bug: -c to multi-host doesn't work anymore, done

#* (2014-12-25) persistent mode:
#   now work in batch mode also: when disconnected during
#   execution of cmds1, will reconnect automatically and proceed with the
#   original task from where left off,          #<------done
#   need to:
#       document.
#       currently just retry once,  when "disconnected", need to cover other
#           conditions
#-i100 won't work, only -i 100, or -i9

////

=== known issue

configparser is a evil - it is convenient/quite way to dump config file content
into options_cfg array for later reference, but the implementation is buggy and
may cause a lot of trouble/concerns later.

with this in config: the login_jtac_server shows "[list" in options_cfg

    #this implementation has some potential issue...TODO/buggy
    #considering this:
    #   set login_jtac_server [list             \
    #           "$"     "$go_jtac_server"       \
    #           "sword" "$unixpass"             \
    #   ]
    #


with this, SKIP_retry always shows 3 in options_cfg!

    set reconnect_eval {
        ......
        global SKIP_retry1
        if ![info exists SKIP_retry1] {
            set attlab_account $attlab_account2
            set attlab_pass $attlab_pass2
            set SKIP_retry1 1
            puts "retry $SKIP_retry1 time!"
        } elseif {$SKIP_retry1==1} {
            set attlab_account $attlab_account3
            set attlab_pass $attlab_pass3
            set SKIP_retry1 2
            puts "retry $SKIP_retry1 time!"
        } elseif {$SKIP_retry1==2} {
            set attlab_account $attlab_account4
            set attlab_pass $attlab_pass4
            set SKIP_retry1 3
            puts "retry $SKIP_retry1 time!"
        } else {
            set attlab_account $attlab_account5
            set attlab_pass $attlab_pass5
            puts "SKIP_RETRY1 is $SKIP_retry1........................"
            unset SKIP_retry1
            puts "too much wrong login, will exit..."
        }
        ......
    }

currently workaround is to not dump options_cfg to opt, and options_cfg to
options,if option name looks "SKIP_".

= tcl notes

== tcl man pages

install tcl-doc package:

    sudo apt-get install tcl-doc

then:

    man lappend

for "overlapping man pages" - command that is same as other unix tools, e.g.
lsearch:

    ping@ubuntu47-3:~$ whatis lsearch
    lsearch (3)          - linear search of an array
    lsearch (3tcl)       - See if a list contains a particular element

then to view tcl's lsearch man page:

    man 3tcl lsearch

== tcl var

As a matter of fact, every string is a legal Tcl variable name.

    set -> abc
    set 1 abc
    ......

=== ""

* just serve to group strings
* quotes themselves are not part of the parameter

=== {}

* braces themselves are not part of the parameter
* to "group", AND to "defer" the evaluation
* used in "control structures" (while, for, foreach..) to defer the evaluation
  of vars in the body or conditions - very important/fundamental concept in
  TCL!

.example:

    while {$retry_count <= $retry_max} {
    }

.what happened inside `while`:

when "while" see the condition `$retry_count <= $retry_max`, since it is inside
of braces {}, the evaluation of vars $retry_count and $retry_max will be
"defered". So "while" still "see" literal `$retry_count <= $retry_max`, instead
of `0<=4`. otherwise `0<=4` will be true forever and while won't have chance to
stop.

and, since evaluation in braces got defered, `while`, and some other control
structures, will evaluate by itself.

same applies to `for`. `foreach`.

.example:

    #this won't work!:
    #while [regexp {%(\d)} $string_new -> num] {}
    #this works..
    while {[regexp {%(\d)} $string_new -> num]} { }

.if 

the above process may not apply to `if`.

`if` structure just evaluate the condition once. so the condition does not need
to be deferred in `if` => `{}` is not needed in that sense. but grouping is
still needed in some cases. 

if in the cases grouping is also not needed, `{}` can be omited!

    if $a {puts abc}


== string

Everything is a string in Tcl, (weird? or cool?)
but functions that expect a number (like expr)
will use that 'string' as an integer:

    % set str " 123 "
    123
    % set num [expr $str*2]
    246

=== string replace

* from original string `$init_template`, 
* search for a string `cmds1` , and
* replace it with value of var `$cmds`, 
* place the changed string to a var (name from evaluation of `init_$cmds`)


    regsub -all {cmds1} $init_template "$cmds" init_$cmds

    set value [string replace $value 0 0]

    regsub -all \
        {\$regex4onecmd} $code_template \
        "\{$regex4onecmd\}" temp

    regsub -all $word $issue4onecmd "\$$word" issue4onecmd
    regsub -all {\$issue4onecmd_holder} $temp "\{$issue4onecmd\}" temp

regsub with backreference:

    regsub -- {([^\.]*)\.c} file.c {cc -c & -o \1.o} ccCmd

scan $cmd_output, locate all braces and escape them:

    if {[regsub -all {([{}])} $cmd_output {\\\1} cmd_output] > 0} {
        myputs "substituted cmd_output now looks $cmd_output" 3
    }

    eval [subst {regexp {$regex} {$cmd_output} -> $vars}]]

this is sometime very useful and a must, otherwise if cmd_output contains
unmatched braces, the statements to be eval.ed will bail out with errors.

=== string repeat

    myputs2 "\n\n[string repeat "- " 10]"
    myputs2 "\n[string repeat "- " 20]\n"
    regsub {SSH} $value "ssh[string repeat " " 5]" newvalue

=== string trim

to remove a matching "substring"

    send -i $session [subst [string trim $anti_idle_string '"']]

=== string index

    set firstchar [string index $value 0]

=== an empty string

its hard to implement an "empty" string sometime..

these don't work:

    crtc -a 'set prefix_mark ""' -c "show version %T" myrouter
    crtc -a "set prefix_mark \"\"" -c "show version %T" myrouter

the final value of prefix_mark is two literal quotes: "", not empty.

workaround is to use `0` to indicate "nothing".

    crtc -a "set prefix_mark 0" -c "config" -c "save backup%H_%T" myrouter

=== scan

    if {[scan $buf "search %s" domainname] == 1} {
    }

=== format

very nice char - ctrl-char translation: P330

    spawn $env(SHELL)
    interact -re "/\\\\(.)" {
        scan $interact_out(l,string) %c i
        send [format %c [expr $i-96]]
    }


== list

=== lsearch

    if {[lsearch -exact $vars $word]!=-1} {
    set opt_ind [lsearch $p_argv2 $opt]
    set ix [lsearch -exact $list $value]
    set hostnum [lsearch [array names session2host] $session1]
    if {[lsearch -exact $list $item] != -1} {
    if {[lsearch -exact $list $item] != -1} {
    if {[lsearch "pings ping" $env(USER)] < 0} {
    if {[lsearch "pings ping" $env(USER)] < 0} {}
    if {[lsearch "pings ping" $env(USER)] < 0} {
    set hostlist_all_except_last [lrange $arglist [lsearch $arglist "-h"]+1 end]
    if {[lsearch $email_on_event "EMAIL_LOG_ON_LOGIN"]>=0} {


=== lrange

    set vars [lrange $regex_vars_list 3 end]
    set arglist [lrange $argv 0 end-1]
    set hostlist_all_except_last [lrange $arglist [lsearch $arglist "-h"]+1 end]
    [lrange $cmd_list $repeat_pos-$repeat_num $repeat_pos-1]

=== llengh

    set cout_llen [llength $cout_list]
    if {[llength $args]} {}

=== lindex

    set argvn [lindex $argv end] ;#get the last param



== array

    expect1.17> set aa1(1 2） 3                            
    wrong # args: should be "set varName ?newValue?"       
        while executing                                    
    "set aa1(1 2） 3"                                      

    expect1.15> set "aa1(1 2)" 3                           
    3                                                      
    expect1.16> parray aa1                                 
    aa1(1 2) = 3                                           

== control structures

=== switch

.switch is functionally same as `if else`, but in a more compact form.

    switch -exact -- $user_key {
        "i" {
            ......
        }
        "s" {
            ......
        }
        default {
            ......
        }
    }

    switch -exact -- $argvn {
        "-G" {              ;#{{{5}}}
        }
        "-H" {              ;#{{{5}}}
            system less "~/.crtc-tutor.txt" 
        }
        "-K" {              ;#{{{5}}}
        }
        default {           ;#{{{5}}}
            puts "unsupported parameter or need a session name:\'$argvn\'!"
            usage
        }
    }

    foreach {dash value} $arglist {
        switch -exact -- $dash {
            #-A/-b/-B      ;#{{{5}}}
            "-A" { eval $value }
            "-b" { lappend pre_cmds_cli($data_index) $value; }
            "-B" { lappend post_cmds_cli($data_index) $value; }
            #-e/-s/-c     ;#{{{5}}}
            #"-c" { lappend cmds1($data_index) $pattern_common_prompt $value; }
            "-e" -
            "-s" -
            "-c" { lappend cmds_cli($data_index) $value; }
        }
    }

NOTE: an annoying issue caused by braces in comment (again!):

    switch -exact -- "abc" {
        "blabla" {
        }
        default {
            #} "reversely" matched braces here {
            puts "this won't be executed!"
        }
    }

if you are lucky, you'll get below kind warning:

    extra switch pattern with no body

if you are not, it just ignore "default" clause, and won't warn anything!

    % switch -exact -- "abc" {
        "blabla" {
        }
        default {
            #}  send -i exp6 {show system uptime
            puts "this won't be executed!"
        }
    } 
    % 

    % switch -exact -- "abc" {
        "blabla" {
        }
        default {
            #} "reversely" - matched braces here {
            puts "this won't be executed!"
        }
    }
    %

same error will be triggered if leaving some comments and enf of each block:

    switch -exact -- $user_key {
        "q" {   ;#{{{4}}}
            #close; wait
        }

        default {       ;#{{{4}}}
            if $select_host {
            } else {
            }   ;#else
        }       ;#default
    }           ;#switch

.an example

this is wrong: trying to use a empty string, will always get a match. in below
code the "default" clause will never be examined.

    switch -regexp -- $action_name {
        "RECONNECT.*" {   ;#{{{5}}}
        ......
        }
        "EXIT.*" {   ;#{{{5}}}
            exit
        }
        "RETRY.*" {   ;#{{{5}}}
        }
        "CTRLC.*" {   ;#{{{5}}}
        }
        "CONTINUE.*" {   ;#{{{5}}}
        }
        "" {   ;#{{{5}}}
            send_user -- "no action name configured!"
        }
        default {       ;#{{{5}}}
            send_user -- "<<<will execute configured action\
                group -$action_name-!!"
            puts "\n\\\"$action_name\\\" looks a normal cmdgroup"
            puts "will execute"
            exec_cmds $router $action_name
        }
    }

=== -regexp -matchvar

http://www.tcl.tk/man/tcl8.5/TclCmd/switch.htm

this is handy:

    set bar "aa 11 22 bb abc def"
    switch -regexp -matchvar foo -- $bar {
        a(b*)c {
            puts "matched var food is -$foo-"
            puts "Found [string length [lindex $foo 1]] 'b's"
        }
        d(e*)f(g*)h {
            puts "matched var food is -$foo-"
            puts "Found [string length [lindex $foo 1]] 'e's and\
                [string length [lindex $foo 2]] 'g's"
        }
    }

matched string will be saved in a list named "foo", in the way similiar to
expect_out:

* first elments is the full matched string
* 2nd element is the info captured in the first parenthesis
* 3nd element is the info captured in the 2nd parenthesis
* so on so forth


result:

    matched var food is -abc b- 
    Found 1 'b's                


== array operation

* to backup/copy an array:

    array set regex_info_backup [array get regex_info]

* to assign value

    array set optmap {                   \
        "-a" attribute                   \
        "-A" auto_paging                 \
    }

== string vs. list

.everything is a string

.not everything is a list

list requires some certain format.

    tclsh> set y3 { a b "My name is "Goofy"" }
    a b "My name is "Goofy""

    tclsh> lindex $y3 2
    list element in quotes followed by "Goofy""" instead of space

There is nothing wrong with y3 as a string. However, it is not a list.

.braces are not part of list


.things are diff when looking as a string and list

    set y {a b {Hello world!}}
    set z {a [ { 1 } }

    % set y {a b {Hello world!}}             
    a b {Hello world!}                       
    % string length $y                       
    18                                       
    % llength $y                             
    3                                        

string y, has 18 chars as a string, and 3 elements as a list

.when list processing doesn't work easily, try string processing

    >set reconnect_eval {
        set login_info(myrouter) [list                  \
            "$" "telnet alecto-re0."      \
            "login: " "b"                  \
            "Password:" "b"                 \
            ">" "set cli screen-width 300"              \
            ">" "set cli timestamp"                     \
        ]
    }

    >set b [list "ogin" $reconnect_eval]

    ogin {
    set login_info(myrouter) [list                   "$" "telnet alecto-re0."       "login: " "b"                   "Passw
    ord:" "b"                  ">" "set cli screen-width 300"               ">" "set cli timestamp"                      ]
    }

    >set c [lrange $b 1 end]
    {
    set login_info(myrouter) [list                   "$" "telnet alecto-re0."       "login: " "b"                   "Passw
    ord:" "b"                  ">" "set cli screen-width 300"               ">" "set cli timestamp"                      ]
    }

    eval {$c}
    invalid command name "{
    set login_info(myrouter) [list                   "$" "telnet alecto-re0."       "login: " "b"                   "Passw
    ord:" "b"                  ">" "set cli screen-width 300"               ">" "set cli timestamp"                      ]
    }"
    while evaluating {eval {$c}}

    >set d [string trimright [string trimleft $c \{] \}]

       set login_info(myrouter) [list                   "$" "telnet alecto-re0."       "login: " "b"                   "Pa
    ssword:" "b"                  ">" "set cli screen-width 300"               ">" "set cli timestamp"
      ]
    >eval $d
    {$} {telnet alecto-re0.} {login: } b Password: b > {set cli screen-width 300} > {set cli timestamp}


=== split and join: string vs list

* split: split a string to a list
* join: join a list into a string


.split

    set cout_list_prev [split $cmd_output_prev "\n"]
    set regex_vars_list [split $regex4onecmd "@"]

tcl `split` and `join` is tricky: you split a string into a list then join them
back, you got a different string!


e.g, this won't work sometime

    set regex_info_resolve($login_index) \
        [join [list "1" $line_num $regex $the_vars] "@"]

this works better:

    set regex_info_resolve($login_index) \
        [list [join [list "1" $line_num $regex $the_vars] "@"]]

TODO

=== append vs. lappend

roughly, 

* `append` is string operation, and `lappend` is list operation
* append won't leave a space between the two elments, lappend will.

* append essential

    set var "var$string" 
    
    |
    |
    v

    append var "abc" "def"

* lappend essential

    set list "$list $newlist"

    |
    |
    v

    lappend list $newlist

* more examples    

    set line_new [append line_new "\n"]
    append buf [set expect_out(buffer)]
    append session_input_patterns [subst {
    append handler_proc "global init_$cmds\n"

    lappend list $array($name)
    lappend event_action_list "$user_pattern" "$user_action"



== list , concat ,eval

.list: preserve the original list (don't disassamble)

    tclsh8.6 [/opt/ActiveTcl-8.6/bin]concat a b "hello world" 
    a b hello world                                           
    tclsh8.6 [/opt/ActiveTcl-8.6/bin]list a b "hello world"   
    a b {hello world}                                         

.concat

group all elements of all lists into a new list - disassamble first level
sub-list and put them together as a new list. 

    tclsh8.6 [~]concat a b "Hello world"
    a b Hello world                     

    % concat a b {c d e} {f {g h}}
    a b c d e f {g h}

.eval

* process parameters exactly the same way as concat:
* treat all its arguments as list
    - The elements from all of the lists are used to form a new list that is
      interpreted as a command. 
    - The first element becomes the command name. 
    - The remaining elements become the arguments to the command.
* do $ and command substitution 
* use `list` in arguments if you don't want them to be broken up
+
compare:

    tclsh8.6 [~]eval append v2 {a b } {c {d e}}              
    abcd e                                                   
    tclsh8.6 [~]eval append v3 [list {a b}] [list {c {d e}}]
    a bc {d e}                                              

[verse, eval man page]
====
Eval  takes  one or more arguments, which together comprise a Tcl script
containing one or more  commands.  Eval concatenates all its arguments in the
same  fashion  as  the  concat  command,  passes  the  concatenated  string to
the Tcl interpreter recursively, and returns the result of  that evaluation (or
any error generated by it).  Note that the list command quotes sequences of
words in such a way that they are not further expanded by the eval command.                      

====

in Expect book, one example:

this is wrong:

    spawn [lrange $argv 1 end] ;# WRONG!

this is correct:

    eval spawn [lrange $argv 1 end]

reason:

    spawn [lrange $argv 1 end] => 
    spawn "sleep 5" =>
    "sleep 5" is ONE string as a whole, but there is no such unix command

    eval spawn [lrange $argv 1 end] =>
    eval spawn "sleep 5" =>
    concat {spawn "sleep 5"} and execute the code =>
    execute code: "spawn sleep 5"

== backslash

P92:
Just remember two rules:
1. Tel translates backslash sequences.
2. The pattern matcher treats backslashed characters as literals.
These rules are executed in order and only once per command.

== exec vs. system

.exec:
fork a unix process and execute it as a child-process

.system:
similiar to fork, but:

* no I/O redirection
* internally much faster
* process parameters like concat (like eval + exec) 

this works:

    append content "\nrunning expect: [exp_version] @ [exec which expect]"
    puts "$content"

result:

    running expect: 5.45 @ /usr/bin/expect

this doesn't (system does not redirect its output)

    append content "\nrunning expect: [exp_version] @ [system which expect]"
    puts "$content"

result:

    /usr/bin/expect
    running expect: 5.45 @

== subst flags

    % set abc {
     -nocase -re {\(no\)} {
         myputs "detected -\(no\)-"
         myputs "will send cmd -yes-, and continue expect"
         send -i [set session] {yes\r}
         mysleep 30
         exp_continue
     }
    }

     -nocase -re {\(no\)} {
         myputs "detected -\(no\)-"
         myputs "will send cmd -yes-, and continue expect"
         send -i [set session] {yes\r}
         mysleep 30
         exp_continue
     }

    % set session 123
    123
    % subst $abc

     -nocase -re {(no)} {
         myputs "detected -(no)-"
         myputs "will send cmd -yes-, and continue expect"
    }    send -i 123 {yes
         mysleep 30
         exp_continue
     }

    % subst -nobackslashes $abc

     -nocase -re {\(no\)} {
         myputs "detected -\(no\)-"
         myputs "will send cmd -yes-, and continue expect"
         send -i 123 {yes\r}
         mysleep 30
         exp_continue
     }

    % subst -nobackslashes -nocommands $abc

     -nocase -re {\(no\)} {
         myputs "detected -\(no\)-"
         myputs "will send cmd -yes-, and continue expect"
         send -i [set session] {yes\r}
         mysleep 30
         exp_continue
     }

    %

=== example

----
good one:subs backslashes {{{1}}}                                              1 bad one: no subs backslashes {{{1}}}
[Thu Apr 21 14:31:37 EDT 2016]:[myrouter]:spawn_login:..substituted      2 [Thu Apr 21 14:27:36 EDT 2016]:[myrouter]:spawn_login:..substit
    ;#{{{3}}}                                                            3      ;#{{{3}}}
+--  6 lines: set oldtimeout 200000                                  +   4 +--  6 lines: set oldtimeout 200000
                myputs2 "session:[myrouter]:you typed ESC key here..    10                 myputs2 "session:\[myrouter\]:you typed ESC key
                myputs2 "                                               11                 myputs2 "\nyou have the control now...\n"
you have the control now...                                          +  12 +-- 28 lines: mycatch "stty -raw"
"                                                                       40                                     send_user "==output for  my
+-- 28 lines: mycatch "stty -raw"                                    +  41 +-- 97 lines: ;
                                    send_user "==output for  myroute   138                                     send_user "\n[subst [clock 
"                                                                    + 139 +--  3 lines: } else {
+-- 97 lines: ;                                                        142                                 exp_send -i exp6 {~8N\vV.b\r} <1>
                                    send_user "                      + 143 +--165 lines: return "RETURN_EXPECT_SENDFIRST0_NORMAL"
[subst [clock format [clock seconds] -format $dateformat]](local) 
"                                                          
+--  3 lines: } else {                                     
}                               exp_send -i exp6 {~8N\vV.b      <2>
+--165 lines: return "RETURN_EXPECT_SENDFIRST0_NORMAL"     
[Thu Apr 21 14:31:37 EDT 2016]:[myrouter]:spawn_login:..get new stri   ~
----

<1> working example: "\r" got substituded - hence removed from the password char
<2> non-working example: "\r" retained because of subst `-nobackslashes` knob,
and so `\r` will be treated as part of password chars. this will run into issues.

== dynamic coding: eval + subst

=== example1

we have array cmds2 defined, along with other arrays with similiar name `cmdsN`:

    set cmds2(myrouter) [list   \
        "show chassis alarm"    \
        "show system uptime"    \
    ]

    set cmds3(myrouter) [list   \

    set cmds4(myrouter) [list   \

    set cmds5(myrouter) [list   \

    ...

now we want to refer the element of one of the array, but in a dynamic way
that, to which array we are referring to need to be a variable that can be
changed as needed in the run time, for that purpose you may create code like
this:

.this doen't work:

    foreach a_cmd $cmds($login_index)] {
        puts "get a cmd $a_cmd"
    }

`cmds(myrouter)` looks just like a new array, which is not defined, and also
not what we wanted

.this doesn't work:

    foreach a_cmd ${cmds}($login_index)] {
        puts "get a cmd $a_cmd"
    }

with `{}` the `cmds` will not associate with `(myrouter)` now, but:

. ${cmds} evaluate to cmds2
. we now get a wrong `foreach` statement:

    foreach a_cmd cmds2(myrouter)

here we are expecting to iterate in a "value" `$cmds2(myrouter)`, not a
variable name `cmds2(myrouter)`. 

.these doesn't work

    foreach a_cmd $[set cmds]($login_index)] {
        puts "get a cmd $a_cmd"
    }

    foreach a_cmd $${cmds}($login_index)] {
        puts "get a cmd $a_cmd"
    }

These seems to be right, but we'll end up with a literal string
`$cmds2(myrouter)`, not the value of variable `cmds2(myrouter)`

    get a cmd $cmds2(myrouter)

.this works:

    set cmds cmds2
    foreach a_cmd [set ${cmds}($login_index)] {
        puts "get a cmd $a_cmd"
    }

what it does:

. use `${cmds}()` to make it not look like an array `cmds(myrouter)`
. now we got `[set cmds2(myrouter)]`
. the `[]` will evaluate `set cmds2(myrouter)` expression, resulting in the
value of variable `cmds2(myrouter)`

another way is to use `subst` to evaluate the value of the array

    set cmds cmds2
    foreach a_cmd [subst $[set cmds]($login_index)] {
        puts "get a cmd $a_cmd"
    }

=== example2

this is wrong:

    set i 101
    set cmds${i}($login_index) $a_cmd_resolved
    myputs "get an array cmds$i: $cmds$i($login_index)"

the goal is to say: 

    get an array cmds101: "show system uptime"

but the `$cmds` look like a variable, which does not exist, and not what we
wanted. we need to:

. make `cmds` to group with `$i` and get `cmds101`
. get the value of `cmds101(myrouer)`: `$cmds101(myrouter)`

this workaround works:

    myputs "get an array cmds$i: [set [subst cmds$i]($login_index)]"


=== cautions with subst

considering this dynamic code:

    set pattern "(% |> |# |\\\$ |%|>|#|\\\$)$"

    set myexpectcmd {
        expect {            
            ......
            -re "$pattern" {                           
                myputs "expected pattern -$pattern_slash- captured"
                ......
            }
            ......
        }
    }

    puts "myexpectcmd looks:\n[subst $myexpectcmd]"
    eval [subst $myexpectcmd]

the original pattern will look:

    -re "(% |> |# |\\\$ |%|>|#|\\\$)$" {...}

`subst` will evaluate a regex one more time, so after `subst`, the orignial
expect statement `-re "$pattern"` will become:

    -re "(% |> |# |\$ |%|>|#|\$)$" {...}

and this will cause issues in the original expect code.


to solve this there are at least 2 solutions:

* to "compensate" some lost `\`
* protect with `{}` 

.this works

    if ![regsub -all {\\\$} $pattern {\\\\\$} pattern_slash] {
        set pattern_slash $pattern
    }

    set myexpectcmd {
        expect {            
            ......
            -re "$pattern_slash" {                           
                myputs "expected pattern -$pattern_slash- captured"
                ......
            }
            ......
        }
    }

so the original pattern now will appear the same when eval see it. to
illustrate the change:

    % set pattern "(% |> |# |\\\$ |%|>|#|\\\$)$"
    (% |> |# |\$ |%|>|#|\$)$
    % regsub -all {\\\$} $pattern {\\\\\$} pattern_slash
    2
    % set pattern_slash
    (% |> |# |\\\$ |%|>|#|\\\$)$

.this works

    set pattern "(% |> |# |\\\$ |%|>|#|\\\$)$"

or (not tested):

    set pattern {(% |> |# |\\$ |%|>|#|\\$)$}

and then:

    set myexpectcmd {
        expect {            
            ......
            -re {$pattern} {                           
                myputs "expected pattern -$pattern_slash- captured"
                ......
            }
            ......
        }
    }

=== cautions with eval

eval is always tricky, although it's powerful.

    set expectcmd {
        #puts "user_action is $user_action"
        send_user "$expect_out(buffer)\n"
        puts "user_action configured as \"RETRY\""
    }

    eval [subst $myexpectcmd]

there are at least 3 issues in above code:

* the `subst` will resolve all var-looking strings, even in comment
* the resolution may not work - it may contains unmatched braces that broke the
  whole statement `expectcmd`
* the `\` won't work as expected. 

quick solutions/workarounds:

* avoid having var in comments
* use one more nested var substitution to avoid being resolved.
* avoid using \

below codes work better:

    set send_user_expect_out {
        catch {send_user "$expect_out(buffer)\n"}
    }

    set expectcmd {
        $send_user_expect_out
        puts "user_action configured as RETRY"
    }

    eval [subst $myexpectcmd]

== continuation

. explicit continuation: a back slash at the end: `\` will be replaced by
one space
. implicit continuation: an open brace at the end

== tcl comment

Tcl comments behave a lot like commands. They can only be used where commands
can be used

.comment can't include unmatched braces

this is wrong:

    #this is a comment {
    set a "abc"

and this is wrong:

    #this is a comment }
    set a "abc"

.commment can't be appended behind interact

this is wrong:

    interact { #this is our interact
        ......
    }

this is correct:

    #this is our interact
    interact { 
        ......
    }

.even if the number of braces matched, but still this will cause issue: 

    if 1 {
    #myputs "will send cmd -show system uptime-, "
    #}  send -i exp6 {show system uptime        #<------
    #mysleep 30
    puts "this won't be executed"
    }

this is the error:

    wrong # args: extra words after "else" clause in "if" command

NOTE: this will cause the "default" clause in `switch` command being dropped
"silently"!

== tcl error processing: catch

.the simplest form

put tcl statement in a brace and catch, this will prevent the script from exit
on error.

    catch {
        #send_user "$expect_out(buffer)\n"
        #myputs "output looks -$expect_out(buffer)-"
        send_user "$output_new\n"
    }

    if [catch "spawn -noecho $env(SHELL)" reason] {
        ......
    }

    catch {exec "grep $login_index $config_file"}

.wrap with a proc:

    proc mycatch {cmd} {    ;#{{{2}}}
        if { [catch {eval $cmd} msg] } {
           puts "Something seems to have gone wrong:"
           puts "Information about it: $::errorInfo"
           return 1
        } else {
            return 0
        } 
    }

    mycatch "stty -raw"

== file operation

=== file command

    set h_debugfile [open $debugfile w]
    set file [open $file r]


    tclsh8.6 [~]set abc /home/ping/bin/crtc/crtc.conf                  
    /home/ping/bin/crtc/crtc.conf                                      

.rootname

    tclsh8.6 [~]file rootname $abc                                     
    /home/ping/bin/crtc/crtc                                           

.extension

    tclsh8.6 [~]file extension $abc                                    
    .conf                                                              

.dirname

    tclsh8.6 [~]file dirname $abc                                      
    /home/ping/bin/crtc                                                

.tail

    tclsh8.6 [~]file tail $abc                                         
    crtc.conf                                                          

.basename

    tclsh8.6 [~]exec basename $abc                                     
    crtc.conf                                                          


    tclsh8.6 [~]file rootname [file tail $abc]                         
    crtc                                                               

    tclsh8.6 [~]file executable $abc                                   
    0                                                                  

=== open/read/flush/fconfigure

.open

this is a very typical file open/read/close process...

    if [file exists $file] {
        set file [open $file r]
        while {[gets $file buf] != -1} {
            if {[scan $buf "search %s" domainname] == 1} {
                close $file
                return $domainname
            }
        }
        close $file
        error "no domain declaration in $file"
        return 0
    } else {
        puts "no file $file exists!"
        return 0
    }

another commonly used technique is to wrap open with catch

    if [catch {open $infile r} fp] {
        #sth wrong
        return 1
    } else {
        #use $fp to read ...
    }

.flush

force buffered data out
                                                                        
.gets

read a line at a time, and return length of strings read, or -1 when done

    while {[gets $file line] != -1} {
        # do something with $line
    }


.read

read a fix number of characters, or entire file if not specified.

    set chunk [read $file 100000]

.eof

return 0 when eof encountered
        

read 100k chars at a time, until end of file.

    while {![eof $file]} {
        set buffer [read $file 100000]
        # do something with $buffer
    }

read whole file in a buffer, then process each line

    foreach line [split [read $file] "\n"] {
        # do something with $line
    }

=== pipeline: work with existing unix command!

    tclsh8.6 [~/temp]echo "a1 100" > a1.txt                             
    tclsh8.6 [~/temp]echo "b1 200" >> a1.txt                            
    tclsh8.6 [~/temp]echo "a2 90" >> a1.txt                             
    tclsh8.6 [~/temp]cat a1.txt                                         
    a1 100                                                              
    b1 200                                                              
    a2 90                                                               
    tclsh8.6 [~/temp]set input [open "|sort a1.txt" r]                  
    file6                                                               
    tclsh8.6 [~/temp]read $input                                        
    a1 100                                                              
    a2 90                                                               
    b1 200                                                              

=== cd, pwd

== proc

P244, good sum of info sharing method between caller and proc:

. global
. return value
. `upvar a a` to bind var `a` in caller to var `a` in proc
. `upvar $a b` to "pass as reference"

.optional parameter

    proc myputs {msg {level 1} args} { 
    }



=== upvar: passing by reference (pointer)

    proc myproc {name1 name2} {
        upvar $name1 p1 $name2 p2
        set p1 1
        set p2 2
    }

    set n1 10
    set n2 20
    myproc n1 n2        #<------

    puts "n1,n2 now changed to $n1,$2"

result is:

    n1,n2 now changed to 1,2  

NOTE: it has to be `myproc n1 n2` - call by name. not `myproc $n1 $n2` - call by
value

                        VVVVVVVVVV VVVVVVVVVVVVVVVV
    proc do_pag {router cmds_array cmd_output_array {pa_intv 0} {pattern_timeout 120} {pa_pair 1}} { 

        upvar $cmds_array p_cmds_array
        upvar $cmd_output_array p_cmd_output_array
        ...
        set p_cmds_array(x) y
    }

    set cmds "cmds1"
    do_pag $login_index $cmds cmd_output_array_${cmds}_prev $interval_cmd $waittime_cmd $pa_pair]
                        ^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

this equals to:

    do_pag $login_index cmds1 cmd_output_array_cmds1_prev $interval_cmd $waittime_cmd $pa_pair]
                        ^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                        passing name, not value, to proc

=== upvar vs global

sometime I :

* need a variable to pass value across function calls 
* want to avoid to use function parameters.
* avoid using global, because the variable is not important in
the rest part of the code. However, 

    proc myinteract {router} {    
        ......
        set command ""
        ......
    }

    proc interact_c {router} {    

        ......
        upvar command command1
        upvar router router1
        upvar commandgroup commandgroup1
        global global_data; eval $global_data
        ......
    }

per P244: the two vars can have same name, so simply:

    set var 5
    proc setvar {} {
        upvar var var
        set var 10
    }
    setvar
    puts "$var"

this will print 10, not 5.

in `upvar var var`, first `var` belongs to caller's scope, while 2nd belongs to
process scope. 


=== upvar to associate var1 var to var2 name

suppose:

    >set a "b"        
    b                                                 
    >set b "bvalue"   
    bvalue                                            

print value of c:

    >set c [set $a]   
    bvalue                                            

same can be done:

    >upvar 0 $a x     
    >set c $x         
    bvalue                                            


    >set y $a         
    b                                                 
    >set c $y         
    b                                                 

=== uplevel

    proc exec_cmds2 {login_index {cmds "cmds1"}} {  
        uplevel {eval $update_code_template}
    }

    proc interact_connection_close {} {     
        uplevel {set is_respawn 1}
    }

to build a customized "return":

    proc myreturn {args} {         
        if {[llength $args]==0} {uplevel return}
        if {[llength $args]==1} {uplevel [list return [lindex $args 0]]}
        if {[llength $args]==2} {
            set funcname [lindex $args 1]
            puts "==> leaving $funcname now"
            uplevel [list return [lindex $args 0]]
        }
    }

the `list` is necessary, otherwise below code won't work:

    proc myreturn {args} {         
        ...
        if {[llength $args]==1} {uplevel {return [lindex $args 0]}}
        ...
    }

unfortunately this still doesn't work well. below proc `proc1` will actually
return 2, not 1.

    proc proc1 {} {
        myreturn 1 "proc1"
        return 2
    }



=== unknown

    tclsh8.6 [~]proc unknown args {
    >puts "abc"
    >}
    tclsh8.6 [~]aaa
    abc

    tclsh8.6 [~]aaaa
    abc

== argv

.example1: $argv/$argc/$argv0/$argv N

                 count is $argc
                 -------------list is $argv
    myscript.tcl param1 param2
    ------------ -----  ------
    $argv0       |      |
                 |      |[lindex argv 1]
                 |[lindex argv 0]

* '$argv0' as script name, 
* '$argv' as a list of parameters following script name
* '$argc' is size of list
* '[lindex $argv 0]' as 1st param,'[lindex $argv 1]',etc


== namespace

seesm no much usage for me.

== regexp and regsub

.capture all match, and return a list

    set issue_words [regexp -all -inline {(\w+)} $issue4onecmd]

.capture all "submatch"?

seems no "built-in" support on this. but not hard to get, based on the result
returned from above.

http://stackoverflow.com/questions/17176121/getting-values-for-all-submatches-alone-leaving-the-match-in-regexp-inline

    % set ospf 
     
    Feb 20 22:52:04
    Address          Interface              State     ID               Pri  Dead
    10.192.0.41      xe-3/1/0.0             Full      192.168.0.6      128    35
    10.192.0.45      xe-4/1/0.0             Full      192.168.0.7      128    33
    1.1.1.2          so-1/2/0.0             Full      100.100.100.100  128    39
    2.2.2.2          so-1/2/1.0             Full      20.20.20.20      128    34
     
    % regexp -all -inline {(\S+)\s+Full} $ospf 
    {xe-3/1/0.0             Full} xe-3/1/0.0 
    {xe-4/1/0.0             Full} xe-4/1/0.0 
    {so-1/2/0.0             Full} so-1/2/0.0 
    {so-1/2/1.0             Full} so-1/2/1.0


regexp and resub can be used together to accomplish sth like this:

    if [regexp {(ssh\s{1,4})\S} $value -> tobereplaced] {
        myputs "found pattern -$tobereplaced-" 3
        regsub "$tobereplaced" $value \
            {ssh -o "StrictHostKeyChecking no" } newvalue
        myputs "step -$value- became newstep $newvalue" 3
    }

* use a pattern to match
* but just to replace the sub-pattern, not the whole pattern

seems no good way to do same in one `regsub`


put regexp in a variable:

    eval [subst {regexp {$regex} {$cmd_output} -> $vars}]]


== to kill a telnet/ssh session

method1: to send a logout command, e.g, in Junos it's "exit"

    send -i $session "exit\r"

method2: use expect `close` and `wait` command, to close a spawned process

    catch {close $session;wait $session}


== logical operation

   if {[info exists options_cli(env_proof)] && \
            ( ($options_cli(env_proof) != 0) || \
             ![string equal $options_cli(env_proof) ""]\
            )
       } {
           puts abc
   }

=== string compare

this works:

    % if {1!=2} {puts abc}
    abc

This doesn't:

    % if {[1!=2]} {puts abc}
    invalid command name "1!=2"

To workaround use this:

    % if {![string equal 1 2]} {puts abc}
    abc

=== ternary operator

    set myexpectcmd [expr $slow_mode ? myexpectcmd_slow : myexpectcmd]


    expect1.4> set a 0
    0
    expect1.7> set b [expr $a?0:1]
    1
    expect1.8> set a 1
    1
    expect1.10> set b [expr $a?0:1]
    0

this doesn't work:

    expect1.6> set b [$a?1:0]
    invalid command name "0?1:0"
        while executing
    "$a?1:0"

== debugging

.myputs

.trace function call:

    #in the begging
    myputs "==>entering reload_data"

    #in the end
    myputs "==>leaving reload_data"

.trace cmd


== cron/event-scheduler-like

    proc at {time args} {
      if {[llength $args]==1} {set args [lindex $args 0]}
      set dt [expr {([clock scan $time]-[clock seconds])*1000}]
      after $dt $args
    } ;# RS
    at 9:31 puts Hello
    at 9:32 {puts "Hello again!"}

 proc atNext {time fmt cmd} {
      return [after [expr {([clock scan $time -format $fmt]-[clock seconds])*1000}] $cmd]
 }
 # Example:
 puts [atNext {Saturday 10:00} {%A %H:%M} {puts "Hallo Welt"}]
 vwait forever


== nested function call

be cautious about the return value.



== tcl error messages

"too many nested evaluations (infinite loop?)"

problem of recursive function call.


== coding issues

related code:

----
myinteract {
    ......
    if {<$enable_user_patterns || $enable_user_patterns==1} {
        set myinteract_session_input_patterns ""
    } else  {
        set myinteract_session_input_patterns2 ""       <1>
    }

    set interactcmd [subst {
        interact {
            -input $user_spawn_id 
                $myinteract_user_input_patterns
            -output $session_output
            -input $session_input 
                $myinteract_session_input_patterns
                $myinteract_session_input_patterns2
            -output $user_spawn_id
        }
    }]

    myputs "interactcmd looks $interactcmd" 3

    if !$nofeature {
        #eval $interactcmd {{{4}}}
        eval $interactcmd
    }
    ......
}
----

<1> without this, there will be a VERY TRICKY issue with user_patterns

testcase: 

    crtc myrouter, login via jtac-server then telnet to router

issue:

generate a file named test:

    echo "Connection closed by foreign host" > test

now test the persistent/RECONNECT action, defined in user_pattern:

this doesn't work:

    labroot@alecto-re0> file show test
    Apr 23 22:43:19
    Connection closed by foreign host

this will work:

    labroot@alecto-re0> start shell
    Apr 23 22:43:21
    % cat test
    Connection closed by foreign host
    interact: detected event:
    -Connection closed by foreign host-
    matches pattern:
    -Connection closed by foreign host-
    now execute action group -RECONNECT-!!
    action is RECONNECT
    persistent mode set, willlll reconnect in 10s

    <<<<count {10}s before proceeding...

removing the jtac server from login process, make it directly login to router.

    if !$in_jserver {
        set login_info($login_index) [subst {    \
            $login_jtac_server                     \
            $login_info($login_index)            \
        }]
    }

this will work.

and, exiting from router won't trigger pattern match, but exiting from jtac
server will.

    labroot@alecto-re0> exit
    Apr 23 22:16:30
    Connection closed by foreign host.
    pings@svl-jtac-tool01:~$ exit
    logout
    Connection to 172.17.31.80 closed
    interact: detected event:
    received message:
    -Connection to 172.17.31.80 closed-
    matches pattern:
    |Connection to (d{1,3}.){3}d{1,3} closed|Connection to S+ closed|Connection reset by peer-
    now execute action group -RECONNECT-!!
    action is RECONNECT
    persistent mode set, willlll reconnect in 10s




== crontab

=== error1: env

    no such variable
        (read trace on "env(USER)")
        invoked from within
    "set options(log_seperator)          "
        <<<<<<<<<<<<<<<<<<< new logs since: <<<<<<<<<<<<<<<<<<<<<<<
        < [time_now] $env(USER) {1} <
        <<<<<<<<<<..."
        ("eval" body line 62)
        invoked from within
    "eval $config_default"
        (file "/home/ping/bin/crtc/crtc" line 6925

=== error2: send_tty

    send_tty: cannot send to controlling terminal in an environment when there is no controlling terminal to
    send to!
        while executing
    "send_tty $msg"
        invoked from within
    "if !$in_shell {
                send_tty $msg
            } else {
                puts -nonewline $msg
            }"
        invoked from within
    "if $verbose {
            if !$in_shell {
                send_tty $msg
            } else {
                puts -nonewline $msg
            }
        }"
        (procedure "myputs2" line 4)
        invoked from within
    "myputs2 "<<<CRTC:$login_index:start to login, please wait ...\n""
        ("foreach" body line 2)
        invoked from within
    "foreach login_index $hostlist_full {
        myputs2 "<<<CRTC:$login_index:start to login, please wait ...\n"
        myputs2 "<<<CRTC:$login_index:to interup..."
        (file "/home/ping/bin/crtc/crtc" line 8046)

=== error3: stty

http://expect.sourceforge.net/FAQ.html#q27

    stty: impossible in this context
    are you disconnected or in a batch, at, or cron script?stty: impossible in this context
    are you disconnected or in a batch, at, or cron script?stty: ioctl(user): bad file number

        while executing
    "stty -raw"
        invoked from within
    "subst $myexpectcmd"
        invoked from within
    "if [expr {$enable_user_patterns && [array exists user_patterns]}] {

            myputs "enable_user_patterns set ($enable_user_patterns) and user_pattern..."
        (procedure "myexpect" line 102)
        invoked from within
    "myexpect $router $pattern      $datasent $pattern_timeout [expr !$pa_pair] 0    "
        invoked from within
    "if {[regexp {GRES\s*(\d*)} $datasent -> interval_gres]} {
                    myputs "GRES command detected!"
                    if {[string equal $interval_..."
        (procedure "do_pag" line 166)
        invoked from within
    "do_pag $login_index  login_info cmd_output_array_login_info  $interval_cmd $waittime_login"
        (procedure "spawn_login" line 53)
        invoked from within
    "spawn_login $login_index"
        ("foreach" body line 5)
        invoked from within
    "foreach login_index $hostlist_full {
        myputs2 "<<<CRTC:$login_index:start to login, please wait ...\n"
        myputs2 "<<<CRTC:$login_index:to interup..."
        (file "/home/ping/bin/crtc/crtc" line 8046)

== misc

=== editor/tcl shell/etc

tkcon GUI looks good, 
but seems not stable - hanging after running crtc

=== tclreadline

The tclreadline package makes the GNU Readline library available for
interactive tcl shells. This includes history expansion and file/command
completion. Command completion for all tcl/tk commands is provided and commmand
completers for user defined commands can be easily added. tclreadline can also
be used for tcl scripts which want to use a shell like input interface. In this
case the ::tclreadline::readline read command has to be called explicitly.

    sudo apt-get install tclreadline

then add below in `~/.tclshrc` and `~/.expect.rc`:

    if {$tcl_interactive} {
        package require tclreadline 
        ::tclreadline::Loop
    } 

TIP: for me, I just like the arrow up/down, tab complete.

=== teacup

activetcl tool, used to install extensions.

=== dejagnu



=== make script executable but not readable ..

It  is often useful to store passwords (or other private information) in Expect
scripts.  This is not recommended since anything that is stored on a computer
is susceptible to being accessed by any‐ one.  Thus, interactively prompting
for passwords from a script is a smarter idea than embedding them literally.
Nonetheless, sometimes such embedding is the only possibility.

Unfortunately, the UNIX file system has no direct way of creating scripts which
are executable but unreadable.  Systems which support setgid shell scripts may
indirectly simulate this as follows:

Create the Expect script (that contains the secret data) as usual.  Make its
permissions be 750 (-rwxr-x---) and owned by a trusted group, i.e., a group
which is allowed to read it.   If  necessary,
create a new group for this purpose.  Next, create a /bin/sh script with
permissions 2751 (-rwxr-s--x) owned by the same group as before.

The result is a script which may be executed (and read) by anyone.  When
invoked, it runs the Expect script.

=== return in `source` file

will make the rest part skipped

=== NewHeadline

this will give unexpected result:

    puts "download_folder looks $download_folder!"
    if [file exists $download_folder] {
        puts "$download_folder does not exists"
    } elseif [catch "set download_folder [pwd]"] {
        set download_folder "/var/tmp"
    } else {
        error "can't locate a valid download_folder"
        exit
    }

result:

    download_folder looks ~/download_folder!
    /home/ping does not exists


== reference

* https://www.tcl.tk/man/tcl8.4/TclCmd/re_syntax.htm[re_syntax - Syntax of Tcl
  regular expressions]
* http://www.nist.gov/el/msid/infotest/dlibes.cfm 

* TCL programming cookbook

.shell script to automate interaction:

only this works:

shell script to automate telnet/ftp, not working for ssh

    (
        echo "labroot"
        sleep 2
        echo "lab123"
        sleep 2
        echo "show version | no-more"
        echo "show config | no-more"
        sleep 20
    ) | telnet -K alecto-re0.ultralab.juniper.net > showver.txt

for telnet this doesn't work:

    telnet -K alecto-re0.ultralab.juniper.net << EOF
    labroot
    lab123
    show version | no-more
    EOF

for ssh this doesn't work:

    (
        sleep 5
        echo "lab123"
        sleep 2
        echo "show version | no-more"
        echo "show config | no-more"
        sleep 20
    ) | ssh -t -t labroot@alecto-re0.ultralab.juniper.net

    pings@svl-jtac-tool01:~/bin$ ./test.sh                                                        
    Warning: Permanently added 'alecto-re0.ultralab.juniper.net' (DSA) to the list of known hosts.
    labroot@alecto-re0.ultralab.juniper.net's password:                                           

endif::expectnote[]



= expect notes

== expect resources

http://expect.sourceforge.net/

expect over windows:
http://docs.activestate.com/activetcl/8.4/expect4win/ex_usage.html

ifdef::internal[]

== exploring expect book progress

* P43 (2016-04-27) 
* P60 (2016-04-29) 
* P73 (2016-04-30) 
* P100(2016-05-01) 
* P120(2016-05-02) 
* P140(2016-05-03) 
* P152(2016-05-05) 
* P160(2016-05-06) 
* P188(2016-05-07)
* P227(2016-05-13) 
* P244(2016-05-14) 
* P256(2016-05-15) 
* P264(2016-05-18) 
* P270(2016-05-20) 
* P290(2016-05-21) 
* ftp over telnet (2016-05-22) 
* P307(2016-05-25) 
* P324(2016-05-26) 
* P340(2016-05-27) 
* P361(2016-05-30) 
* P380(2016-06-03) 


endif::internal[]

== `expect`

[width="80%",cols="3,7",options="header"]
|=====================================================================
|exp_ cmd        |Functions
|expect          |expect message from current spawned process
|expect_user     |expect message from stdin 
|expect_tty      |expect message from /dev/tty
|=====================================================================



=== expect syntax

    expect [[-opts] pat1 body1] ... [-opts] patn [bodyn]

best style in practice:

    expect {
        -i $process -re $pattern {
            myputs "expected pattern -$pattern- matched!"
            puts "will sleep for $reconnect_interval and retry"
            mysleep $reconnect_interval
            puts "retry current cmd -$datasent- now..."
            continue
        }
        timeout {
            if {$escape_count < 3} {
                incr escape_count
                puts "press ctrl-c again to escape ..."
                #just repeat sending ctrlc if got timeout
                exp_send -i $process "[CONST CTRL_C]\r"
                exp_continue
            } else {
                puts "not able to escape(not seeing expected prompt\
                    -$pattern-, will exit"
                exit
            }
        }
    }


=== expect_out

.expect_out with nested pattern P116
image::https://cloud.githubusercontent.com/assets/2038044/12874355/8c867f54-cda0-11e5-819e-28c15d3325e1.png[]

    [root@ftosx1 Expect]# expect
    expect1.1>  expect -re "I(.*)BA"
    The dog is black and I like ABBA group
    expect1.2>  send $expect_out(buffer)
    The dog is black and I like ABBAexpect1.3>
    expect1.4>
    expect1.5> send $expect_out(0,string)
    I like ABBAexpect1.6>
    expect1.7> send $expect_out(1,string)
     like ABexpect1.8>
    expect1.9>

* internal buffer          : all user input (not including typo fix editting).
  no way to access this buffer: "The dog is black and I like ABBA group", P74
* in expect_out(buffer)    : the original entire *matched* string, "The dog is
  black and I like ABBA"
* in expect_out(0, string) : the match to the pattern, "I like ABBA"
* in expect_out(1, string) : the first parenthesized subpattern that match our
  pattern, "like AB"
* expect_out(spawn_id)     : spawn_id of matching process P254

----
                       |<-pattern->|     |<-pattern->|
internal buffer : xxxxx|mm(nnnn)mmm|yyyyy|m(nnnnnnn)m|zzzzz|   <1>
                            |      ^             |   ^
                            |      <6>
                            |                    |   <8>
                            | <2>
                            |                    |<7>
                            v                    v
expect_out(buffer)  :|<-             ->|<-               ->| <3>

expect_out(0,string):      |<--------->| <4>

expect_out(1,string):         |<-->| <5>

----

<1> expect_out buffer strings in "internal buffer"
<2> when a match found, move strings matched to whole expect pattern into
expect_out(buffer)
<3> expect_out(buffer) contains all matched strings, plus chars that came earlier
but did not match
<4> expect_out(0,string) contains all matched strings
<5> expect_out(1,string) contains first substring in first ()
<6> internal buffer now start from the char next to the previous match
<7> when a new match found, go step 2

.simulating expect_out behavior with "switch"

    -re ".+" {  ;#{{{5}}}
        myputs "get new strings -[set expect_out(0,string)]-"
        append buf [set expect_out(buffer)]
        myputs "this make the buf looks:\n-[set buf]-"
        switch -regexp -matchvar match -- [set buf] {
            {$pattern} {        ;#{{{6}}}
                set buf_match_loc [string first \
                    [lindex [set match] 0] [set buf]]
                set buf_rm_bef_matched [string replace \
                    [set buf] 0 [set buf_match_loc]-1 ]
                set buf [string trimleft \
                    [set buf_rm_bef_matched] [set match]]
                myputs "buf now looks:\n-[set buf]-" 3
            }
        }
    }


=== expect -indice 

this code:

    expect ">"
    send "configure\r"
    expect -indice "#"
    puts "\nstart of expect_out -----------------"
    parray expect_out
    puts "end of expect_out -----------------"

generated this array:


    labroot@alecto-re0> configure
    Entering configuration mode

    [edit]
    labroot@alecto-re0#
    start of expect_out -----------------
    expect_out(0,end)    = 70
    expect_out(0,start)  = 70
    expect_out(0,string) = #
    expect_out(1,end)    = 1628
    expect_out(1,start)  = 1627
    expect_out(1,string) = $
    expect_out(2,end)    = 1627
    expect_out(2,start)  = 1627
    expect_out(2,string) = $
    expect_out(buffer)   =  configure
    Entering configuration mode

    [edit]
    labroot@alecto-re0#
    expect_out(spawn_id) = exp6
    end of expect_out -----------------

=== expect -re

both expect and interact support OR

.anchor ^ and $:

expect anchors at the beginning of whatever input it has received
without regard to line boundaries.

=== expect -gl

.implicit -gl

    expect "a*"

.explicit -gl

    expect -gl "a*"

explit -gl is useful to match some special patterns:

    expect -gl "timeout"
    expect -gl "-re"

=== expect *

can be used to move all old (internal) buffer into expect_out(buffer), which
can be accessed.

    timeout {
        expect *
        puts "expect_out(buffer) now looks -[set expect_out(buffer)]-"
        puts "timeout when looking for pattern $pattern"
    }

=== expect (nothing)

    expect

expect either "timeout" or "eof"?

expect ignores patterns for which it has no spawn ids. If the expect command
has no valid spawn ids at all, it will just wait. P269.

=== expect timeout

these are all the same:

    expect {
        {timeout} {
            puts "timeout1!!!"
        }
        timeout {
            puts "timeout2!!!"
        }
        "timeout" {
            puts "timeout3!!!"
        }
    }

=== expect eof

matches when spawned process close connection actively - before expect timeout
first.


=== expect pattern no-op

just match, but no-operation (p190)

    expect $pattern


=== why "\\\$"?

    set pattern "(% |> |# |\\\$ |%|>|#|\\\$)$"

in a typical tcl regex pattern usage environment, it will be scanned and
processed twice:

* tcl:          "\\\$"  -> "\$"
* regex:        "\$"    -> literal `$` sign

see P330 and P91

now, with subst and eval, one more scan will be gone through:

* subst         `\\\$`  -> `\$`
* tcl           `\$`    -> literal `$` sign
* regex         `$`     interpretted as "end of the string"

so if goal is to use a regex to indicate a literal `$`, then the original
pattern needs to be either protected from being evaluated, or compenstated with
more `\`. 

more rules: P126

"non-substitution" behavior: occurs anywhere "$" is not followed by an
alphanumeric character such as in the string "$+". 

it looks, 

* everything can be a tcl var

    tclsh8.6 [~]set + abc              
    abc                                

* not any var will be substituted

    tclsh8.6 [~]puts $+                
    $+                                 
    tclsh8.6 [~]puts "$+"              
    $+                                 

    tclsh8.6 [~]puts "[set +]"         
    abc                                

=== glob vs. regex

* ^
* $
* \ the next char literally

    foreach filename [glob *.expl {
        set file [open $filenamel
            # do something with $file
        close $file
    }

* [] a range of char
* * anything (.* in regex)
* ? single char (. in regex)
* {} a choice of string (what is this?)
* ~ home

glob vs. regex:

image::https://cloud.githubusercontent.com/assets/2038044/14946263/673ebc4a-0fed-11e6-8d69-fb1420d1cee2.png[]

P126:

for regex: unless preceded by a backslash, the $ is special no matter where in
the string it appears. The same holds for "^".

    expect -re "% $|foo"

for glob: the glob pattern matcher only treats " as special if it appears as
the first character in a pattern and a $ if it appears as the last character





=== expect common patterns common example

.match any numbers: -1 0.35 22.2 ...

    expect -re "-?(01\[1-9J\[0-9J*)?\\.?\[0-9J*"

a simpler but may not be "precise" one: P189

    "^\[0-9]+$"

.match a return , P112

    expect -re "(.*)\n" {
    }

.match just one line from command output 

    expect -re "\n(.*)\r"       #(P119)
    expect -re "(\[^\r]*)\r\n"  #(P133)

so this looks precise and enough:

    expect -re "\n(\[^\r]*)\r

per my test sometime this works better:

    expect -re "(\[^\n]*)\r\n"

so to match 1st/2nd/3rd line of command output:

----
send "cat a.txt\r"            <1>
match the 1st line of output
expect -re "\r\n(\[^\n]+)\r\n"
            ---- ------- ----
            <2>
                   <3>
                          <4>
----

<1> user type a cmd "cat a.txt" and hit return
<2> the cmd "cat a.txt" will be echoed back, and "return" is converted to the
first return+newline \r\n
<3> 1st line in the real output of the cmd
<4> newline for the 2nd line of output

or do it line by line:

    send "cat $infile\r"
    #absorb cmdline itself
    expect -re "^(\[^\n]*)\r\n"
    #first line
    expect -re "^(\[^\n]*)\r\n"
    #second line
    expect -re "^(\[^\n]*)\r\n"

.collect user input info from stdin :

    expect_user {
        -re "(\[^\n]+)\n" {
        }
    }

NOTE: use `\n` here instead of `\r`? 


.match 2nd line of command output

these code works well:

----
send "cat $infile\r"    <1>
#absorb the echoed "cat filename.txt" line
expect -re "^(\[^\n]*)\r\n"     <2>
#check the 2nd line, see if any error happened
expect -re "^(\[^\n]*)\r\n" {   <3>
    switch -regexp -matchvar match -- $expect_out(1,string) {
        "No such file" {return 1}
    }
}

set timeout -1
expect {
    -re "(^\[^\n]*)\r\n\[^\n]*$pattern_common_prompt" {         <5>
        puts $out "$expect_out(1,string)"
        send_user "."
        close $out
    }
    -re "^(\[^\n]*)\r\n" {                                      <4>
        puts $out $expect_out(1,string)
        send_user "."
        exp_continue
    } timeout {
        puts "timeout!"
        close $out
    }
}
----

<1> cat a whole file
<2> ignore the 1st line, which will always be the echoed command line user
typed in. to ignore it just match the very 1st line of the "cat file" output
and do nothing
<3> check the 2nd line, and see if there are errors
<4> for each single new line, extract the content, ignoring both `\r\n` and
save to file via puts. (puts will append a \n again, so this effectively just
removes the \r)

.retrieve the prompt

    send "\r"
    expect -re "\r\n(\[^\r]+)$" {
    }

.capture a "prompt" , works good most of the time. modified from P121

    set options(pattern_common_prompt) "(% |> |# |\\\$ |%|>|#|\\\$)$"

simplied as:

    set options(pattern_common_prompt) "((%|>|#|\\\$) ?)$"

further simplified (one less \):

    set options(pattern_common_prompt) "((%|>|#|\\$) ?)$"

TIP: reason `\\$` is same as `\\\$` here, is because: TCL won't do
substitution to a single `$` char - it does not look a valid var - a var with
no name... This only applies to a single `$` as a special case.

=== `expect` limitation and workaroud

it's hard to diff between "timeout" to a pattern match, and a stalled "character
flow" with the native `expect` command.

    expect {
        -re ”$pattern" {
        }
        timeout {
        }
    }

when "timeout" fires, it only indicates a no match to the pattern $pattern.

.workaround

crtc "slow_mode"

=== expect-send pair essentials

* use expect, send "must" be used together, otherwise out-of-order screen
  output will occur.

"send" command is too fast - it just send the string out and immediately
return, without awaiting for any output! actually in the case of interaction
with remote machine with telnet/ssh, the cmd display is always too slow to be
printed in the right place - right after the "send", and before other expect
statements are executed.

an "expect" after the "send", is important to slow down the send  ,and to
"sync" the command sending and cmd output displaying!

see "test1.exp".

    send_user "will send a timeout message after expect match"   ;#<1>
    expect {
        -re "$pattern" {
            send -i $spawn_id "#this is an timeout\r"       ;#<5>
            puts "within expect, sleep 5s "                 ;#<2>
            sleep 5
            puts "within expect, send 2 timeout again"      ;#<3>
            send -i $spawn_id "#this is an timeout\r"       ;#<7>
            send -i $spawn_id "#this is an timeout\r"       ;#<8>
        }
    }

    puts "will send show version after expect match"        ;#<4>

    expect {
        -re "$pattern" {
            send_user "got a match, send show version now\n"        ;#<6>
            eval {send -i [set session] "show version\r"}           ;#<13> 
        }


=== expect vs regexp vs switch

    regexp "$pattern" "$string" match substring1 substring2 .. 

    switch -exact -- $string {
        "$pattern" {
            $action
        }
        "$pattern" {
            $action
        }
        default {
            $action
        }

* very alike.
* same internal pattern matcher
* expect_out(0,string) = match
* expect_out(1,string) = substring1
* expect_out(2,string) = substring2

=== exp_continue

VERY useful, I used it a lot...

    #IMPORTANT: absorbing any possible extra prompts or whatever
    set timeout_old $timeout
    set timeout 1
    expect -i $process -re ".+" {exp_continue -continue_timer}  #<------
    set timeout $timeout_old

    expect {            
    -i "$user_spawn_id" ;#{{{4}}}
        -re [CONST $key_interact] {   ;#{{{5}}}               
            myputs2 "session:\\\[$router\\\]:you typed $key_interact key\
                here..."
            myputs2 "\nyou have the control now...\n"
            mycatch "stty -raw"
            #set oldmode [stty -raw]
            myputs "myexpect return RETURN_EXPECT_USER_INTERUPT"
            return "RETURN_EXPECT_USER_INTERUPT"
        }
        -re ".+" { ;#{{{5}}}
            puts "you typed something here...type $key_interact if you\
                want to interupt crtc..."
            exp_continue        #<------
        }

    -i $process ;#{{{4}}}
        -re {$pattern_more} {   ;#{{{5}}}
            exp_send -i $process $pattern_more_key
            exp_continue        #<------
        }
    }

==== absorbing extra chars

This is a very useful and frequently used feature: absorbing any possible extra
prompts or whatever:

    set timeout 1
    expect -i $session -re ".+" {exp_continue -continue_timer}

without this sometime expect - send - expect sequence will lose
synchronization.



=== expect -notransfer

change the default behavior: never clear data from internal buffer, even after
a match

=== expect null

null: all zero byte: "00000000"

=== expect_user

* special case of expect/send -i $user_spawn_id

expect_user vs. gets stdin

    -echo -reset -exact "!c" { 
        ......
        set read_stdin [gets stdin]
        ......
    }

vs.

p347

    -echo -reset -exact "!c" { 
        ......
        expect_user {
            -re "..." {
            }
            -re "..." {
            }
        }
        ......
    }


=== expect -brace

P160:
interesting discussion: how expect "guess" the "role" of its parameters:

==== default "rule" of guess

* one line arguments are treated as one pattern
* multiple line arguments, are treated as a pattern-action list

the pattern action list is usually put inside of a brace

.multiple argument, using continuation
===================================

    expect \
    patl actl \
    pat2 act2 \
    pat3 act3

===================================

.single argument, using braces
===================================

    expect {
        patl actl
        pat2 act2
        pat3 act3
    }

===================================

==== inconsistencies


a one line pattern, is treated as pattern-action lists, because the pattern
itself "looks like" a pattern-action list (due to the existence of `\n`)

.single line pattern (wrongly) treated as a pattern-action list
===================================

    expect "\npatl actl \npat2 act2 \n"

    set pattern "\npat1 act1 \npat2 act2 \n"
    expect $pattern

===================================

what are the pattern and action?

. one single pattern containing a list of strings, no action? (no)
. same as previous two example, still treated as a list of patterns actions,
instead of a single pattern? (yes)

test:

    expect [~]expect "\npat1 act1 \npat2 act2 \n"
    pat1        #<------input
    invalid command name "act1"         #<------error
    while evaluating {expect "\npat1 act1 \npat2 act2 \n"}



even within "brace", a one line pattern-action list is still treated as one
single pattern, just because it's in one line, and so "looks like" one pattern

.one line pattern-action list (wrongly) treated as a single pattern
===================================

    expect [~]set timeout 180
    180
    expect [~]set pattern2 {pat1 act1 pat2 act2}
    pat1 act1 pat2 act2
    expect [~]expect $pattern2
    pat1 act1 pat2 act2         #<------input
    expect [~]                  #<------match and return

===================================



==== force a single argument treated as a single pattern

In order to force a single argument to be treated as a pattern, use the -gl
flag

.guarantee a single pattern
===================================

    set pattern "\npat1 act1 \npat2 act2 \n"
    expect -gl $pattern

===================================

==== force a single arugment treated as a list of patterns

solution is to use `-brace`

.-brace
===================================

    expect [~]set timeout 180
    180
    expect [~]set pattern2 {pat1 act1 pat2 act2}
    pat1 act1 pat2 act2
    expect [~]expect -brace $pattern2
    pat1
    invalid command name "act1"
    while evaluating {expect -brace $pattern2}
    expect [~]

===================================


=== expect_before/expect_after

== send

[width="80%",cols="3,7",options="header"]
|=====================================================================
|exp_ cmd        |Functions
|send            |send message to current spawned process
|send_user       |send message to stdout
|send_tty        |send message to /dev/tty
|send_log        |send message to file (log_file)
|send_error      |send message to stderr
|=====================================================================

.*send_tty/exp_send_tty vs puts*

send_tty won't be redirected by shell...maybe useful in some cases.

.*send_user/exp_send_user*

* special case of send -i $user_spawn_id
* will be redirected if whole script got redirected (send_tty won't)
* allows logging of output through log_file
* prefered over tcl "puts" (P182)
* commonly used with "log_user 0"
* send_user -raw : disable the output translation (\n->\r\n)

.*send_error/exp_send_error*

.*send_log/exp_send_log*

send to files opened by:

* log_file
* exp_internal



=== send vs. puts

P284

* puts is used for communicating with files opened by Tel's open command
* send works with processes started by spawn
* send: controlled indirectly by log_user
* puts: by default terminate lines with a newline, unless using `-nonewline`,
  This is convenient when it comes to writing text files 
* send: Most interactive programs either read characters one at a time or look
  for commands to end with a \r. intead of newlines. 
* send: -raw is useful in raw mode

.use puts to emulate send_user:

one observation is, when iteract is slow down, because of too many patterns,
all "send_*" will "appear" slow down the same. this includes send, send_user,
send_tty, send_error.

puts, under this specific scenario, can be used to emulate what send_user does.

    proc myputs3 {msg} {    ;#{{{2}}}
        puts -nonewline "$msg\r"
    }

== spawn

it looks, after spawn in proc, the spawn_id (or at least any varible that hold
the value of it) has to be global. otherwise not working..

== email

.mail (expect book)

    set to pings@juniper.net
    set from test@test.net
    set subject "test"
    set body "test"

    exec mail $to << "From: $from
    To: $to
    From: $from
    Subject: $subject
    $body"

.sendmail

    /usr/lib/sendmail -t < body.txt

body.txt:

    To: bob@example.com
    From: tim@example.com
    Subject: Example of conversation threading
    In-reply-to: <put Message-ID of previous mail here>

    Body text here


tested this works:

    set to "pings@juniper.net"
    set from "no-reply-pings@juniper.net"
    #this does not seem to be working
    set replyto "test@test.net"
    set subject "test"
    set body "test"

    exec sendmail -t << "To: $to
    From: $from
    Subject: $subject
    In-reply-to: $replyto
    $body"

[NOTE]
====
* "TO: .." has to be in the very first line.
* "In-reply-to" does not work
====

== \r\n and stty mode

=== all about `\r\n`

.basis
* \r means "return", "carriage return", CR, ascii 13,0x0d, mac use as return,
  displayed as `^M` in unix
* \n means "line feed",                 LF, ascii 10,0x0a, unix use as return,
* "new line" representation: to windows: \r\n, unix:\n, mac:\r

hit enter:

    ping@ubuntu1:~$ 
    ping@ubuntu1:~$ 

tshark:

    183 Jun  5, 2016 15:35:15.279349000     10.85.47.3      10.85.4.32      \x0d
    Jun  5, 2016 15:35:15.279842000 10.85.4.32      10.85.47.3      \x0d\x0a,ping@ubuntu1:~$

type "pwd" hit return:

    ping@ubuntu1:~$ pwd
    /home/ping
    ping@ubuntu1:~$

tshark:

    464 Jun  5, 2016 15:42:18.007030000     10.85.47.3      10.85.4.32      p
    Jun  5, 2016 15:42:18.007436000 10.85.4.32      10.85.47.3      p
    Jun  5, 2016 15:42:18.104298000 10.85.47.3      10.85.4.32      w
    Jun  5, 2016 15:42:18.107340000 10.85.4.32      10.85.47.3      w
    Jun  5, 2016 15:42:18.248993000 10.85.47.3      10.85.4.32      d
    Jun  5, 2016 15:42:18.249344000 10.85.4.32      10.85.47.3      d
    470 Jun  5, 2016 15:42:18.478308000     10.85.47.3      10.85.4.32      \x0d
    Jun  5, 2016 15:42:18.478767000 10.85.4.32      10.85.47.3      \x0d\x0a,/home/ping\x0d\x0a,ping@ubuntu1:~$

router: hit enter:

    labroot@seahawks-re0>

    labroot@seahawks-re0>

tshark:

    279 Jun  5, 2016 15:39:20.148506000     10.85.47.3      172.19.161.201  \x0d
    Jun  5, 2016 15:39:20.149195000 172.19.161.201  10.85.47.3      \x0d\x0a,\x0d\x0a
    Jun  5, 2016 15:39:20.149595000 172.19.161.201  10.85.47.3      labroot@seahawks-re0>

.in summary

* in line mode, term driver do some work "behind the scene", making
  things more convenient:
  - input: you hit return \r, translated to \n, then echoed.
  - output: \n to \r\n (so your input `\r` eventually triggered a `\r\n`)
  - to make it , use `expect_user "\n"`, or `expect_user "\r\n"`

* in raw mode, term driver skipped some work: no translation/special char
  process (still echo?):
  - input: your \r remains \r, to match it use `expect_user "\r"`
  - output: no \n to \r\n translation
  - but, to make it easier, send will still do \n -> \r\n translation
  - use -raw to disable this send behavior

some typical usage examples:

    expect "1st line pattern\r\n2nd line"
    send_user "keyword pattern found!\n"
    send "pwd\r"

=== raw/cooked mode

.mode

    general             most common form
    ====================================
    line-oriented       cooked
    cha-oriented        raw

.line mode

* term driver do a lot of work to make things more convenient:
  - buff all keystrokes until a return is pressed P197
  - echo
  - some translations
* when buffered, special chars are interpreted to do special things: backspace
  to delete, etc
* this provide a "minimally intelligent user-interface" and drastically
  simplified most programs
* input(keystroke):\r -> \n translation P198
* output(display): terminal driver translate: \n to \r\n (P198) in line mode

.line mode examples P197

    send "Enter your name: "
    expect_user "\n"

user can fix typo and hit return when done

.raw mode

* expect_user does not wait for a return, and will try match immediately after
  keystroke
* \r works fine
* input: no \r -> \n translation

    send "Enter your name: "
    expect_user "\r"

* ouput: \n just move to newline without "return"
* output: "new line becomes line feed" P345 (meaning no \n -> \r\n output translation)
* output: send_user automatically translate newline to \r\n P345(P197), -raw
  disable this

.other nodes: in expect/send

* in send, use \r to represent "hit a return"
* in expect(including expect_user) cmd, terminal driver do translations: 
  - input(keystroke) translate: return (\r) to \n (new line) only in line
    mode (P192, P198)
* in send(send_user)
  - output(display) translate: translate \n to \r\n also under raw mode
  - output translation can be disabled by `-raw`


=== mode changing: stty

* expect stty calls native stty command in the system, so any native stty
  parameters can be provided
* expect stty provides some extra parameters for user's convenience

    stty echo|-echo
    stty raw|-raw|cooked|-cooked


TIP: and when using these parameters, expect stty won't call system stty.

* possibility of losing chars while switching modes
* should be executed during time when user is NOT typing (P199)

    stty raw ;# Right time to invoke stty
    send "Continue? Enter y or n: "
    stty raw ;# Wrong time to invoke stty

.inconvenience of no "stty -info"

this is the best practice so far: always flip it back first before flip it,
this will ensure tracking of oldmode won't be messed up.

to backup oldmode

    if [info exists oldmode] {eval stty $oldmode}
    set oldmode [stty -echo]

to recover oldmode

    eval stty $oldmode

otherwise considering this:

    #backup oldmode
    set oldmode [stty -echo]
    #backup oldmode again
    set oldmode [stty -echo]
    #recover old mode
    eval stty $oldmode

initially say stty parameter is "raw, echo", first oldmode is set to "raw echo",
the second oldmode will be set to "raw -echo".

.example of "mode dependent commands" P345

    system cat file
    exec kill -STOP [pidl
    expect -re "(. *) \n"

== match_max and full_buffer

.match_max
defines the size of the buffer (in bytes) used internally by expect. 

[quote,,]
While  reading  output, more than 2000 bytes can force earlier bytes to be
"forgotten".  This may be changed with the function match_max.  (Note that
excessively large values can slow down the pattern matcher.)  If patlist is
full_buffer, the corresponding body is executed if match_max bytes have been
received and no other patterns have matched.  Whether or not the full_buffer
keyword is used, the forgotten characters are written to expect_out(buffer).

NOTE: to understand this knob, you have to understand how expect cmd works
internally: spawned process generate output in the form of "chunk" by "chunk",
whenever expect received one chunk of output, it will put in an internal buffer
and scan the whole buffer to match patterns.  if no patterns found and more
chunks of strings come in, expect will re-scan the whole accumulated buffer to
match patterns. if this whole internal buffer grows bigger and bigger (because
of no match) and exceeding the configured "match_max" size, expect will "throw
away" the older strings and only keep the latest "match_max" size of buffer for
further pattern match - this esentially limits the performance impact
introduced by a huge buffer (because of no match) to a certain, controllable
extent.

P166 section "Pattern Debugging" , provided a detail illustration with
exp_internal.

some related performance fine-tune method: P151

* match incoming strings "line by line", and append each new line into a buff
* design a match to skip some predictable but unwanted strings blocks, and then
  use a second match to search in a much smaller rest part of the input

.full_buffer

when full, all internal buffer will be moved to expect_out

buffer slow users input, and send to process in a bulk, every 3s or full_buffer
reach, whichever comes first. great! P152

    set timeout 3
    while 1 {
        expect_user
        eof exit
        timeout {
            expect_user "*,,
            send $expect_out(buffer)
            full_buffer {send $expect_out(buffer)}
        }
    }

== log_user/log_file

.log_user

    log_user -info|0|1

suppress all output from spawned process, plus the echo from "spawn" command

* By default, the send/expect dialogue is logged to stdout (and a logfile if
open).  
* The logging to stdout is disabled by the command "log_user 0" and
reenabled by "log_user 1".  
* Logging to the logfile is unchanged 
* The -info flag causes log_user to return a description of the most recent
  non-info arguments given.
* commonly used with "send_user" (after log_user 0)

[NOTE]
====

it seems, log_user rely on "expect" to work, without "expect", the output won't
be suppressed...

----
log_user 0              #<------<1>
send_verbose "copying\n"
send "cat > $outfile\r"
set fp [open $infile r]
while {1} {
    if {-1 == [gets $fp buf]} {
        break
    } else {
    }
    send_verbose "."
    send -- "$buf\r"
}
if {$verbose_flag} {
    send_user "\n"			;# after last "."
}
send_user "now send eof\n"
send "\004"				;# eof
send_user "now close fp\n"
close $fp
expect -re "$prompt"    #<------<2>
log_user 1
----

<1> suppress spawned process output to screen(stdout)
<2> test shows: without this the log_user above doesn't work...

====

.log_file

* "ANY" output from spawned process 
  - including all control chars
  - not record "send", but will record if echoed from spawned process
  - so password won't be recorded since it is not echoed
* any diagnostics info expect itself generated
* "raw" log info - exactly what user is doing
* but not including Tcl's cmd output - puts
  - info from `send_user` will be recorded - one main diff with puts!
* by default, it looks the log file generated this way will contains '\r\n'
  - `\n` will be displayed as a new line
  - `\r` will be displayed as a `^M` in vim
  - if the desire is not to record `\r`, then use explicity file puts, instead
    simply `log_file`. see example and explanation in P257.

flags:

* -noappend     not to append, but to overwrite
* -a            ignore "log_user 0"
* -info         return current status


    #!/usr/bin/expect
    log_user 0                  ;#<------disable default disaplay from spawned process
    spawn $env(SHELL)
    stty raw -echo              ;#<------put terminal in raw mode 
                                ;#so no need "press" \r"
                                ;#no output that user can see

    set timeout -1
    set fp [open typescript w]  ;#<------open a file for log

    expect {
        -re ".+" {
            send -i $user_spawn_id $expect_out(buffer)
            exp_continue
        }

        eof exit

        -i $user_spawn_id -re "(.*)\r" {            ;#<------strip `\r`
            send -i $spawn_id $expect_out(buffer)   ;#<------send (w/o \r)
            puts $fp $expect_out(1,string)          ;#<------log w/o \r
            puts $fp [exec date]
            flush $fp
            puts "get -$expect_out(1,string)- in slashr"
            exp_continue
        }

        -re ".+" {
            send -i $spawn_id $expect_out(buffer)
            puts -nonewline $fp $expect_out(buffer)
            flush $fp
            puts "get -$expect_out(buffer)- in dotplus"
            exp_continue
        }
    }

=== pending issue

there is no known/existing/good way to do log cleanup: interpret all contrl chars.
this works close, but seems still buggy

    cat seahawks-re0.log | col -bp | less -R
    cat seahawks-re0.log | iconv -t utf-8 | col -bp | less -R


== signal

    Name        Description
    ===================================
    SIGINT      interrupt               ^c
    SIGTERM     software termination
    SIGQUIT     quit                    ^\
    SIGHUP      hangup
    SIGKILL     kill
    SIGPIPE     pipe write failure
    SIGSTOP     stop (really "suspend")
    SIGTSTP     keyboard stop
    SIGCONT     continue
    SIGCHLD     child termination
    SIGWINCH    window size change
    SIGUSRl     user-defined
    SIGUSR2     user-defined

SIG_IGN
SIG_DFL

spawn -ignore

.SIGINT

* triggered by ctrl-c under cook mode, changable via stty
* under raw mode, ctrl-c will not trigger SIGINT, but instead be sent literally
  to spawned process

.SIGTSTP/SIGSTOP/SIGCONT

ctrl-z  => SIGTSTP (not SIGSTOP, which can't be caught)
fg      => SIGCONT

`man 7 signal`:

 SIGCONT   19,18,25    Cont    Continue if stopped
 SIGSTOP   17,19,23    Stop    Stop process
 SIGTSTP   18,20,24    Stop    Stop typed at terminal

    The signals SIGKILL and SIGSTOP cannot be caught, blocked, or ignored.

need confirm:

"Control-Z sends SIGTSTP, not SIGSTOP. An important difference between them is
that programs can catch or ignore SIGTSTP, but not SIGSTOP. Programs may catch
TSTP and perform cleanup operations before suspending execution, but STOP
causes the process to stop without any notice. –"
http://superuser.com/questions/287428/whats-the-difference-between-s-and-z-inside-a-terminal

.trap all signals

    for {set i 1} {$i<=[trap -max]} {incr i}
        catch {trap $handler $i}
    }

.`exit -onexit`


== interact

* continues after a match (not like expect) 
* continue to shuttle chars back and forth between user and process

=== interact_out

* no `interact_out(buffer)`
* interact_out(0,string)        matched strings
* interact_out(1,string)        value captured in 1st parenthesis 
* interact_out(0,start)
* interact_out(0,end)

=== interact in a loop
P339

=== nested interact (interact under interact)
P338


=== interact default action: interpreter
P341

these are same, but only if X is the last pattern

    interact X
    interact X interpreter

testing shows, 

* press + will enter interpreter, then
* if type "return" will return to remote device
* else if type "exit" will exit script

question:

how does this work: P9

    while 1 {
        expect {
            eof {break}
            "UNREF FILE*CLEAR\\?" {send "y\r"}
            "BAD INODE*FIX\\?" {send "n\r"}
            "\\? " {interact + }
        }
    }


=== interact on eof

default/implicit action is "return"

=== interact timeout

* diff value can be used for input from user or spawned process: P343

    interact {
        timeout 10 {
            send_user "Keep typing-we pay you by the character!"
        }
        -o
        timeout 600 {
            send_user "It's been ten minutes with no response.\
                I recommend you go to lunch!"
        }
    }


* anti_idle_string candicates: P343-P344
- " "
- " \177"
- ctrl-g
- null

maybe the best will be to track if `^vim` is ever entered, then choose diff one
as anti_idle_string!

this doesn't work:

    interact {
        "!Q" exit
        timeout $timeout {
            puts "timeout $timeout!"
            incr timeout
        }
    }

will prints below, but every 2s exactly:

timeout 2!
timeout 3!
timeout 4!
timeout 5!
timeout 6!



=== implicit spawn_id and `-o`

                    expect/interact
                       |    
                       |   <------
                       |     -o "abc"
                       |    
                       |    
                       | (-i $spawn_id)
        o             telnet
                       |
       /|\  -----------+----------//--          remote machine
       / \             |                
                       |
                       |
             ====================>


    interact {
        "eunuchs" {send "unix"} \      user->      process
        "vmess" {send "vms"}     X       --------->
        "dog" {send "dos"}      / 
        -0
        "unix" {send_user "eunuchs"} \ user      <-process
        "vms" {send_user "vmess"}     X  <---------
        "dos" {send_user "dog"}      / 
    }

=== `-i` (explicit spawn_id)

-i $spawn_id

                    expect/interact
                       |
                       |
              ----->   |
              "!d"     |
                       | (-i $spawn_id)
        o             telnet
                       |
       /|\  -----------+----------//--     spawned process
       / \             |                
                       |
                       |
             ====================>

    interact {
        $user_key $act
        -i $proc1 {
            -re $pat1 act1
            -re $pat2 act2
        }
    }
        

=== `-u`

                    expect/interact
                       |
                       |
              ----->   |
                       |
                       | (-i $spawn_id)
                       |  telnet
                       |
    spawned -----------+----------//--     spawnd process
    process            |                
                       |
                       |
             ====================>

=== `-input` `-output`

* default

    interact -input $i -output $01 -output $02
    interact -input $i -output "$01 $02"

* default

    - If the first -input is omitted, user_spawn_id is used. 
    - If the first -output is omitted, spawn_id is used. 
    - If the -output after the second -input is omitted, user_spawn_id is used.

* with -u

    interact -u $proc -output $out -input $in

this is same as:

    interact -input $proc -output $out -input $in -output $proc

* multiple -output or -input (kibitz)

    interact
        -input $user_spawn_id   -output $process
        -input $userin          -output $process
        -input $process         -output $user_spawn_id
                                -output $userout

* combine multiple input/output

there 2 are the same:

    interact -input $i -output $o1 -output $o2
    interact -input $i -output "$o1 $o2"

The following command takes the input from i1 and i2 and sends it to 01.

    interact -input "$i1 $i2" -output $o1

* two -input flags in a row with no -output in between causes the first input
source to be discarded.  This can be quite useful for the same reasons that it
is occasionally handy to redirect output to /dey/null in the shell.  

Using these shorthands, it is possible to write the interact in kibitz more
succinctly: 

    interact { 
        -input "$user_spawn_id $userin" -output $process 
        -input $process -output "$user_spawn_id $userout"
    }



==== kibitz

basic usage: p355

    kibitz user1
    kibitz user1 tty1
    kibitz user1 vim
    kibitz user1 kibitz user2

    kibitz -noproc -tty ttyab user1

NOTE: this doesn't work well:

    kibitz user1 -noproc -tty ttyab

.how it works

    interact
        -input $user_spawn_id   -output $process
        -input $userin          -output $process
        -input $process         -output $user_spawn_id
                                -output $userout


=== -iwrite

-iwrite flag controls whether the spawn id is recorded, in this case, to
interact_out (spawn_id) .

    interact {
        -input  "$user_spawn_id $userin"
        -iwrite "foo" {actionl}
                "bar" {action2}
        -iwrite "baz" {action3
    }

The -iwrite flag forces the spawn_id element of interact_out to be written P361

=== indirect spawn_id

2 method of dynamic expect/interact to switch between multiple spawn_id:

* use a expect/interact loop, use normal $spawn_id value, change -i varible
  value, then continue to next expect/interact execution

* in current expect/interact, use indirect spawn_id, any change will take place
  right away.

indirect spawn_id make it possible to dynamically update spawn_id inside of
expect/interact, very useful!

.*examples in book:*

    interact {
        -input inputs -iwrite eof {
            set index [lsearch $inputs $interact_out(spawn_id)]
            set inputs [lreplace $inputs $index $index]
            set outputs [lreplace $outputs $index $index]
            if {[llength $inputs]==0} return
        } -output $process
        -input $process -output outputs
    }

* When inputs and outputs are modified, interact modifies its behavior to
  reflect the new values on the lists.
* The length of the input list is checked to see if any spawn ids remain. 
* If this check is not made, the interact will continue the connection.
* However, with only the process (in process) participating, all that will happen
  is that the output of it will be discarded.  
* This could conceivably be useful if, for example, there was a mechanism by
  which spawn ids could be added to the list (perhaps through another pattern).

.*examples in crtc:*

    set myinteractcmd [subst -nocommands {\n\
        interact {\n\

            -input $user_spawn_id\n\ 
                $myinteract_user_input_patterns\n
            -output process_output\n\

            -input kibitz_spawn_id
                ......
            -output process_output\n\

            -input process_input\n\
                $myinteract_process_input_user_patterns\n\
                $myinteract_process_input_patterns_misc\n\
                $myinteract_process_input_patterns_static\n\
            -output $user_spawn_id\n\
                $myinteract_user_output_patterns\n\
            -output kibitz_spawn_id\n\
        }\n\
    }]

    eval $myinteractcmd

the "kibitz_spawn_id" and "process_input" "process_output" here all indirect.
so from an user input "!k", trigger any change of these vars, will change the
interact behaviors, dynamically.



=== -reset



=== puts under interact

this won't give good format:

    puts "interact: detected event\n-$event-!!"

this won't either:

    puts "interact: detected event:"
    puts "-$event-"

tried send_tty, send_user, not change...

-reset seems resolved it.


=== inter_return vs return

inter_return  causes  interact  to  cause  a  return in its caller.  For
example, if "proc foo" called interact which then executed the action
inter_return, proc foo would return.


== expect vs. interact

these are all different technique to deal with certain situations:

=== call expect_user under interact

to collect user info

=== call expect under interact

an interesting effect:
absort some text output that will otherwise display in the screen...

P?

crtc:

    myinteract : !K -> interact_K -> inviteremote ..

    proc inviteremote {} {        ;#{{{4}}}
        expect {
            "is not logged in" {
                send_user "$kibitz_user1 is not logged in yet, check and\
                    try again later!"
                set inkibitz 0
            }
            -re "Escape sequence is \\^\\\]" {

                send_user "kibitz succeeded!\n"
            }
        }
    }

result:

    who are you going to invite?                      
    user1                                             
    will invite user1 ...                             
    kibitz succeeded!                                 
                                                      
the normal interaction should be like:

    ping@ubuntu47-3:~$ kibitz -noproc user1                             
    asking user1 to type:  kibitz -23054                                
    write: user1 is logged in more than once; writing to pts/69         
    Escape sequence is ^]                                               
                                                                        

=== call interact under expect

to allow user takes control for a while, they typically type some strokes to
return back to expect

    expect {
        ...
        script/expect control
        ...
        interact \\ {
            ...
            user control
            ...
        }
        ...
        script/expect control
        ...
    }

=== call interact under interact (nested interact)


== `-i` option

* expect/send/interact
* wait/close/match_max
* parity/remove_nulls

    spawn_id
    any_spawn_id
    error_spawn_id
    user_spawn_id
    tty_spawn_id



== Expect specific cmds

=== interpreter

When Expect is interactively prompting for commands, it is actually running a
command called interpreter. 

When Expect is interactively prompting for commands, it is actually running a command
called interpreter. 

When Expect is interactively prompting for commands, it is actually running a command
called interpreter. 

=== prompt1/2

== Expect errors

.a typo in code:

    expect {
        -i $spawn_id {{{4}}}    #<------
        -re {$} {
            puts "found a match to -$pattern1-"
            puts "expect_out(buffer) looks -$expect_out(buffer)-"
        }
        timeout {
            puts "timeout when looking for pattern"
        }
    }
    exp_internal 0

this expect will timeout with below debug msg:

    (spawn_id exp6) match glob pattern "{{4}}"? no
    "$"? no
    expect: timed out

.running under unix at

send_tty: cannot send to controlling terminal in an environment when there is
no controlling terminal to send to!                                                                    

== "screen" sharing 

* shell redirection

    crtc -a "set double_echo 1" myrouter > temp.log 
    tail -f temp.log

* log_file 

    tail -f router.log

* internal redirection

    catch {exec echo "[set interact_out(0,string)]" >  [set dest_tty]}

* kibitz

== debug Expect script

=== debug 1

=== trace

=== exp_internal

    exp_internal 0          no diagnostics
    exp_internal 1          send pattern diagnostics to standard error
    exp_internal -f file 0  copy standard output and pattern diagnostics to file
    exp_internal -f file 1  ..and send pattern diagnostics to standard error
                            
=== expect debugger


= tcl extentions

== sqlite: most popular tcl extension?

    sudo apt-get install sqlite3 libsqlite3-tcl

