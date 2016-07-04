# 1line-perf-monitoring
Native monitor CPU, RAM, DISK, SWAP (via Top) for linux VMs, send to a file, then send to ELK for dashboards.


# How it work:
1. Top formated will display all values.
2. We will loop on ansible host file: contains the list of our vms (but you can adapt and loop on a flat file if you don't use ansible)
3. A cron will run that command and send all in a flat file.
4. This flat file is pickup by beats and send to ELK for dashboard.

# 1 Top

We use that line to format top:

    PERF=$(top -bn1 | awk '/Cpu/ { cpu= "CPU_used(%):" 100 - $8 };/Mem:/ { mem= "RAM_used(%):" $5 / $3 * 100 };/Swap:/ { swap= "SWAP_used(KB):" $5 }; END { print cpu " | " mem " | " swap}  '); DISK=$(df -t ext4 | awk '/dev/ { print $5+0; exit} '); echo "$(date -Is) | vm:$(hostname) | $PERF | DISK_used(%):$DISK"
    output: 2016-07-04T21:18:41+0200 | vm:server1 | CPU_used(%):4.8 | RAM_used(%):62.6701 | SWAP_used(KB):0 | DISK_used(%):60

# 2 Ansible loop

Create a script file and add the following:
    nano /root/script/perf.sh

You need to be located in ansible home:

    cd /root/your_ansible_home
    /usr/local/bin/ansible all -m shell -a "PERF\=\$(top -bn1 | awk '/Cpu/ { cpu\= \"CPU_used(%):\" 100 - \$8 };/Mem:/ { mem\= \"RAM_used(%):\" \$5 / \$3 * 100 };/Swap:/ { swap\= \"SWAP_used(KB):\" \$5 }; END { print cpu \" | \" mem \" | \" swap} '); DISK\=\$(df -t ext4 | awk '/dev/ { print \$5+0; exit} '); echo \"\$(date -Is) | vm:\$(hostname) | \$PERF | DISK_used(%):\$DISK\""

Run to check result:
    /root/script/perf.sh

    backend | SUCCESS | rc=0 >>
       2016-07-04T21:20:51+0200 | vm:backend | CPU_used(%):0.9 | RAM_used(%):76.6379 | SWAP_used(KB):0 | DISK_used(%):53
    dbm | SUCCESS | rc=0 >>
       2016-07-04T19:20:52+0000 | vm:db | CPU_used(%):4 | RAM_used(%):44.179 | SWAP_used(KB):0 | DISK_used(%):19

# 3. Crontab script

    crontab -e

    # run perf
    */10 * * * * /bin/bash /root/script/perf.sh | tee --append /root/script/perf.log > /dev/null 2>/dev/null

# 4. Beat --> ELK

