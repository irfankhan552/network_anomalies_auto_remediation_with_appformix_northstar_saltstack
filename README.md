# Demo overview:  

- Appformix is used for network devices monitoring (SNMP and telemetry).  
- Appformix send webhook notifications to Salt master 
- SaltStack automatically makes REST calls to Northstar SDN controller to put the "faulty" device in maintenance mode.  
    - The "faulty" device will be considered logically down for a certain amount time, and the SDN controller will reroute the
    LSPs around this device during the maintenance period.  
    - After the maintenance period, LSPs are reverted back to optimal paths. 
    
![unplanned-maintenance-operations.png](resources/unplanned-maintenance-operations.png)  


# Demo building blocks: 

- Juniper devices
- Northstar SDN controller. Version 4 or above is required
- Appformix
- SaltStack

# webhooks overview: 

- A webhook is notification using an HTTP POST. A webhook is sent by a system A to push data (json body as example) to a system B when an event occurred in the system A. Then the system B will decide what to do with these details.  
- Appformix supports webhooks. A notification is generated when the condition of an alarm is observed. You can configure an alarm to post notifications to an external HTTP endpoint. AppFormix will post a JSON payload to the endpoint for each notification.
- SaltStack can listens to webhooks and generate equivalents ZMQ messages to the event bus  
- SaltStack can reacts to webhooks 

# Building blocks role: 

## Appformix:  
- Collects data from Junos devices.
- Generates webhooks notifications (HTTP POST with a JSON body) to SaltStack when the condition of an alarm is observed. The JSON body provides the device name and other details. 

## SaltStack: 
- Only the master is required.   
- Listens to webhooks 
- Generates a ZMQ messages to the event bus when a webhook notification is received. The ZMQ message has a tag and data. The data structure is a dictionary, which contains information about the event.
- The reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions. 
- The reactor sls file used in this content does the following: 
    - It parses the data from the ZMQ message and extracts the network device name. 
    - It then passes the data extracted the ZMQ message to a runner and execute the runner. The runner makes REST calls to Northstar SDN controller to put the "faulty" device in maintenance mode. 

## Gitlab  
- This SaltStack setup uses a gitlab server for external pillars (variables) and as a remote file server (templates, sls files, ...).  

## Northstar: 
- Handle the REST calls received by SaltStack, i.e put the "faulty" device in maintenance mode. 
    - The "faulty" device will be considered logically down for a certain amount time, and Northstar will reroute the LSPs around this device during the maintenance period. 
    - After the maintenance period, LSPs are reverted back to optimal paths. 

# Requirements: 

- Install appformix
- Configure appformix for network devices monitoring
- Install northstar (version 4 or above)
- Add the same network devices to northstar 
- Install SaltStack

# Prepare the demo: 

## Appformix  

### Install Appformix. 
This is not covered by this documentation

### Configure Appformix for network devices monitoring 

Appformix supports network devices monitoring using SNMP and JTI (Juniper Telemetry Interface) native streaming telemetry.  
- For SNMP, the polling interval is 60s.  
- For JTI streaming telemetry, Appformix automatically configures the network devices. The interval configured on network devices is 60s.  

Here's the [**documentation**](https://www.juniper.net/documentation/en_US/appformix/topics/concept/appformix-ansible-configure-network-device.html)  

In order to configure AppFormix for network devices monitoring, here are the steps:
- manage the 'network devices json configuration' file. This file is used to define the list of devices you want to monitor using Appformix, and the details you want to collect from them.    
- Indicate to the 'Appformix installation Ansible playbook' which 'network devices json configuration file' to use. This is done by setting the variable ```network_device_file_name``` in ```group_vars/all```
- Set the flag to enable appformix network device monitor. This is done by setting the variable ```appformix_network_device_monitoring_enabled``` to ```true``` in ```group_vars/all```
- Enable the Appformix plugins for network devices monitoring. This is done by setting the variable ```appformix_plugins``` in ```group_vars/all```
- re run the 'Appformix installation Ansible playbook'.

Here's how to manage the 'network devices json configuration file' with automation:  
Define the list of devices you want to monitor using Appformix, and the details you want to collect from them:    
```
vi configure_appformix/network_devices.yml
```

Execute the python script [**network_devices.py**](configure_appformix/network_devices.py). It renders the template [**network_devices.j2**](configure_appformix/network_devices.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**network_devices.json**](configure_appformix/network_devices.json).  
```
python configure_appformix/network_devices.py
```
```
more configure_appformix/network_devices.json
```

From your appformix directory, update ```group_vars/all``` file: 
```
cd appformix-2.15.2/
vi group_vars/all
```
to make sure it contains this:
```
network_device_file_name: /path_to/network_devices.json
appformix_network_device_monitoring_enabled: true
appformix_jti_network_device_monitoring_enabled: true
appformix_plugins:
   - plugin_info: 'certified_plugins/jti_network_device_usage.json'
   - plugin_info: 'certified_plugins/snmp_network_device_routing_engine.json'
   - plugin_info: 'certified_plugins/snmp_network_device_usage.json'
```

Then, from your appformix directory, re-run the 'Appformix installation Ansible playbook':
```
cd appformix-2.15.2/
ansible-playbook -i inventory appformix_standalone.yml
```
### Configure the network devices with the SNMP community used by Appformix

You need to configure the network devices with the SNMP community used by Appformix. The script [**snmp.py**](configure_junos/snmp.py) renders the template [**snmp.j2**](configure_junos/snmp.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**snmp.conf**](configure_junos/snmp.conf). This file is then loaded and committed on all network devices used with SNMP monitoring.
 
```
python configure_junos/snmp.py
configured device 172.30.52.85 with snmp community public
configured device 172.30.52.86 with snmp community public
```
```
more configure_junos/snmp.conf
```

### Configure the network devices for JTI telemetry

For JTI native streaming telemetry, Appformix uses NETCONF to automatically configure the network devices:  
```
lab@vmx-1-vcp> show system commit
0   2018-03-22 16:32:37 UTC by lab via netconf
1   2018-03-22 16:32:33 UTC by lab via netconf
```
```
lab@vmx-1-vcp> show configuration | compare rollback 1
[edit services analytics]
+    sensor Interface_Sensor {
+        server-name appformix-telemetry;
+        export-name appformix;
+        resource /junos/system/linecard/interface/;
+    }
```
```
lab@vmx-1-vcp> show configuration | compare rollback 2
[edit]
+  services {
+      analytics {
+          streaming-server appformix-telemetry {
+              remote-address 172.30.52.157;
+              remote-port 42596;
+          }
+          export-profile appformix {
+              local-address 192.168.1.1;
+              local-port 21112;
+              dscp 20;
+              reporting-rate 60;
+              format gpb;
+              transport udp;
+          }
+          sensor Interface_Sensor {
+              server-name appformix-telemetry;
+              export-name appformix;
+              resource /junos/system/linecard/interface/;
+          }
+      }
+  }

lab@vmx-1-vcp>
```
Run this command to show the installed sensors: 
```
lab@vmx-1-vcp> show agent sensors
```

If Appformix has serveral ip addresses, and you want to configure the network devices to use a different IP address than the one configured by appformix for telemetry server, execute the python script [**telemetry.py**](configure_junos/telemetry.py). 
The python script [**telemetry.py**](configure_junos/telemetry.py) renders the template [**telemetry.j2**](configure_junos/telemetry.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**telemetry.conf**](configure_junos/telemetry.conf). This file is then loaded and committed on all network devices used with JTI telemetry.  

```
more configure_appformix/network_devices.yml
```
```
python configure_junos/telemetry.py
configured device 172.30.52.155 with telemetry server ip 192.168.1.100
configured device 172.30.52.156 with telemetry server ip 192.168.1.100
```
```
# more configure_junos/telemetry.conf
set services analytics streaming-server appformix-telemetry remote-address 192.168.1.100
```
Verify on your network devices: 
```
lab@vmx-1-vcp> show configuration services analytics streaming-server appformix-telemetry remote-address
remote-address 192.168.1.100;

lab@vmx-1-vcp> show configuration | compare rollback 1
[edit services analytics streaming-server appformix-telemetry]
-    remote-address 172.30.52.157;
+    remote-address 192.168.1.100;

lab@vmx-1-vcp> show system commit
0   2018-03-23 00:34:47 UTC by lab via netconf

```
```
lab@vmx-1-vcp> show agent sensors

Sensor Information :

    Name                                    : Interface_Sensor
    Resource                                : /junos/system/linecard/interface/
    Version                                 : 1.1
    Sensor-id                               : 150000323
    Subscription-ID                         : 562950103421635
    Parent-Sensor-Name                      : Not applicable
    Component(s)                            : PFE

    Server Information :

        Name                                : appformix-telemetry
        Scope-id                            : 0
        Remote-Address                      : 192.168.1.100
        Remote-port                         : 42596
        Transport-protocol                  : UDP

    Profile Information :

        Name                                : appformix
        Reporting-interval                  : 60
        Payload-size                        : 5000
        Address                             : 192.168.1.1
        Port                                : 21112
        Timestamp                           : 1
        Format                              : GPB
        DSCP                                : 20
        Forwarding-class                    : 255

```


## Northstar 

### Install Northstar (version 4 or above)  
This is not covered by this documentation

### Add the same network devices to Northstar  
This is not covered by this documentation

## Gitlab

This SaltStack setup uses a gitlab server for external pillars and as a remote file server.  

### Install Gitlab

There is a Gitlab docker image available https://hub.docker.com/r/gitlab/gitlab-ce/

You first need to install docker. This step is not covered by this documentation.  

Then:  

Pull the image: 
```
# docker pull gitlab/gitlab-ce
```

Verify: 
```
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-ce             latest              09b815498cc6        6 months ago        1.33GB
```

Instanciate a container: 
```
docker run -d --rm --name gitlab -p 9080:80 gitlab/gitlab-ce
```
Verify:
```
# docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                PORTS                                                 NAMES
9e8330425d9c        gitlab/gitlab-ce             "/assets/wrapper"        5 months ago        Up 5 days (healthy)   443/tcp, 0.0.0.0:3022->22/tcp, 0.0.0.0:9080->80/tcp   gitlab
```

### Configure Gitlab

Create the organization ```organization```.    
Create the repositories ```network_parameters``` and ```network_model``` in the organization ```organization```.      
The repository ```network_parameters``` is used for SaltStack external pillars.    
The repository ```network_model``` is used as an external file server for SaltStack   



## SaltStack

### Install SaltStack
This is not covered by this documentation

### Configure the Salt master configuration file

ssh to the Salt master and edit the salt master configuration file:  
```
vi /etc/salt/master
```

Make sure the master configuration file has these details:  
```
runner_dirs:
  - /srv/runners
```
```
engines:
  - webhook:
      port: 5001
```
```
ext_pillar:
  - git:
    - master git@gitlab:organization/network_parameters.git
```
```
fileserver_backend:
  - git
  - roots
```
```
gitfs_remotes:
  - ssh://git@gitlab/organization/network_model.git
```
```
file_roots:
  base:
    - /srv/salt
    - /srv/local

```
So: 
- the Salt master is listening webhooks on port 5001. It generates equivalents ZMQ messages to the event bus
- runners are in the directory ```/srv/runners``` on the Salt master
- pillars (humans defined variables) are in the gitlab repository ```organization/network_parameters``` 
- the gitlab repository ```organization/network_model``` is a file server for SaltStack (sls files, templates, ...) 

### Update the Salt external pillars

Create a file ```northstar.sls``` at the root of the  external pillars gitlab repository ```organization/network_parameters``` with this content: 
```
northstar: 
    authuser: 'admin'
    authpwd: 'juniper123'
    url: 'https://192.168.128.173:8443/oauth2/token'
    url_base: 'http://192.168.128.173:8091/NorthStar/API/v2/tenant/'
    maintenance_event_duration: 60
```
The runner that SaltStack will execute to make REST calls to northstar will use these variables.  


For the ```northstar.sls``` file to be actually used, update the ```top.sls``` file at the root of the gitlab repository ```organization/network_parameters```.  
Example:  
```
{% set id = salt['grains.get']('id') %} 
{% set host = salt['grains.get']('host') %} 

base:
  '*':
    - production
    - northstar

{% if host == '' %}
  '{{ id }}':
    - {{ id }}
{% endif %}
```


### Update the Salt reactor

The reactor binds sls files to event tags.  
The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run.  
So these sls files define the SaltStack reactions.  
This reactor binds ```salt/engines/hook/appformix_to_saltstack``` event to this file ```/srv/reactor/northstar_maintenance.sls``` 

```
# more /etc/salt/master.d/reactor.conf
reactor:
   - 'salt/engines/hook/appformix_to_saltstack':
       - /srv/reactor/northstar_maintenance.sls

```

Restart the Salt master:
```
service salt-master stop
service salt-master start
```

The command ```salt-run reactor.list``` lists currently configured reactors:  
```
salt-run reactor.list
event:
    ----------
    _stamp:
        2018-04-10T13:40:12.062140
suffix:
    salt/reactors/manage/list
|_
  ----------
  salt/engines/hook/appformix_to_saltstack:
      - /srv/reactor/northstar_maintenance.sls
```

### Create the reactor sls file 

- The sls reactor file ```/srv/reactor/northstar_maintenance.sls``` parses the data from the ZMQ message that has the tags ```salt/engines/hook/appformix_to_saltstack``` and extracts the network device name.  
- It then passes the data extracted the ZMQ message to the python function ```put_device_in_maintenance``` of the ```northstar``` runner and execute the python function. 

```
# more /srv/reactor/northstar_maintenance.sls
{% set body_json = data['body']|load_json %}
{% set devicename = body_json['status']['entityId'] %}
test_event:
  runner.northstar.put_device_in_maintenance:
    - args:
       - dev: {{ devicename }}
```

### Create the Salt runner

As you can see in the Salt master configuration file ```/etc/salt/master```, the runners directory is ```/srv/runners/```.   
So the runner ```northstar``` is ```/srv/runners/northstar.py```  

This runner defines a python function ```put_device_in_maintenance```.  
The python function makes REST calls to Northstar SDN controller to put a device in maintenance mode.  

The device will be considered logically down for a certain amount time, and the SDN controller will reroute the LSPs around this device during the maintenance period.  
After the maintenance period, LSPs are reverted back to optimal paths. 

```
# more /srv/runners/northstar.py
from datetime import timedelta, datetime
import requests
from requests.packages.urllib3.exceptions import InsecureRequestWarning
from requests.auth import HTTPBasicAuth
from pprint import pprint
import yaml
import json
import salt.client
import salt.config
import salt.runner

def put_device_in_maintenance(dev):
 opts = salt.config.master_config('/etc/salt/master')
 caller = salt.client.Caller()
 local_minion_id = caller.cmd('grains.get', 'id')
 runner = salt.runner.RunnerClient(opts)
 pillar = runner.cmd('pillar.show_pillar', [local_minion_id])
 url = pillar['northstar']['url']
 maintenance_event_duration = pillar['northstar']['maintenance_event_duration']
 url_base = pillar['northstar']['url_base']
 authuser = pillar['northstar']['authuser']
 authpwd = pillar['northstar']['authpwd']
 requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
 headers = { 'content-type' : 'application/json'}
 data_to_get_token = {"grant_type":"password","username":authuser,"password":authpwd}
 r = requests.post(url, data=json.dumps(data_to_get_token), auth=(authuser, authpwd), headers=headers, verify=False)
 headers = {'Authorization':str('Bearer ' + r.json()['access_token']), 'Accept' : 'application/json', 'Content-Type' : 'application/json'}
 url = url_base + '1/topology/1/nodes'
 r = requests.get(url, headers=headers, verify=False)
 for i in r.json():
   if i['hostName'] == dev:
    node_index = i['nodeIndex']
 maintenance_url = url_base + '1/topology/1/maintenances'
 maintenance_data = {
     "topoObjectType": "maintenance",
     "topologyIndex": 1,
     "user": "admin",
     "name": "event_" + dev,
     "startTime": datetime.now().isoformat(),
     "endTime": (datetime.now() + timedelta(minutes=maintenance_event_duration)).isoformat(),
     "elements": [{"topoObjectType": "node", "index": node_index}]
     }
 m_res = requests.post(maintenance_url, data=json.dumps(maintenance_data), headers=headers, verify=False)
 return "done"
```

You can test manually the runner.  
```
salt-run northstar.put_device_in_maintenance dev=core-rtr-p-02
```

Then log in to the Northstar GUI and verify in the ```topology``` menu if the device ```core-rtr-p-02``` is in maintenance. 



# Run the demo: 

## Create Appformix webhook notifications.  

You can do it from Appformix GUI. Select:  
- settings
- Notification Settings
- Notification Services
- add service.    
    - service name: appformix_to_saltstack  
    - URL endpoint: provide the Salt master IP and Salt webhook listerner port (```HTTP://192.168.128.174:5001/appformix_to_saltstack``` as example).  
    - setup  


## Create Appformix alarms, and map these alarms to the webhook you just created.

You can do it from the Appformix GUI. Select:   
- Alarms
- add rule
    - Name: in_unicast_packets_core-rtr-p-02,  
    - Module: Alarms,  
    - Alarm rule type: Static,  
    - scope: network devices,  
    - network device/Aggregate: core-rtr-p-02,  
    - generate: generate alert,  
    - For metric: interface_in_unicast_packets,  
    - When: Average,  
    - Interval(seconds): 60,  
    - Is: select Above,  
    - Threshold(Packets/s): 300,  
    - Severity: Warning,  
    - notification: custom service,  
    - services: appformix_to_saltstack, 
    - save.

## Watch webhook notifications and ZMQ messages  

Run this command on the master to see webhook notifications:
```
# tcpdump port 5001 -XX 
```

Salt provides a runner that displays events in real-time as they are received on the Salt master:  
```
# salt-run state.event pretty=True
```

## Trigger an alarm  to get a webhook notification sent by Appformix to SaltStack 

Either you DIY, or, depending on the alarms you set, you can use one the automation content available in the directory [trigger_alarms](trigger_alarms).  

### Requirement to use the automation content available in the directory [trigger_alarms](trigger_alarms)

There is a SaltStack requirement to use the automation content available in the directory [trigger_alarms](trigger_alarms). You first need to install on the master or on a minion the dependencies to use a SaltStack proxy for Junos. And then you need to start one junos proxy daemon per device. These details are not covered by this documentation.  

Here's how to use the automation content available in the directory [trigger_aarms](trigger_alarms).  

### generate traffic between 2 routers 
Add the file [generate_traffic.sls](trigger_alarms/generate_traffic.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  

And run this command on the master:   
```
# salt "core-rtr-p-02" state.apply junos.generate_traffic
```
### Change interface speed on a router

Add the file [change_int_speed.sls](trigger_alarms/change_int_speed.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  
Add the file [speed.set](trigger_alarms/speed.set) to the directory ```template``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).    
Run this command on the master:   
```
# salt "core-rtr-p-02" state.apply junos.change_int_speed
# salt "core-rtr-p-02" junos.cli "show system commit"
# salt "core-rtr-p-02" junos.cli "show configuration | compare rollback 1"
# salt "core-rtr-p-02" junos.cli "show configuration interfaces ge-0/0/1"
```

### Change MTU on a router

Add the file [change_mtu.sls](trigger_alarms/change_mtu.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  
Add the file [mtu.set](trigger_alarms/mtu.set) to the directory ```template``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).    
Run this command on the master:   
```
# salt "core-rtr-p-02" state.apply junos.change_mtu
# salt "core-rtr-p-02" junos.cli "show system commit"
# salt "core-rtr-p-02" junos.cli "show configuration | compare rollback 1"
# salt "core-rtr-p-02" junos.cli "show configuration interfaces ge-0/0/1"
```

## Verify on SaltStack 

Have a look at the tcpdump output 

Have a look at the the ZMQ messages

```
# salt-run state.event pretty=True
salt/engines/hook/appformix_to_saltstack        {
    "_stamp": "2018-04-06T16:43:39.866009",
    "body": "{\"status\": {\"description\": \"NetworkDevice core-rtr-p-02: average ifInUcastPkts above 300 {u'ge-0/0/4_0': {u'sample_value': 15.48, u'status': u'inactive'}, u'demux0': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/5': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/4': {u'sample_value': 15.48, u'status': u'inactive'}, u'ge-0/0/3': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/2': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/1': {u'sample_value': 17.78, u'status': u'active'}, u'ge-0/0/0': {u'sample_value': 0, u'status': u'inactive'}, u'pfe-0/0/0_16383': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/9': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/8': {u'sample_value': 0, u'status': u'inactive'}, u'fxp0_0': {u'sample_value': 98.63, u'status': u'inactive'}, u'tap': {u'sample_value': 0, u'status': u'inactive'}, u'em1_0': {u'sample_value': 844.7, u'status': u'active'}, u'pfh-0/0/0_16384': {u'sample_value': 0, u'status': u'inactive'}, u'pip0': {u'sample_value': 0, u'status': u'inactive'}, u'pimd': {u'sample_value': 0, u'status': u'inactive'}, u'mtun': {u'sample_value': 0, u'status': u'inactive'}, u'gre': {u'sample_value': 0, u'status': u'inactive'}, u'em1': {u'sample_value': 844.7, u'status': u'active'}, u'em2': {u'sample_value': 0, u'status': u'inactive'}, u'lc-0/0/0': {u'sample_value': 0, u'status': u'inactive'}, u'irb': {u'sample_value': 0, u'status': u'inactive'}, u'lo0_16385': {u'sample_value': 47.6, u'status': u'inactive'}, u'dsc': {u'sample_value': 0, u'status': u'inactive'}, u'fxp0': {u'sample_value': 98.63, u'status': u'inactive'}, u'vtep': {u'sample_value': 0, u'status': u'inactive'}, u'cbp0': {u'sample_value': 0, u'status': u'inactive'}, u'jsrv': {u'sample_value': 0, u'status': u'inactive'}, u'rbeb': {u'sample_value': 0, u'status': u'inactive'}, u'lo0_0': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/7': {u'sample_value': 0, u'status': u'inactive'}, u'pp0': {u'sample_value': 0, u'status': u'inactive'}, u'pfh-0/0/0': {u'sample_value': 0, u'status': u'inactive'}, u'pfe-0/0/0': {u'sample_value': 0, u'status': u'inactive'}, u'lsi': {u'sample_value': 0, u'status': u'inactive'}, u'pfh-0/0/0_16383': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/1_0': {u'sample_value': 17.78, u'status': u'active'}, u'lc-0/0/0_32769': {u'sample_value': 0, u'status': u'inactive'}, u'NetworkDeviceId': u'core-rtr-p-02', u'ipip': {u'sample_value': 0, u'status': u'inactive'}, u'lo0_16384': {u'sample_value': 0, u'status': u'inactive'}, u'ge-0/0/6': {u'sample_value': 0, u'status': u'inactive'}, u'lo0': {u'sample_value': 47.6, u'status': u'inactive'}, u'pime': {u'sample_value': 0, u'status': u'inactive'}, u'esi': {u'sample_value': 0, u'status': u'inactive'}, u'em2_32768': {u'sample_value': 0, u'status': u'inactive'}, u'jsrv_1': {u'sample_value': 0, u'status': u'inactive'}}\", \"timestamp\": 1523033019000, \"entityType\": \"network_device\", \"state\": \"active\", \"entityDetails\": {}, \"entityId\": \"core-rtr-p-02\", \"metaData\": {}}, \"kind\": \"Alarm\", \"spec\": {\"aggregationFunction\": \"average\", \"intervalDuration\": 60, \"severity\": \"warning\", \"module\": \"alarms\", \"intervalCount\": 1, \"metricType\": \"ifInUcastPkts\", \"name\": \"ping_in_unicast_packets\", \"eventRuleId\": \"f9e6102e-39b8-11e8-b8ce-0242ac120005\", \"mode\": \"alert\", \"intervalsWithException\": 1, \"threshold\": 300, \"comparisonFunction\": \"above\"}, \"apiVersion\": \"v1\"}",
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Connection": "keep-alive",
        "Content-Length": "3396",
        "Content-Type": "application/json",
        "Host": "192.168.128.174:5001",
        "User-Agent": "python-requests/2.18.4"
    }
}
salt/run/20180406164340760297/new       {
    "_stamp": "2018-04-06T16:43:40.762533",
    "fun": "runner.northstar.put_device_in_maintenance",
    "fun_args": [
        {
            "dev": "core-rtr-p-02"
        }
    ],
    "jid": "20180406164340760297",
    "user": "Reactor"
}
...
...
...
salt/run/20180406164340760297/ret       {
    "_stamp": "2018-04-06T16:43:43.143082",
    "fun": "runner.northstar.put_device_in_maintenance",
    "fun_args": [
        {
            "dev": "core-rtr-p-02"
        }
    ],
    "jid": "20180406164340760297",
    "return": "done",
    "success": true,
    "user": "Reactor"
}
```
## Verify on Northstar 

Then log in to the Northstar GUI and verify in the topology menu if the device core-rtr-p-02 is in maintenance. 
![Northstar_maintenance.png](resources/Northstar_maintenance.png)  

