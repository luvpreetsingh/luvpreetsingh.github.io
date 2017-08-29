---
layout: post
date: 2017-07-03 07:00
title: Sending Nginx logs to Elasticsearch via Rsyslog
---

**Requirements** - Nginx, Rsyslog latest version with modules configured, and elasticsearch.
I also suggest you to grab some basics about rsyslog before reading this.

![rsyslog](/assets/rsyslog.jpg)   ![elasticsearch](/assets/elasticsearch.jpg) 



## RSYSLOG INITIAL SETUP

Rsyslog is a log processor. It does not generate logs but it handles the way they are processed.  
Rsyslog is installed in the system by default. But it does not come with all the input and output modules compiled in it. So, firstly we need to install the modules we need for our use case.    

There are 2 modules we are going to need, 

* omleasticsearch - to send logs to elastic
* mmnormalize - to normalize the log messages into fields

Below are the steps required to add these :-

Firstly we will add the rsyslog repository to our system's `/etc/apt/sources.list`,

    $ sudo add-apt-repository ppa:adiscon/v8-stable 

Then update our system cache,

    $ sudo apt-get update

Finally, we will install the required modules,

    $ sudo apt-get install rsyslog-mmnormalize rsyslog-elasticsearch

By this way, you can add any module you want. But we only need these 2.

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


Nginx can forward logs to rsyslog easily. It can do that by 2 ways, through Unix socket and by IP socket.
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

If you do not include the `facility`, `tag` and `severity`, it will send the default values of these properties. By default these are their values, 
* `facility = local7`
* `tag = nginx`
* `severity = info`

If you are wondering what these things are, these are the properties a log message has. Rsyslog judges the logs according to these properties. [Click here](http://www.rsyslog.com/doc/master/configuration/properties.html) to read them in detail.

* * *

## CONFIGURING RSYSLOG TO SEND LOGS TO ELASTIC 

Elasticsearch takes in data only in JSON format. To parse our plain text log messages and convert them into JSON fields, we will use the module `rsyslog-mmnormalize`.

Rsyslog can parse the log data into the JSON format which is required by elasticsearch by using the liblognorm library and the mmnormalize module. The data is then sent to elastic.

A custom template for the data format has to be used. So, we need a new format(JSON) for data which we can easily create by creating a custom template,

    template(name="all-json-nginx"
         type="list"){
     property(name="$!all-json")
    }

All the fields parsed are passed into the `$!all-json` variable.
To add more fields, we could do, 


    template(name=”all-json-nginx” type=”list”){
        constant(value="{")
          property(name="timereported" dateFormat="rfc3339" format="jsonf" outname="@timestamp")  # the timestamp
        constant(value=",")
          property(name="hostname" format="jsonf" outname="host")  # the host generating stats
        constant(value=",")
          property(name="$!all-json" position.from="2")            # finally, we'll add all metrics
    }

Here above, we have added the hostname as extra field. Adding hostname can be useful if you are handling logs from multiple servers.

We have declared our custom template. Now, we need to write a parsing rule which can parse our data and make different fields using it.
Here, rulebase is the file where parsing rules are set, here below are the contents of that file,

    rule= api: %remote_addr:word% %ident:word% %auth:word% [%@timestamp:char-to:]%] "%method:word% %request:word% HTTP/%httpversion:float% "%status:number% %bytes_sent:number% %requesttime:float% "%referrer:char-to:"%" "%agent:char-to:"%"%blob:rest%

Here is a log entry which is divided into fields perfectly according to the above rules,

     xx.xx.xx.xx - - [2017-08-29T15:22:28+05:30] "GET /api/v1/fleet/fleet_bookings/?booking_type=checkout HTTP/1.1 "200 12 0.609 "-" "python-requests/2.9.1" "-"

We have made rule for the normalization of logs. Now, we need to tell rsyslog to perform this normalization action. This is how we do it,

    action(type="mmnormalize"
      rulebase="/opt/rsyslog/apache.rb"
    )

Remember, by default it parses the message field of a syslog log entry. If we want to parse some other field, we can pass it as a parameter, `variable=hostname`, or any other property we want to name, in the action directive.


The rule here will be hard to understand if you are viewing it for the first time. It works in the same way as logstash filter plugin. Here are links where you can read more about them.
* [http://www.liblognorm.com/files/manual/configuration.html#date-rfc3164](http://www.liblognorm.com/files/manual/configuration.html#date-rfc3164)
* [http://www.liblognorm.com/files/manual/sample_rulebase.html](http://www.liblognorm.com/files/manual/sample_rulebase.html)


Now, we have achieved the following pieces, 

* Configured Nginx to send logs to rsyslog
* Written the parsing rule to parse messages

The next and final piece of the puzzle is to send these logs to elasticsearch.

We will use the `omelasticsearch` module. This is the action which will be performed by rsyslog,

    action(type="omelasticsearch"
      template="all-json"  
      searchIndex="testing-logs"
      searchType="logs"
      server="127.0.0.1"
      serverport="9200"
      uid="user"
      pwd="pass"
      bulkmode="on"  
      action.resumeretrycount="-1" 
      queue.type="LinkedList"      
      queue.highwatermark="40000"  
      queue.spoolDirectory="/var/spool/rsyslog/queues"
      queue.filename="rsyslog-testing-logs"
      queue.lowwatermark="5000" 
      queue.maxdiskspace="100m" 
      queue.size="50000"    
      queue.dequeuebatchsize="1000" 
      queue.saveonshutdown="on"
    )

Here is the breakdown of this action, 

* template = Name of the custom template earlier declared
* searchIndex = Index in elasticsearch where to store the data
* searchType = Type of the document
* server = address of server where elasticsearch is running
* serverport = port of server on which elasticsearch is running
* bulkmode = to use the elasticsearch bulk API
* action.resumeretrycount = retry if it fails to send due to some reason

The `uid` and `pwd` options are only to be used if you have used authentication on elastic side.

* uid = username 
* pwd = password

These are the basic things you need to understand. When elastic server is ON, the data will be sent to it without any difficulty.


You have your all the pieces to solve the puzzle. Now put them in order to solve it. Create a file in `/etc/rsyslog.d/` folder ending in **.conf**. [Here is the complete file in correct order](https://gist.github.com/luvpreetsingh/a863ad26a2423b5a7dde755949b9a5e9).


#### NOTE ON QUEUES :

I have used queues here. Queues provide reliability to our flow of logging data. Logs are really important data for system debugging. And loss of these logs can make life hard at times. So, to avoid data loss, queues are really helpful. If anyday due to any reason, when elastic is OFF OR network is causing issues, then data will be queued in the queue. When things go normal again, then data will be sent to elastic. This is really important and easy to implement.

Queues are a really important and a vast topic for this post. I suggest you read them where they are explained best i.e, by rsyslog author himself.
 
* [Queue Analogy](http://www.rsyslog.com/doc/v8-stable/whitepapers/queues_analogy.html)
* [Queue Concepts](http://www.rsyslog.com/doc/v8-stable/concepts/queues.html)

Please share it if you find it helpful.


<script src="//platform.linkedin.com/in.js" type="text/javascript"> lang: en_US</script>
<script type="IN/Share" data-url="https://luvpreetsingh.github.io/nginx-to-rsyslog/" data-counter="top"></script>

<script>window.twttr = (function(d, s, id) {
  var js, fjs = d.getElementsByTagName(s)[0],
    t = window.twttr || {};
  if (d.getElementById(id)) return t;
  js = d.createElement(s);
  js.id = id;
  js.src = "https://platform.twitter.com/widgets.js";
  fjs.parentNode.insertBefore(js, fjs);

  t._e = [];
  t.ready = function(f) {
    t._e.push(f);
  };

  return t;
}(document, "script", "twitter-wjs"));</script>


<a class="twitter-share-button" href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via={{ site.twitter_username }}&related={{ site.twitter_username }}" rel="nofollow" target="_blank" title="Share on Twitter" data-size="large">Twitter</a>

<div id="fb-root"></div>
<script>(function(d, s, id) {
  var js, fjs = d.getElementsByTagName(s)[0];
  if (d.getElementById(id)) return;
  js = d.createElement(s); js.id = id;
  js.src = "//connect.facebook.net/en_GB/sdk.js#xfbml=1&version=v2.10";
  fjs.parentNode.insertBefore(js, fjs);
}(document, 'script', 'facebook-jssdk'));</script>

<div class="fb-share-button" data-href="https://luvpreetsingh.github.io/nginx-to-rsyslog/" data-layout="button_count" data-size="large" data-mobile-iframe="true"><a class="fb-xfbml-parse-ignore" target="_blank" href="https://www.facebook.com/sharer/sharer.php?u=https%3A%2F%2Fluvpreetsingh.github.io%2Fnginx-to-rsyslog%2F&amp;src=sdkpreparse">Share</a></div>

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
