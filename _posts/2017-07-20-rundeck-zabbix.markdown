---
title: "Runbook Automation with Zabbix and Rundeck"
layout: post
date: 2017-07-20 10:29
image: /assets/images/zabbix-rundeck.jpg
headerImage: false
tag:
- rundeck
- zabbix
- automation
category: blog
author: kosyanyanwu
description: Integrate Zabbix trigger based actions into Rundeck
---

## Introduction

[Zabbix](https://www.zabbix.com/) is an open source enterprise-level software designed for real-time monitoring of millions of metrics.

[Rundeck](http://rundeck.org/) is a job scheduler and runbook automation tool.

This article explains how you can integrate Zabbix trigger based actions into Rundeck. The code snippets and examples are written in Python3, using Zabbix API version 3.2 and Rundeck API version 20.

## Real world example
* A service stops running
* Zabbix fires trigger and sends notifications
* Ops receive notification
* Ops goes to restart service
* Hurray!!! service is running again
* Probably another one stops running in another server
* And the cycle continues ...

## Disadvantages
* Process is repetitive, manual and sometimes annoying :(
* No easily accessible preserved output from executed remote actions

## Zabbix meets Rundeck (Real world example)
* A service stops running
* Zabbix fires trigger
* Zabbix action calls middleware
* Middleware executes job on Rundeck
* Middleware sends acknowledgement to Zabbix
* Zabbix receives acknowledgement
* Ops continues partying, nothing left for them to do

## Integration requirements
* Map Zabbix hosts to Rundeck nodes
* Map Zabbix trigger names to Rundeck job names
* On trigger event, pass related host and trigger from Zabbix to Rundeck
* Return job execution status from Rundeck as an event acknowledgement in Zabbix

Here is the [big picture]( https://www.zabbix.com/files/zabconf2016/wolfgang_alper_IntelliTrend-Zabbix-Meets-Ops-With-Rundeck.pdf) of what we intend to achieve.
![Screenshot](https://user-images.githubusercontent.com/10073270/28416679-eb719696-6d4c-11e7-8228-e27edf407986.png)

{% highlight raw %}
Now let us dive into the real thing ... Get your text editor ready.
{% endhighlight %}

## Get Zabbix API key
Before you can access any data inside of Zabbix you'll need to log in and obtain an [authentication token](https://www.zabbix.com/documentation/3.2/manual/api). This can be done using the `user.login` method.

{% highlight python %}
    payload = {
        "jsonrpc": "2.0",
        "method": "user.login",
        "params": {
            "user": YOUR_ZABBIX_USER_NAME,
            "password": YOUR_ZABBIX_PASSWORD
        },
        "id": 1,
        "auth": None
    }

    url = 'https://ZABBIX_URL/api_jsonrpc.php'
    headers = {
        'content-type': 'application/json'
    }

    auth_token = requests.get(url,
                        data=json.dumps(payload),
                        headers=headers
                        ).json().get('result')
    )
{% endhighlight %}

## Get Rundeck API key
This can be gotten from `RUNDECK_URL:PORT_NUMBER/user/profile`

## Map Zabbix hosts to Rundeck nodes
Rundeck uses a resource document to declare the resource models used by a project to define the set of Nodes that are available. This file is usually found in `/etc/rundeck/projects/project_name/` in the Rundeck server. We will be using the yaml resource format in this case.

Create a file called resource.yml on your local machine. We will dump our mapping of Zabbix hosts to the file and eventually copy it over to Rundeck.

Make an API call to Zabbix to get the host details. Here is an example code snippet.
{% highlight python %}
    url = 'ZABBIX_URL/api_jsonrpc.php'
    headers = {
        'content-type': 'application/json'
    }
    method = 'host.get'
        params = {
            'output': [
                'host',
                'name',
                'description'
            ],
            'selectTriggers': [
                'description',
                'status'
            ]

        }
    payload = {
        'method': method,
        'params': params,
        'jsonrpc': '2.0',
        'id': 1,
        'auth': ZABBIX_API_KEY
    }

    host_details = requests.get(url,
                            data=json.dumps(payload),
                            headers=headers
                            ).json().get('result')
{% endhighlight %}

Here is an example node object definition.
{% highlight python %}
    node = {
            host: {
                'nodename': NAME_OF_THE_NODE,
                'hostname': NAME_OF_THE_HOST,
                'description': description,
                'username': SSH_USERNAME,
                'ssh-keypath': SSH_KEY_PATH,
                'tags': ''
            }
        }
{% endhighlight %}
You can read more about node entries [here](http://rundeck.org/docs/man5/resource-yaml.html) and format yours appropriately.

Next steps would be to loop through the `host_details` value gotten from Zabbix API, fill the node object with the appropriate values and dump the node object to the `resources.yml` file using `yaml.dump()`. With this, we have been able to successfully map our Zabbix hosts to Rundeck nodes :)

Do not forget to add Rundeck's local host to resources.yml file. copy the file over to `/etc/rundeck/projects/project_name/resources.yml` on Rundeck's server.

## Map Zabbix trigger names to Rundeck job names
Create a file called jobs.yml on your local machine. We will dump our mapping of Zabbix triggers to the file and eventually load it to Rundeck.

Now here is the tricky part, how do we know what triggers to look out for? Should Rundeck do stuff for some triggers or all the triggers? I would suggest having a tag name in front of triggers that you want Rundeck to take action for. We can prepend something like `[RD]` to the trigger names on Zabbix and pull just those triggers beginning with the prefix we specified.

From the API call made earlier to get `host_details`, we also have trigger details returned alongside.
Here is an example of a job list definition.
{% highlight python %}
    job = [{
            'sequence': {
                'strategy': 'node-first',
                'keepgoing': False,
                'pluginConfig': {
                    'WorkflowStrategy': {
                        'node-first': None
                    }
                },
                'commands': [{'exec': ''}]
            },
            'nodesSelectedByDefault': True,
            'executionEnabled': True,
            'nodeFilterEditable': False,
            'scheduleEnabled': True,
            'name': TRIGGER_NAME,
            'description': TRIGGER_DESCRIPTION,
            'nodefilters': {
                'filter': HOST_NAME
            }
        }]
{% endhighlight %}
You can read more about job entries [here](http://rundeck.org/docs/man5/job-yaml.html) and format yours appropriately.

Next steps would be to loop through the `trigger_details` value gotten from Zabbix API, fill the job list with the appropriate values and dump the job list to the `jobs.yml` file using `yaml.dump()`. Update the `commands` section with the appropriate command you want Rundeck to execute. With this, we have been able to successfully map our Zabbix triggers to Rundeck jobs :) Now you can [load the jobs file to Rundeck](http://rundeck.org/docs/man5/job-yaml.html#loading-and-unloading).

## On trigger event, pass related host and trigger from Zabbix to Rundeck
How can we magically pass trigger event from Zabbix to Rundeck? This can be made possible by having a middleware script placed in Zabbix server. This middleware script will be executed by Zabbix as specified in `Zabbix actions` section with these arguments passed into the file --> `{HOST.NAME} {EVENT.ID} {TRIGGER.NAME}`.
For instance, let us assume the middleware script is called `middleware.py`. The remote command from Zabbix action will call the script as `python /path/to/middleware.py {HOST.NAME} {EVENT.ID} {TRIGGER.NAME}`.

When this middleware script is executed from Zabbix with the specified arguments passed in, the script gets the host name, event id and trigger name from it.

Recall that we mapped trigger names to job names on Rundeck earlier. Now, this middleware will make an API call to Rundeck to get the job it would execute from the trigger name passed in. Here is an example code snippet.
{% highlight python %}
    rundeck_url = 'http://RUNDECK_URL:PORT_NUMBER'
    payload = {
        'authtoken': RUNDECK_API_KEY,
        'jobExactFilter': trigger_name
    }
    headers = {
        'Accept': 'application/json'
    }
    job = requests.get(
        url, params=payload, headers=headers
    )
{% endhighlight %}
You can read more about Rundeck API [here](http://rundeck.org/docs/api/).

With the job info we got from the above code, all we need to do is make another call to Rundeck API to execute the job.

## Return job execution status from Rundeck as an event acknowledgement in Zabbix
After Rundeck successfully executes the job, using the return value, make an API call to Zabbix to return an acknowledgement. Here is an example code snippet.
{% highlight python %}
    url = 'ZABBIX_URL/api_jsonrpc.php'
    headers = {
        'content-type': 'application/json'
    }
    event_id = EVENT_ID_FROM_SCRIPT_ARGS
    method = 'event.acknowledge'
    job = JOB_INFO_FROM_PREVIOUS_API_CALL
    message = ('Rundeck-Execution id: {id}, status: {status}, '
               'job: {job}, project: {project}, info: {info}'
               ).format(
                   id=job_exec_info['id'],
                   status=status,
                   job=job['name'],
                   project=job['project']
    )
    method = 'host.get'
    params = {
        'eventids': event_id,
        'message': message,
        'action': 1
    }
    payload = {
        'method': method,
        'params': params,
        'jsonrpc': '2.0',
        'id': 1,
        'auth': ZABBIX_API_KEY
    }

    requests.post(url,
                  data=json.dumps(payload),
                  headers=headers
                 )
{% endhighlight %}

Phew! This article was too long right? When I had to implement this, the only information I found was [this pdf](https://www.zabbix.com/files/zabconf2016/wolfgang_alper_IntelliTrend-Zabbix-Meets-Ops-With-Rundeck.pdf). But yea, that was how succesfully integrate Zabbix trigger based actions into Rundeck.
