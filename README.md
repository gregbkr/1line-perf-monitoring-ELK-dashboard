# 1line performance (top) monitoring to ELK dashboard
Monitor CPU, RAM, DISK, SWAP (via Top) for linux VMs, send to a file, then send to ELK for dashboards.

![perf.JPG](https://github.com/gregbkr/1line-perf-monitoring-ELK-dashboard/raw/master/perf.JPG)

## How it work:
1. Top formated will display all values.
2. We will loop on ansible host file: contains the list of our vms (but you can adapt and loop on a file listing your inventory)
3. A cron will run that command and send all in a flat file.
4. This flat file is pickup by filebeat and send to ELK for dashboard.

## Why this setup?
Cadvisor+prometheus are great, but on each VM, need to:
* expose the data so open a port --> it's prometheus which will pull the data
* need to run Cadvisor
Here ansible already got the key of VMs, so he does all the job, pull the perf then push data to ELK.

## To improve:
Send directly perf to ELK via syslog, without flat log file and filebeat.

## 1. Top

We use that line to format top:

    PERF=$(top -bn1 | awk '/Cpu/ { cpu= "CPU_used(%):" 100 - $8 };/Mem:/ { mem= "RAM_used(%):" $5 / $3 * 100 };/Swap:/ { swap= "SWAP_used(KB):" $5 }; END { print cpu " | " mem " | " swap}  '); DISK=$(df -t ext4 | awk '/dev/ { print $5+0; exit} '); echo "$(date -Is) | vm:$(hostname) | $PERF | DISK_used(%):$DISK"
    output: 2016-07-04T21:18:41+0200 | vm:server1 | CPU_used(%):4.8 | RAM_used(%):62.6701 | SWAP_used(KB):0 | DISK_used(%):60

## 2. Ansible loop

Create a script file and add the following:

    nano /root/script/perf.sh

You need to be located in your ansible home:

    cd /root/your_ansible_home
    /usr/local/bin/ansible all -m shell -a "PERF\=\$(top -bn1 | awk '/Cpu/ { cpu\= \"CPU_used(%):\" 100 - \$8 };/Mem:/ { mem\= \"RAM_used(%):\" \$5 / \$3 * 100 };/Swap:/ { swap\= \"SWAP_used(KB):\" \$5 }; END { print cpu \" | \" mem \" | \" swap} '); DISK\=\$(df -t ext4 | awk '/dev/ { print \$5+0; exit} '); echo \"\$(date -Is) | vm:\$(hostname) | \$PERF | DISK_used(%):\$DISK\""

Run to check result:

    chmod +x /root/script/perf.sh && /root/script/perf.sh

    backend | SUCCESS | rc=0 >>
       2016-07-04T21:20:51+0200 | vm:backend | CPU_used(%):0.9 | RAM_used(%):76.6379 | SWAP_used(KB):0 | DISK_used(%):53
    dbm | SUCCESS | rc=0 >>
       2016-07-04T19:20:52+0000 | vm:db | CPU_used(%):4 | RAM_used(%):44.179 | SWAP_used(KB):0 | DISK_used(%):19

## 3. Crontab script

    crontab -e

    # run perf
    */10 * * * * /bin/bash /root/script/perf.sh | tee --append /root/script/perf.log > /dev/null 2>/dev/null

## 4. FileBeat --> ELK

### 4a. ELK config

Logstash-Input:

```
input {
  tcp {
    ...
  }
  udp {
    ...
  }
  beats {
    port => "5001"
  }
}
```

Logstash-Filters:

```
filter {

    if [type] == "perf" {
    
      #################################################
      ### perf.log
      #################################################
      if [message] =~ "^\d+-\d+-\d+T\d+:\d+:\d+\+\d+\s+\|\s+vm:.*$" {
        grok {
          match => { "message" => "%{TIMESTAMP_ISO8601:logtime}\s*\|\s*vm:%{HOSTNAME:vm}\s*\|\s*CPU_used\(%\):%{NUMBER:cpu_used_perc:float}\s*\|\s*RAM_used\(%\):%{NUMBER:ram_used_perc:float}\s*\|\s*SWAP_used\(KB\):%{NUMBER:swap_used_kb:float}\s*\|\s*DISK_used\(%\):%{NUMBER:disk_used_perc:float}" }
        }
        date {
          match => [ "logtime", "YYYY-MM-dd'T'HH:mm:ssZ" ]
          timezone => "Europe/Paris"
          #locale => "en"
        }
        mutate {
          add_field => { "@source" => "perf" }
          rename    => [ "source" , "file" ]
        }
      }
      else { drop{} }
    }

}
```

### 4b. FILEBEAT

Create filebeat conf file

    nano /root/filebeat.yml 
```
filebeat:
  prospectors:
    -
      paths:
        - "/mnt/root/script/perf.log"
      document_type: filebeat

output:
  logstash:
    enabled: true
    hosts: ["your_elk_logstash_ip:5001"]
    index: logstash

  console:
    pretty: true
```

Run filebeat

    docker run -d --name filebeat -v /root/filebeat.yml:/filebeat.yml:ro -v /root/script/perf.log:/mnt/root/script/perf.log:ro sbex/filebeat

Import in kibana the dashboard kibana-perf.json


## Checks

Search kibana:

    @source=perf
    
Should return:
```
Time 	message  
July 4th 2016, 22:00:09.000	2016-07-04T22:00:09+0200 | vm:zels | CPU_used(%):1.1 | RAM_used(%):84.096 | SWAP_used(KB):0 | DISK_used(%):41
July 4th 2016, 22:00:08.000	2016-07-04T22:00:08+0200 | vm:databaseslave | CPU_used(%):1.5 | RAM_used(%):96.1679 | SWAP_used(KB):0 | DISK_used(%):27
July 4th 2016, 22:00:07.000	2016-07-04T22:00:07+0200 | vm:logserver | CPU_used(%):22.7 | RAM_used(%):96.8789 | SWAP_used(KB):0 | DISK_used(%):81
July 4th 2016, 22:00:07.000	2016-07-04T20:00:07+0000 | vm:ethereum | CPU_used(%):7.8 | RAM_used(%):96.6918 | SWAP_used(KB):0 | DISK_used(%):32
July 4th 2016, 22:00:07.000	2016-07-04T20:00:07+0000 | vm:stats-and-chats | CPU_used(%):1 | RAM_used(%):91.4329 | SWAP_used(KB):0 | DISK_used(%):71
July 4th 2016, 22:00:07.000	2016-07-04T20:00:07+0000 | vm:backendtrader | CPU_used(%):1.9 | RAM_used(%):87.6561 | SWAP_used(KB):0 | DISK_used(%):42
July 4th 2016, 22:00:06.000	2016-07-04T20:00:06+0000 | vm:sst | CPU_used(%):0.2 | RAM_used(%):76.3923 | SWAP_used(KB):0 | DISK_used(%):30
``` 

filebeat logs:

    Docker logs filebeat

 flat file perf.log:

    cat /root/script/perf.log
    

