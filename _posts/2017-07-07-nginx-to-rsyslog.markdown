---
layout: post
date: 2017-07-03 07:00
title: Sending Nginx logs to Elasticsearch via Rsyslog
---

Requirements - Nginx, Rsyslog latest version with modules configured, and elasticsearch.

## RSYSLOG INITIAL SETUP - 

Rsyslog is a log processor. It does not generate logs but it handles the way they are processed.  
Rsyslog is installed in the system by default, but it does not come with all the input and output modules installed in it. So, firstly we need to install the modules we need for our use case.    

Firstly we will add the rsyslog repository to our system's /etc/apt/sources.list,

    $ sudo add-apt-repository ppa:adiscon/v8-stable 

Then update our system cache

    $ sudo apt-get update

Finally, we will install the required modules,

    $ sudo apt-get install rsyslog-mmnormalize rsyslog-elasticsearch

By this way, you can add any module you want. I will explain the functioning of these modules later.

* * *

<!---
Manual Mode -
    Here are the libraries, $ sudo apt-get install libfastjson liblognorm-dev liblognorm2 libgcrypt11-dev libestr* liblogging-stdlog0 liblogging-stdlog1 liblogging-stdlog-dev

    They are for facilitating the working of rsyslog.

After that, we enable modules which are needed i.e, elasticsearch(for outputting data to elasticsearch) and mmnormalize(for parsing the data), 

$ sudo ./configure -/-enable-elasticsearch -/-enable-mmnormalize -/-enable-module_name # loads the configuration required
remove the slashes .

Then, 
$ sudo make # compiles it

Then,  
    $ sudo make install # installs the compiled rsyslog into relevant directories


-->


## CONFIGURING NGINX TO SEND LOGS TO RSYSLOG


Nginx can forward logs to rsyslog easily. It can do that by 2 ways, through Unix socket and by UDP.
For that firstly, we have to tell rsyslog to listen to nginx logs via UDP or unix socket.

For that, we have to load the udp and unix socket input modules in rsyslog,

    $ sudo vim /etc/rsyslog.conf

And make sure these lines are not commented,

    module(load="imuxsock") # opens unix socket at /dev/log

    module(load="imudp")  # loads the udp module
    input(type="imudp" port="514")  # opens udp port at port 514

<!---    module(load="imudp" Port=514) # by default is 514
    module(load-”imuxsock” Socket=/dev/log)  -->

After that, we have to tell nginx to push its logs to rsyslog. That can be done in 2 ways,

First, 

    access_log syslog:server=127.0.0.1:514,facility=local7,tag=nginx,severity=info;

Second, 

    access_log syslog:server=/dev/log,facility=local7,tag=nginx,severity=info;

If you do not include the `facility`, `tag` and `severity`, it will send the default valuse. By default these are their values, 
`facility = local7`, `tag = nginx` and `severity = info` .



## CONFIGURING RSYSLOG TO SEND LOGS TO ELASTIC 

The `rsyslog-elasticsearch` module we installed earlier is used to send logs to elasticsearch. Elasticsearch takes in data in JSON format. Elasticsearch can store the data accurately only if we send the data from rsyslog in JSON format. To parse our log messages and convert them into JSON fields, we will the module `rsyslog-mmnormalize` module.

Rsyslog can parse the log data into the JSON format which is required by elasticsearch. It uses the liblognorm library and the mmnormalize module to convert the data into json. The data is then sent to elastic.

It uses custom template and the mmnormalize module.

template(name="all-json-nginx"
    type="list"){
 property(name="$!all-json")
 }

All the fields are passed into the $!all-json variable.
To only include the selected OR to add more fields, we could do, 


template(name=”all-json-nginx” type=”list”){
    constant(value="{")
      property(name="timereported" dateFormat="rfc3339" format="jsonf" outname="@timestamp")  # the timestamp
    constant(value=",")
      property(name="hostname" format="jsonf" outname="host")  # the host generating stats
    constant(value=",")
      property(name="$!all-json" position.from="2")            # finally, we'll add all metrics
}

Here above, we have added the hostname as extra field.

And here is the parsing through mmnormalize

action(type="mmnormalize"
  rulebase="/opt/rsyslog/apache.rb"
)

Remember, by default it parses the message field of a syslog entry. If we want to parse some other field, we can pass it as a variable=hostname, or any other property we want to name, in the action directive.


Here, rulebase is the file where parsing rules are set, here below are the contents of that file, 

rule=tag-sent-by-nginx(default=nginx): %remote_addr:word% %ident:word% %auth:word% [%@timestamp:char-to:]%] "%method:word% %request:word% HTTP/%httpversion:float% "%status:number% %bytes_sent:number% "%referrer:char-to:"%" "%agent:char-to:"%"%blob:rest%
The patterns are by default included in the liblognorm library.


We can pass the rule in here also, but that makes the code unclean and causes confusion.

When elastic is ON, the data will be sent to it without any difficult.

When elastic is OFF, then data will be queued in the action queue. When elastic again will be ON, then data will be sent to elastic.

<!---
Extra Plugins -

                         
rsyslog statistic counter Queues                     
                                                   
Queue
For each queue inside the system its own set of statistics counters is created. If there are multiple action (or main) queues, this can become a rather lengthy list. The stats record begins with the queue name (e.g. "main Q" for the main queue; ruleset queues have the name of the ruleset they are associated to, action queues the name of the action).
size – currently active messages in queue
enqueued – total number of messages enqueued into this queue since startup
maxsize – maximum number of active messages the queue ever held
full – number of times the queue was actually full and could not accept additional messages
discarded.full – number of messages discarded because the queue was full
discarded.nf – number of messages discarded because the queue was nearly full. Starting at this point, messages of lower-than-configured severity are discarded to save space for higher severity ones.
 

2 - Storing info about IPs,

http://www.rsyslog.com/doc/master/configuration/modules/mmdblookup.html

We can use this plugin to add extra info about the IP addresses.
-->
