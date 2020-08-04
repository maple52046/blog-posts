---
title: 'OpenStack Glance Image-list 401'
date: 2015-04-08 16:12:00
categories: OpenStack
tags:
  - Glance
  - Juno
  - OpenStack
aliases:
  - 2015/04/08/OpenStack-Glance-image-list-401/
---

架設 OpenStack Juno ，安裝完 Glance 後進行測試時，glance 指令會一直出現 401 錯誤。

```
root@localhost:~# glance --debug image-list
curl -i -X GET -H 'User-Agent: python-glanceclient' -H 'Content-Type: application/octet-stream' -H 'Accept-Encoding: gzip, deflate, compress' -H 'Accept: */*' -H 'X-Auth-Token: ***' http://140.114.91.220:9292/v1/images/detail?sort_key=name&sort_dir=asc&limit=20
Request returned failure status 401.
Invalid OpenStack Identity credentials.
```

<!-- more -->

查看 `/var/log/glance/api.log` 看到以下的錯誤訊息:

```
2015-04-08 15:34:03.127 18799 ERROR keystonemiddleware.auth_token [-] Bad response code while validating token: 403
2015-04-08 15:34:03.128 18799 WARNING keystonemiddleware.auth_token [-] Invalid user token. Keystone response: {u'error': {u'message': u'You are not authorized to perform the requested action: identity:validate_token', u'code': 403, u'title': u'Forbidden'}}
```

但是 keystone 指令本身沒問題:

```
root@localhost:~# keystone --debug token-get
DEBUG:keystoneclient.auth.identity.v2:Making authentication request to http://localhost:35357/v2.0/tokens
INFO:urllib3.connectionpool:Starting new HTTP connection (1): twin-26
DEBUG:urllib3.connectionpool:Setting read timeout to 600.0
DEBUG:urllib3.connectionpool:"POST /v2.0/tokens HTTP/1.1" 200 1084
+-----------+----------------------------------+
|  Property |              Value               |
+-----------+----------------------------------+
|  expires  |       2015-04-08T08:31:37Z       |
|     id    | 6de04b01727841eb90ccaff60f9b0ac7 |
| tenant_id | ed1743be7bac44fca510892f856d5662 |
|  user_id  | 6c0899095ae74fc298dff6b335361ed6 |
+-----------+----------------------------------+
```

Google 到了幾篇參考資料:

- [401 on "glance image-list"](https://ask.openstack.org/en/question/1325/401-on-glance-image-list/)
- [Ask for help with devstack error: "401	Unauthorized"](http://lists.openstack.org/pipermail/openstack-dev/2015-February/056949.html)
- https://review.openstack.org/#/c/154391

第一篇所提到 region 設定，不論是在 `/etc/glance/glance-api.conf` 與 `/etc/glance/glance-registry.conf` 中設定 `auth_region`，還是在環境變數中設定 `OS_REGION_NAME` 都沒有效果。

第二篇參考資料所提到的狀況與我的狀況幾乎一模一樣，透過該篇下面的回應，找到了第三篇參考資料。開頭敘述:

> Most of the services create the service user with the admin permission.
> This is unnecessary for token validation and they should be restricted
> to only having the service role.

## Solution

Glance 的 user role 我本來設定是 `Member`，解決方法就是改成 `admin` role 問題就解決了
