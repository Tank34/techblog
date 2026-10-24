---
title: "Basic ARA-server monitoring with Zabbix"
description: "Creating a Zabbix template to create some assurance for my Ansible playbooks"
date: 2026-10-24T10:00:00+02:00
tags: ["zabbix", "ara-server", "ansible", "openwrt"]
categories: ["Monitoring", "Automation"]
draft: true  
---

# Basic ARA-server monitoring with Zabbix

Trying to got some basic ARA server playbook run assurance. ARA provides a good API, we can poll from Zabbix. Standardised working with pulling data from Zabbix agents tends to keep me away from any push model towards the Zabbix server..

Since I am monitoring the playbook-running host with the Zabbix agent ( and zabbix HTTP agent runs from server/proxy), what I need are some userparameters.

## What we want to know:
1. I want to be sure the ARA service API is available. Because an issue here will fail all the other items.
2. I want assurance that a certain playbook (name/path) has been run in the last 24hours. So I need to configure static parameters with API endpoint filters as parameters (which I will enter in the Zabbix gui). This is because I am using cronjobs for playbooks, and I am not monitoring these in any other way.
2. Additional trigger per ARA-hosts, when a certain playbook/task fails on that host. Here I will implement discovery for all hosts in ARA.

Basically all other data is accessible in the ARA interface when needed, and I don't need any more data collection, triggers or assurance.


## Implementation
1. ARA service API assurance
For this I can use the web.page builtin Zabbix agent item type.

web.page.perf[host,path,port,redirect_limit]

The loading time of a full web page (in seconds).
Return value: Float.

web.page.get gives me to much text:

```
zabbix_agent2 -t web.page.get[http://localhost,/api/v1/,8000]

web.page.get[http://localhost,/api/v1/,8000]  [s|HTTP/1.1 200 OK...
```
On failure we get:
```
zabbix_agent2 -t web.page.get[http://localhost,/api/v1/,8000]

web.page.get[http://localhost,/api/v1/,8000]  [m|ZBX_NOTSUPPORTED] 
```

web.page.perf is better here:
```
zabbix_agent2 -t web.page.perf[http://localhost,/api/v1/,8000]
web.page.perf[http://localhost,/api/v1/,8000] [s|0.002027]
```

On failure we get:
```
zabbix_agent2 -t web.page.perf[http://localhost,/api/v1/,8000]
web.page.perf[http://localhost,/api/v1/,8000] [m|ZBX_NOTSUPPORTED] [Cannot get content of web page: Get "http://localhost:8000/api/v1/": dial tcp :0->127.0.0.1:8000: connect: connection refused]
```

We can test this and trigger:
```
- uuid: 
  name: 'Ansible ARA-Server API Check'
  type: ZABBIX_ACTIVE
  key: 'web.page.get[http://localhost,/api/v1/,8000]'
  history: '0'
  value_type: TEXT
  description: 'Check if the API is available'
  tags:
    - tag: component
      value: ara-api
  triggers:
    - uuid: 
      expression: 'nodata(/Template foobar/web.page.get[http://localhost,/api/v1/,8000],15m)=0'
      name: 'ARA-Server API not available'
      priority: INFO
      manual_close: 'YES'
```

Additionally I will insert a trigger on wrong / no item data on the playbook run status in section 2.




 
2. Playbook <name/path> run assurance.

First collecting the last found playbook in ARA (using path as parameter):
```  
path=$playbook_path -> parameter configured in item
order=-started -> Last started on top
limit=1 -> only the last started

curl -s "http://127.0.0.1:8000/api/v1/playbooks?path=$playbook_path&order=-started&limit=1"  
```

I don't want to expose the port externally, so I am stuck with agent type items instead of API calls from the proxy/server with HTTP agent.
Retrieving the JSON can be done with a userparameter for curl, or using web.page.get. THough while using web.page.get it is important to parse the http output correctly as it includes headers.

Zabbix has this documented:
https://www.zabbix.com/documentation/guidelines/en/thosts/best_practices#getting-data-with-http-agent-and-web.page.get

### TODO USerparameter verhaal verwijderen en item config vervsangen:

Creating the user parameter:
```
UserParameter=ansible.ara.last[*],curl -s "http://127.0.0.1:8000/api/v1/playbooks?path=$1&order=-started&limit=1"
```
Testing:
```
zabbix_agent2 -t ansible.ara.last['/path/to/play']
```

Zabbix item configuration:
```
- uuid:
  name: 'Playbook-description - Get last run raw json'
  type: ZABBIX_ACTIVE
  key: 'ansible.ara.last[playbook-path.yml]'
  delay: 10m
  history: '0'
  value_type: TEXT
  description: |
    Mandatory userparameter: 
    UserParameter=ansible.ara.last[*],curl -s "http://127.0.0.1:8000/api/v1/playbooks?path=$1&order=-started&limit=1"
  tags:
    - tag: component
      value: raw
```

Now I have a nice piece of JSON to play with in Zabbix...

We can find all data in the .results[0] object. And create dependant items for:

- started
- ended
- duration
- status
- whatever

```  
- uuid:
  name: 'Playbook-description - Get last run: Status'
  type: DEPENDENT
  key: 'ansible.ara.last.status[playbook-path.yml]'
  history: 1d
  value_type: CHAR
  preprocessing:
    - type: JSONPATH
      parameters:
        - '$.results[0].status'
  master_item:
    key: 'ansible.ara.last[playbook-path.yml]'
  tags:
    - tag: component
      value: whatever
```

For the started / ended I convert the value to Unix timestamp. Now we can use it in the trigger configuration:
```
- uuid:
  name: 'Playbook-description - Get last run: Started at'
  type: DEPENDENT
  key: 'ansible.ara.last.started[playbook-path.yml]'
  history: 1d
  trends: '0'
  preprocessing:
    - type: JSONPATH
      parameters:
        - '$.results[0].started'
    - type: JAVASCRIPT
      parameters:
        - |
          // Parse ISO 8601 to Unix timestamp
          return Math.floor(Date.parse(value) / 1000);
  master_item:
    key: 'ansible.ara.last[playbook-path.yml]'
  tags:
    - tag: component
      value: foobar
```

Adding assurance to the API failure trigger on section 1. I create a simple trigger on the status item, for no data or wrong data.

Available metrics:
https://ara.readthedocs.io/en/latest/prometheus.html#available-metrics

(completed, failed, running, expired, unknown)

```

```


