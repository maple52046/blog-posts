---
title: Install Rally 筆記 (OpenStack benchmark)
date: 2015-02-24 16:45:00
categories: OpenStack
tags: OpenStack
toc: true
aliases:
  - 2015/02/24/Rally-Installation/
---

[Rally](http://rally.readthedocs.org/en/latest/index.html) 是一套 OpenStack benchmark tool。關於 Rally 的介紹本文就不再贅述。本篇安裝時是以 [OpenStack 官方 Wiki](https://wiki.openstack.org/wiki/Rally#How_To) 的教學為參考內容，紀錄安裝與使用 Rally 的筆記。

## 實驗環境

| Item | Value |
| ---- | ----- |
| Operating System | Ubuntu 12.04 x64 |
| Python version | Python 2.7 |
| OpenStack version | Grizzly (2013) |

## Install Rally

一開始我是想要安裝在 OpenStack controller node 上，但是一直裝不起來，看起來似乎是 python module 的版本問題，不過我沒有仔細研究到底是哪個 module 的哪個版本卡住。

直接在 OpenStack 上開了一個新的 VM，一樣是選擇 Ubuntu 12.04，就可以直接安裝。*(不過現在回想起來，應該要用 virtualenv 來裝就可以解決，也不會影響原本的系統。)*

<!-- more -->

> PS: Rally 可以跟 DevStack 一起部署。參考: [Rally with DevStack all in one installation](https://wiki.openstack.org/wiki/Rally/installation#Rally_with_DevStack_all_in_one_installation)

**Step**:

1. 下載 source code
```
root@rally:~# git clone https://git.openstack.org/stackforge/rally
```

2. 執行 script
```
root@rally:~# ./rally/install_rally.sh
```

3. 安裝並執行 tox (當前版本可以跳過此步驟)
```
root@rally:~# pip install 'tox==1.6.1'
root@rally:~# tox
```

## Deployment initialization

安裝完 Rally 之後，再進行下一步之前，必須要確定該 host 可以使用 OpenStack API

建立 OpenStack credentials (openrc)，然後看看 keystone client 是否可已使用:

建立 RC file:
```bash openrc
export OS_TENANT_NAME=admin
export OS_USERNAE=admin
export OS_PASSWORD=myStackAdmin
export OS_AUTH_URL="http://myopenstack:5000/v2.0/"
export OS_SERVICE_ENDPOINT="http://myopenstack:35357/v2.0"
export OS_SERVICE_TOKEN=myOpenStackAdminToken28143
```

測試 keystone 指令是否能使用:  *(keystone client 已經在 install_rally.sh 的時候就安裝了，不需要額外裝)*
```
root@localhost:~# source openrc
root@localhost:~# keystone tenant-list
... (省略) ...
```

接下來要設定 deployment

### Step 1

首先，複製 samples 裡面的 existing.json，並修改參數

```
root@rally:~# cp rally/samples/deployments/existing.json ./
```

```json mystack.json
{
    "type": "ExistingCloud",
    "auth_url": "http://myopenstack:5000/v2.0/",
    "region_name": "RegionOne",
    "endpoint_type": "public",
    "admin": {
        "username": "admin",
        "password": "myStackAdmin",
        "tenant_name": "benchmark"
    }
}
```

* type 的部分不能換改
* **auth\_url** 這個參數只要使用 public url 的就可以了
* User 一定要是具有 admin role 的 user；tenant 則不一定要特地建立專門使用的，因為測試的過程不會使用到自己的 tenant。

接著執行命令 `rally deployment create --filename=<json file> --name=<name of Openstack deployment>`

```
root@rally:~# rally deployment create --filename existing.json --name myStack
+--------------------------------------+----------------------------+---------+------------------+--------+
| uuid                                 | created_at                 | name    | status           | active |
+--------------------------------------+----------------------------+---------+------------------+--------+
| 3070593f-c583-4bcf-85a8-7dc7345ad146 | 2015-02-24 03:45:37.088393 | myStack | deploy->finished |        |
+--------------------------------------+----------------------------+---------+------------------+--------+
Using deployment: 3070593f-c583-4bcf-85a8-7dc7345ad146
~/.rally/openrc was updated

HINTS:
* To get your cloud resources, run:
	rally show [flavors|images|keypairs|networks|secgroups]

* To use standard OpenStack clients, set up your env by running:
	source ~/.rally/openrc
  OpenStack clients are now configured, e.g run:
	glance image-list
root@rally:~#
```

這時候執行 rally show [option] 可以看到 Openstack 資訊

```
root@rally:~# rally show images
+--------------------------------------+------------------------------+------------+
| UUID                                 | Name                         | Size (B)   |
+--------------------------------------+------------------------------+------------+
| 53c59f00-509a-4056-b5b2-ff1984c9323f | Ubuntu 14.04 (Cloud Image)   | 256836096  |
| 75426fa5-f73b-46a4-84fa-4a44f9cbc9fb | Ubuntu 12.04 (Cloud Image)   | 261423616  |
| 7bd19a5c-0eba-439c-a627-599df9d9ad4d | CentOS 7                     | 418688512  |
+--------------------------------------+------------------------------+------------+
root@rally:~#
```

### Step 2

接下來，執行 `rally deployment check`

```
root@rally:~# rally deployment check
keystone endpoints are valid and following services are available:
+----------+----------+-----------+
| services | type     | status    |
+----------+----------+-----------+
| cinder   | volume   | Available |
| ec2      | ec2      | Available |
| glance   | image    | Available |
| keystone | identity | Available |
| nova     | compute  | Available |
| quantum  | network  | Available |
+----------+----------+-----------+
root@rally:~#
```

## Benchmarking OpenStack

接下來跑一個 benchmark 的範例

### Step 1

從 sample 中複製 benchmark task 範例
```
cp samples/tasks/scenarios/nova/boot-and-delete.json ./task-1.json
```

修改 task 內容

```json task-1.json
{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "m1.tiny"
                },
                "image": {
                    "name": "CirrOS 0.3.3"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                }
            }
        }
    ]
}
```

Flavor name、image name 可以用 rally show [option] 去抓取。
另外，根據 OpenStack Wiki 的[資料](https://wiki.openstack.org/wiki/Rally/Concepts)， flavor 與 image 可以改用提供 id、uuid 的方式。

```json
"args": {
   "flavor_id": 42,
    "image_id": "73257560-c59b-4275-a1ec-ab140e5b9979"
},
```

## Step 2

接下來，執行 `rally -v task start <task json file> --tag <task name>`

最後會出現:
```
root@rally:~# rally -v task start task-1.json --tag "Task 1"

... (過程省略) ...

--------------------------------------------------------------------------------
Task 9148f0ee-d5d6-42a7-a44f-d8cd81b6d309: finished
--------------------------------------------------------------------------------

test scenario NovaServers.boot_and_delete_server
args position 0
args values:
{u'args': {u'flavor': {u'name': u'm1.tiny'},
           u'force_delete': False,
           u'image': {u'name': u'CirrOS 0.3.3'}},
 u'context': {u'users': {u'project_domain': u'default',
                         u'resource_management_workers': 30,
                         u'tenants': 3,
                         u'user_domain': u'default',
                         u'users_per_tenant': 2}},
 u'runner': {u'concurrency': 2, u'times': 10, u'type': u'constant'}}
+--------------------+-----------+-----------+-----------+---------------+---------------+---------+-------+
| action             | min (sec) | avg (sec) | max (sec) | 90 percentile | 95 percentile | success | count |
+--------------------+-----------+-----------+-----------+---------------+---------------+---------+-------+
| nova.boot_server   | 6.708     | 13.289    | 37.405    | 34.299        | 35.852        | 100.0%  | 10    |
| nova.delete_server | 2.347     | 3.872     | 5.509     | 4.932         | 5.22          | 100.0%  | 10    |
| total              | 9.408     | 17.16     | 39.752    | 38.566        | 39.159        | 100.0%  | 10    |
+--------------------+-----------+-----------+-----------+---------------+---------------+---------+-------+
Load duration: 85.8980619907
Full duration: 97.1042778492

HINTS:
* To plot HTML graphics with this data, run:
	rally task report 9148f0ee-d5d6-42a7-a44f-d8cd81b6d309 --out output.html

* To get raw JSON output of task results, run:
	rally task results 9148f0ee-d5d6-42a7-a44f-d8cd81b6d309

Using task: 9148f0ee-d5d6-42a7-a44f-d8cd81b6d309
root@rally:~#
```

跑完之後就會出現如上方範例的簡易報告，如果想要詳盡的報告內容，可以按照提示執行 `rally task report <uuid> --out <html file>` 就可以產生 html 檔案。

Benchmark 參數說明請參考: https://wiki.openstack.org/wiki/Rally/Concepts

另外，Rally 官方也提供了安裝在 docker 的方法，以及如果 Rally 是安裝在 OpenStack controller 時，也有幾個比較簡易的步驟可以參考: http://rally.readthedocs.org/en/latest/tutorial.html

----

# Troubleshooting

## Schema validation error

執行命令 `rally deployment create --filename=<json file> --name=<name of Openstack deployment>` 時發生以下錯誤:

```
root@rally:~# rally deployment create --filename=mystack.json --name=myStack
2015-02-24 03:11:33.579 23212 ERROR rally.api [-] Deployment 59dc5f2c-a83a-42d8-bf49-777ff5e32783: Schema validation error.
Config schema validation error: {'admin': {'username': 'admin', 'password': 'myStackAdmin', 'tenant': 'benchmark'}, 'auth_url': 'http://myopenstack:5000/v2.0/', 'type': 'ExistingCloud'} is not valid under any of the given schemas

Failed validating 'anyOf' in schema:
    {'anyOf': [{'properties': {'admin': {'$ref': '#/definitions/user'}},
                'required': ['type', 'auth_url', 'admin']},
               {'required': ['type', 'auth_url', 'users'],
                'users': {'items': {'$ref': '#/definitions/user'},
                          'type': 'array'}}],
     'definitions': {'user': {'oneOf': [{'properties': {'tenant_name': {'type': 'string'}},
                                         'required': ['username',
                                                      'password',
                                                      'tenant_name']},
                                        {'properties': {'project_domain_name': {'type': 'string'},
                                                        'project_name': {'type': 'string'},
                                                        'user_domain_name': {'type': 'string'}},
                                         'required': ['username',
                                                      'password',
                                                      'project_name']}],
                              'properties': {'password': {'type': 'string'},
                                             'username': {'type': 'string'}},
                              'type': 'object'}},
     'properties': {'auth_url': {'type': 'string'},
                    'endpoint_type': {'enum': ['admin',
                                               'internal',
                                               'public'],
                                      'type': 'string'},
                    'region_name': {'type': 'string'},
                    'type': {'type': 'string'}},
     'type': 'object'}

On instance:
    {'admin': {'password': 'myStackAdmin',
               'tenant': 'benchmark',
               'username': 'admin'},
     'auth_url': 'http://myopenstack:5000/v2.0/',
     'type': 'ExistingCloud'}.
root@rally:~#
```

這時候執行 `rally deployment list` 可以看到 deployment 的狀態是 failed

```
root@rally:~# rally deployment list
+--------------------------------------+----------------------------+---------+----------------+--------+
| uuid                                 | created_at                 | name    | status         | active |
+--------------------------------------+----------------------------+---------+----------------+--------+
| 59dc5f2c-a83a-42d8-bf49-777ff5e32783 | 2015-02-24 03:11:33.195680 | myStack | deploy->failed |        |
+--------------------------------------+----------------------------+---------+----------------+--------+
root@rally:~#
```

雖然這時候，執行 `rally deployment recreate --deployment <uuid of deployment>` 就會**看似**恢復正常。

```
root@rally:~# rally deployment recreate --deployment 59dc5f2c-a83a-42d8-bf49-777ff5e32783
root@rally:~# rally deployment list
+--------------------------------------+----------------------------+---------+------------------+--------+
| uuid                                 | created_at                 | name    | status           | active |
+--------------------------------------+----------------------------+---------+------------------+--------+
| 59dc5f2c-a83a-42d8-bf49-777ff5e32783 | 2015-02-24 03:11:33.195680 | myStack | deploy->finished |        |
+--------------------------------------+----------------------------+---------+------------------+--------+
root@rally:~# rally use deployment --deployment 59dc5f2c-a83a-42d8-bf49-777ff5e32783
Using deployment: 59dc5f2c-a83a-42d8-bf49-777ff5e32783
~/.rally/openrc was updated

HINTS:
* To get your cloud resources, run:
	rally show [flavors|images|keypairs|networks|secgroups]

* To use standard OpenStack clients, set up your env by running:
	source ~/.rally/openrc
  OpenStack clients are now configured, e.g run:
	glance image-list
root@rally:~#
```

但是要做其他動作就會發生錯誤，例如:

```
root@rally:~# rally deployment check
Authentication Issues: user rally doesn't have 'admin' role.
```

**這是因為[OpenStack Wiki 教學文](https://wiki.openstack.org/wiki/Rally/HowTo#Step_1._Deployment_initialization_.28use_existing_cloud.29)裡，deployment.json 少了一些參數。如果是直接從 source code 裡面的 sample 中複製並修改，就不會發生這個問題。**

----
## ConnectionRefused: Unable to establish connection OpenStack API endpoint

```
root@rally:~# rally deployment check
keystone endpoints are valid and following services are available:
Command failed, please check log for more info
2015-02-24 03:48:00.930 24945 CRITICAL rally [-] ConnectionRefused: Unable to establish connection to http://quanta-42.sslab.cs.nthu.edu.tw:35357/v2.0/OS-KSADM/services
2015-02-24 03:48:00.930 24945 TRACE rally Traceback (most recent call last):
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/bin/rally", line 10, in <module>
2015-02-24 03:48:00.930 24945 TRACE rally     sys.exit(main())
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/rally/cmd/main.py", line 42, in main
2015-02-24 03:48:00.930 24945 TRACE rally     return cliutils.run(sys.argv, categories)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/rally/cmd/cliutils.py", line 318, in run
2015-02-24 03:48:00.930 24945 TRACE rally     ret = fn(*fn_args, **fn_kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "<string>", line 2, in check
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/rally/cmd/envutils.py", line 66, in default_from_global
2015-02-24 03:48:00.930 24945 TRACE rally     return f(*args, **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/rally/cmd/commands/deployment.py", line 248, in check
2015-02-24 03:48:00.930 24945 TRACE rally     for service in client.services.list():
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/v2_0/services.py", line 32, in list
2015-02-24 03:48:00.930 24945 TRACE rally     return self._list("/OS-KSADM/services", "OS-KSADM:services")
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/base.py", line 113, in _list
2015-02-24 03:48:00.930 24945 TRACE rally     resp, body = self.client.get(url, **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/adapter.py", line 164, in get
2015-02-24 03:48:00.930 24945 TRACE rally     return self.request(url, 'GET', **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/adapter.py", line 200, in request
2015-02-24 03:48:00.930 24945 TRACE rally     resp = super(LegacyJsonAdapter, self).request(*args, **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/adapter.py", line 89, in request
2015-02-24 03:48:00.930 24945 TRACE rally     return self.session.request(url, method, **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/utils.py", line 318, in inner
2015-02-24 03:48:00.930 24945 TRACE rally     return func(*args, **kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/session.py", line 369, in request
2015-02-24 03:48:00.930 24945 TRACE rally     resp = send(**kwargs)
2015-02-24 03:48:00.930 24945 TRACE rally   File "/usr/local/lib/python2.7/dist-packages/keystoneclient/session.py", line 412, in _send_request
2015-02-24 03:48:00.930 24945 TRACE rally     raise exceptions.ConnectionRefused(msg)
2015-02-24 03:48:00.930 24945 TRACE rally ConnectionRefused: Unable to establish connection to http://quanta-42:35357/v2.0/OS-KSADM/services
2015-02-24 03:48:00.930 24945 TRACE rally
root@rally:~#
```

我會發生這個問題是因為，我將 Rally 安裝在 OpenStack 之外，而 `rally deployment check` 會 call internal url (或是 admin url 我不是很確定)而非 public url。不過解決方法很簡單，**只要將 IP 與 hostname 對應設定到 /etc/hosts 上即可。**
