---
layout: post
title: Automaticaly Updating PID for Nagios
---

If you are using nagios to monitor your applications in production, you might have faced the need to regularly configure Nagios with constantly changing PIDs. 

We used nagios for monitoring and everytime application servers like tomcat are restarted, nagios needs to be re-configured with new PID. This involves a bit of manual intervention every time and needs the same steps to be followed for re-configuration. Hence why not automate?

Here are the steps that would work to automatically perform this task for tomcat : 

### Configuring Tomcat to persist PID ###

*  Update the setenv.sh add these two lines at the beginning of the script
   {% highlight bash %}
   export CATALINA_BASE="/opt/apache-tomcat-7.0.68"
   CATALINA_PID="$CATALINA_BASE/bin/catalina.pid"
   {% endhighlight %}

   Make sure CATALINA_BASE is the directory where tomcat is present and verify that CATALINA_BASE is reachable.

   Sample : 

   {% highlight bash %}
   $ cat setenv.sh 
   export CATALINA_BASE="/opt/apache-tomcat-7.0.68"
   CATALINA_PID="$CATALINA_BASE/bin/catalina.pid"
   export CATALINA_OPTS="$CATALINA_OPTS -Xms12288m"
   export CATALINA_OPTS="$CATALINA_OPTS -Xmx12288m"
   export CATALINA_OPTS="$CATALINA_OPTS -XX:MaxPermSize=1024m"
   export CATALINA_OPTS="$CATALINA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
   $ ./startup.sh 
   Using CATALINA_BASE:   /opt/apache-tomcat-7.0.68
   Using CATALINA_HOME:   /opt/apache-tomcat-7.0.68
   Using CATALINA_TMPDIR: /opt/apache-tomcat-7.0.68/temp
   Using JRE_HOME:        /usr
   Using CLASSPATH:       /opt/apache-tomcat-7.0.68/bin/bootstrap.jar:
                          /opt/apache-tomcat-7.0.68/bin/tomcat-juli.jar
   Using CATALINA_PID:    /opt/apache-tomcat-7.0.68/bin/catalina.pid
   Tomcat started.
   $ cat catalina.pid 
   5340
   {% endhighlight %}

### Setting up password-less SSH Login  ###

In order to let nagios periodically poll using SSH application servers to check for updated PID, it would need to configure SSH Key-Based Authentication on nagios server. The detailed steps are listed [here](http://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/)

### Updating Nagios Configuration ###

Nagios configuration files are structured file where each block is easily parseable. [Pynag](https://github.com/pynag/pynag) has python modules and utilities for reading and updating Nagios configuration. Following are the steps that a shell script needs to achieve :

* Declare all hosts, PID paths for which nagios configuration has to be reloaded.
* Get the latest PID by SSHing to the host.
* Use Pynag to get the current PID
* Use Pynag to update the PID if PID has changed
* Reload nagios

Here is the crude shell script to do the above steps

{% highlight bash %}
#!/bin/bash

#Declare all hosts for which nagios configuration has to be updated
declare -A hosts
hosts=( ["localhost"]="127.0.0.1" )

#Declare all tomacat location/ script to get PID
declare -A tomcats
tomcats=( ["localhost"]="cat /opt/apache-tomcat-8.0.23/bin/catalina.pid" )

#Start updating the nagios configuration

for key in ${!hosts[@]}; do
    latest_pid=$(ssh root@${hosts[${key}]} ${tomcats[${key}]})
    check_command=$(pynag list check_command where object_type=service and \
    host_name="${key}" --cfg_file=../../tests/dataset01/nagios/nagios.cfg \
    --quiet)
    IFS='!' read -ra check_command_STRINGS <<< "$check_command"
    new_command=""
    delimiter="!"
    length=${#check_command_STRINGS[@]}
    for ((i=0;i<=$length;i++)); do
        if [ $i -eq 1 ];then
            new_command+="$latest_pid"$delimiter
            old_pid=${check_command_STRINGS[$i]}
        elif [ ! -z "${check_command_STRINGS[$i]}" -a \
		"${check_command_STRINGS[$i]}" != " " ]; then
            new_command+="${check_command_STRINGS[$i]}"$delimiter
        fi
    done
    new_check_command=${new_command%?}
    update_command=$(pynag update SET check_command=${new_check_command} \
    WHERE host_name="${key}" and object_type=service \
    --cfg_file=../../tests/dataset01/nagios/nagios.cfg --force)
    details_after_update=$(pynag list host_name check_command where \
    object_type=service and host_name="${key}" \
    --cfg_file=../../tests/dataset01/nagios/nagios.cfg --quiet)
    echo "$details_after_update"
done

#When updated, reload nagios
{% endhighlight %}

### Setting up cron job ###
The last step to automate would be to setup a cron job to execute this shell script periodically.
