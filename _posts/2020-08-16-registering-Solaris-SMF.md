---
layout: post
author: Breno Cesar
title:  "Registering your own solaris service management facility"
subtitle: "If you came from linux world and are used to register a systemd service in systemcl, lets check it out how this works in a solaris 10/11 environment"
categories: solaris
date: 2020-08-16 19:07:00
---
If you are used to register a service in a systemd based environment, you known what is a systemcl, what he does and how to configure then to managed a new custom deamon/service/script/what_ever_you_want.
<br>
In a solaris environment is not different, as almost linux/unix like system, it has first and unique PID raised by the system at boot, that is the father for all the others pid that handle the other system services.
<br>
To manage all those services and deamons, in systemd linux system whe have systemctl, and in solaris system, we have the **service management facility**, or [**smf**](https://www.oracle.com/technical-resources/articles/solaris/intro-smf-basics-s11.html).
<br>
SMF use a xml file to configure all the services, if you want to know how a service is configure, you can just type:
<br>
```
-bash-3.2# svccfg export -a /system/cron
```

It will print out a XML file that show's up how this service is configured:
<br>

```xml
<?xml version='1.0'?>
<!DOCTYPE service_bundle SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type='manifest' name='export'>
  <service name='system/cron' type='service' version='0'>
    <create_default_instance enabled='true'/>
    <single_instance/>
    <dependency name='usr' grouping='require_all' restart_on='none' type='service'>
      <service_fmri value='svc:/system/filesystem/local'/>
    </dependency>
    <dependency name='ns' grouping='require_all' restart_on='none' type='service'>
      <service_fmri value='svc:/milestone/name-services'/>
    </dependency>
    <dependent name='cron_multi-user' restart_on='none' grouping='optional_all'>
      <service_fmri value='svc:/milestone/multi-user'/>
    </dependent>
    <exec_method name='start' type='method' exec='/lib/svc/method/svc-cron' timeout_seconds='60'>
      <method_context>
        <method_credential user='root' group='root'/>
      </method_context>
    </exec_method>
    <exec_method name='stop' type='method' exec=':kill' timeout_seconds='60'/>
    <exec_method name='refresh' type='method' exec=':kill -THAW' timeout_seconds='60'/>
    <property_group name='general' type='framework'>
      <propval name='action_authorization' type='astring' value='solaris.smf.manage.cron'/>
    </property_group>
    <property_group name='startd' type='framework'>
      <propval name='ignore_error' type='astring' value='core,signal'/>
    </property_group>
    <stability value='Unstable'/>
    <template>
      <common_name>
        <loctext xml:lang='C'>clock daemon (cron)</loctext>
      </common_name>
      <documentation>
        <manpage title='cron' section='1M' manpath='/usr/share/man'/>
        <manpage title='crontab' section='1' manpath='/usr/share/man'/>
      </documentation>
    </template>
  </service>
</service_bundle>
```
Looking this file, it's hard to debug and understand how it works , but here i will compiling the most usual configurations that you need to know for configuring your own SMF service.
<br>
Below i will show to ho configure a script called "my-application" in the smf, and which are the keys part to do so.
<br>
Firs we need to create a file called "my-app.xml" and than configure each step as bellow.
<br>

### 1 - Header

The first thing you need to do is configure the name of your new service, and this going in to the ```name``` option under ```service_bundle``` and ```service``` section.
All other options can let be standart as showed bellow.
```xml
<?xml version="1.0" ?>
<!DOCTYPE service_bundle
  SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type="manifest" name="application/my-application">
    <service version="1" type="service" name="application/my-application">
.
.
.
    </service>
</service_bundle>
```

### 2 - Dependency
Here we will not waste too much time, basicaly 99,99% of all script/software start only when a environment is runnig under a multi user level / rc.3 / init 3 or what ever you call it in your system.
This is configured under the ```dependency``` section.
```xml
<?xml version="1.0" ?>
<!DOCTYPE service_bundle
  SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type="manifest" name="application/my-application">
    <service version="1" type="service" name="application/my-application">
        <dependency restart_on="none" type="service" name="multi_user_dependency" grouping="require_all">
            <service_fmri value="svc:/milestone/multi-user"/>
        </dependency>
.
.
.
    </service>
</service_bundle>
```
### 3 - User, Enivonment Variable and Path

Here is where the magic starts, if you what to specify globally:
- Which user/group from the system the app will be executed: ```method_credential``` section.
- Which path the script should be executed:```method_context``` section.
- Which environment variable should be used: ```method_environment``` section.

Let's supose that the my-application should be executed with a user called "karlson" in the group "other" over a path "/home/karson" so the block configuration should be like:
```xml
<method_context method_context working_directory='/home/karson'>
    <method_credential user="karlson" group="other" />
</method_context>
```
And for everything that SMF will execute in our script, we will user this java home environment variable:
```xml
<method_environment>
    <envvar name="JAVA_HOME" value="/usr/jdk/instances/jdk1.8.0" />
</method_environment>
```
Until now our block configuration look's like this.
```xml
<?xml version="1.0" ?>
<!DOCTYPE service_bundle
  SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type="manifest" name="application/my-application">
    <service version="1" type="service" name="application/my-application">
        <dependency restart_on="none" type="service" name="multi_user_dependency" grouping="require_all">
            <service_fmri value="svc:/milestone/multi-user"/>
        </dependency>
        <method_context working_directory='/home/karson'>
            <method_credential user="karlson" group="other" />
            <method_environment>
              <envvar name="JAVA_HOME" value="/usr/jdk/instances/jdk1.8.0" />
            </method_environment>
        </method_context>
.
.
.
    </service>
</service_bundle>
```
### 4 - Script execution
Here we specify how the script should be executed, there is a lot of variation for that, but i will show the 2 most common.
<br>

#### 4.1 - Start/Stop script
For a script that you need to inform a input a parameter like **start** or **stop**, just use the block bellow:

```xml
<exec_method timeout_seconds="60" type="method" name="start" exec="/home/karson/my-app.sh %m"/>
<exec_method timeout_seconds="60" type="method" name="stop" exec="/home/karson/my-app.sh %m"/>
<exec_method timeout_seconds="60" type="method" name="refresh" exec=":true"/>
```
obs: Note that you can user varibles like ```%m``` to specify which input you will use for every method.

#### 4.2 - Daemon script
For a script that is executed as a deamon and should be maintained executing, just use the block bellow:

```xml
<exec_method timeout_seconds="60" type="method" name="start" exec="/home/karson/my-app.sh -c /home/karson/my-app.conf"/>
<exec_method timeout_seconds="60" type="method" name="stop" exec=":kill"/>
<exec_method timeout_seconds="60" type="method" name="refresh" exec=":true"/>
```

### 5 - Setting the name and description 

Inform to the smf the name and description for your script, so every user that type ```svcs -l my-app``` for your script will know what is about our script.
For that, we will use the block bellow

```xml
<template>
  <common_name>
     <loctext xml:lang="C">
        My APP
     </loctext>
  </common_name>
  <description>
    <loctext xml:lang="C">
      My awesome application
    </loctext>
  </description>
</template>
```
### 6 - Importing the configuration on SMF
Finaly our app will be configured in the my-app.xml file as a deamon service, so the resulting file will looks like:
 
```xml
<?xml version="1.0" ?>
<!DOCTYPE service_bundle
  SYSTEM '/usr/share/lib/xml/dtd/service_bundle.dtd.1'>
<service_bundle type="manifest" name="application/my-application">
    <service version="1" type="service" name="application/my-application">
        <dependency restart_on="none" type="service" name="multi_user_dependency" grouping="require_all">
            <service_fmri value="svc:/milestone/multi-user"/>
        </dependency>
        <method_context working_directory='/home/karson'>
            <method_credential user="karlson" group="other" />
            <method_environment>
              <envvar name="JAVA_HOME" value="/usr/jdk/instances/jdk1.8.0" />
            </method_environment>
        </method_context>
        <exec_method timeout_seconds="60" type="method" name="start" exec="/home/karson/my-app.sh -c /home/karson/my-app.conf"/>
        <exec_method timeout_seconds="60" type="method" name="stop" exec=":kill"/>
        <exec_method timeout_seconds="60" type="method" name="refresh" exec=":true"/>
        <template>
          <common_name>
             <loctext xml:lang="C">
                My APP
             </loctext>
          </common_name>
          <description>
            <loctext xml:lang="C">
              My awesome application
            </loctext>
          </description>
        </template>
    </service>
</service_bundle>
```

Lets import it with the command bellow:
```
-bash-3.2# svccfg import ./my-app.xml
```

And then start our script:
```
-bash-3.2# svcadm enable my-application
```

I hope that you enjoyed, and be usefull for you as well.

See you !!

[]'s