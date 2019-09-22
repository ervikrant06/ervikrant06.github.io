---
layout: post
title: How to use stackdriver logging and error reporting in python?
tags: [saltstack]
category: [saltstack]
author: vikrant
comments: true
--- 


I started working with saltstack sometime back it's fair to say that I am on learning curve. I was reading about beacon and reactor concept of saltstack. I thought of running one excercise in lab environment to get hands on experience. 

I started salstack setup [referring](https://medium.com/@timlwhite/the-simplest-way-to-learn-saltstack-cd9f5edbc967)

- Most of the examples on internet are using inotify to monitor any conf file so that if anything change with that file on minion then we can trigger some salt module from master. In docker setup when I list the available beacons I don't see inotify in the list you may installl the packages related to inotify to make them appear in the list but I thought of using different beacon (pkg) for this exercise. 

~~~
[root@11b72e3348d3 /]# salt '*' beacons.list_available
7e6fe8bda8ac:
    beacons:
    - pkg
    - btmp
    - diskusage
    - journald
    - load
    - log
    - memusage
    - network_info
    - proxy_example
    - ps
    - salt_proxy
    - service
    - status
    - wtmp
e7819bb5f657:
    beacons:
    - pkg
    - btmp
    - diskusage
    - journald
    - load
    - log
    - memusage
    - network_info
    - proxy_example
    - ps
    - salt_proxy
    - service
    - status
    - wtmp
~~~

- Create beacon, reactor and state file which you want to trigger from reactor. I want to keep the vim-minimal package on latest version on minions. 

~~~
[root@11b72e3348d3 pillar]# cat /srv/pillar/first_beacon.sls 
beacons:
  pkg: 
    - pkgs:
      - vim-minimal
    - refresh: True

[root@11b72e3348d3 /]# cat /srv/reactor/fix.sls 
deploy_mapp:
  cmd.state.sls:
    - tgt: {{ data.id }}
    - kwarg:
        mods: myapp

[root@11b72e3348d3 /]# cat /srv/salt/myapp.sls 
myapp:
  pkg.latest:
  - name: vim-minimal
[root@11b72e3348d3 /]# 
~~~

- Add the reactor section to your salt master file on master docker. Restart the salt master after making the change. 

~~~
[root@11b72e3348d3 /]# tail -5 /etc/salt/master
reactor:
  - 'salt/beacon/*/pkg/':
    - /srv/reactor/fix.sls
[root@11b72e3348d3 /]# 
~~`

- Refresh the pillar status. 

~~~
[root@11b72e3348d3 /]# salt '*' saltutil.pillar_refresh 
[root@11b72e3348d3 /]# salt '*' pillar.items
[root@11b72e3348d3 pillar]# salt '*' beacons.list          
33b8687b88f6:
    beacons:
      enabled: true
      pkg:
      - pkgs:
        - vim-minimal
      - refresh: true
7e6fe8bda8ac:
    beacons:
      enabled: true
      pkg:
      - pkgs:
        - vim-minimal
      - refresh: true
~~~


- Keep an eye on the events at master node. Once it detects any minon not with the latest version of vim-minimal it will trigger the update of package. 

~~~
[root@11b72e3348d3 pillar]# salt-run state.event

salt/job/20190921073429421679/ret/e7819bb5f657  {
    "_stamp": "2019-09-21T07:34:42.415529",
    "cmd": "_return",
    "fun": "state.sls",
    "fun_args": [
        {
            "mods": "myapp"
        }
    ],
    "id": "e7819bb5f657",
    "jid": "20190921073429421679",
    "out": "highstate",
    "retcode": 0,
    "return": {
        "pkg_|-myapp_|-vim-minimal_|-latest": {
            "__id__": "myapp",
            "__run_num__": 0,
            "__sls__": "myapp",
            "changes": {
                "vim-minimal": {
                    "new": "2:7.4.629-6.el7",
                    "old": "2:7.4.160-6.el7_6"
                }
            },
            "comment": "The following packages were successfully installed/upgraded: vim-minimal",
            "duration": 11183.822,
            "name": "vim-minimal",
            "result": true,
            "start_time": "07:34:31.226182"
        }
    },
    "success": true
}
~~~





