---
title: "使用 Nginx Lua 修改 Influxdb API 的返回结构"
author: "axiaoxin"
date: 2017-12-13T15:29:55+08:00
subtitle: ""
image: ""
tags: ["nginx", "lua"]
slug: ""
---

有一个API平台服务，所有接口都通过API平台转发到实际的服务上，然后再把实际服务的结果返回给客户端，API平台的规范是所有实际服务的接口返回都要统一结构为

```json
{
    "code": 0,
    "msg": "",
    "data": {}
}
```

否则无法处理，现在使用influxdb提供的api，他的返回结构并不是API平台需要的结构，所以需要做一层代理转发请求并修改返回结果为API平台需要的结构。

采用openresty来实现这个需求，只需安装好openresty，然后编写一个配置文件即可实现。

实现配置如下：**ngx_lua_restructure_influxdb_response.conf**

```nginx
upstream influxdb{
    server 127.0.0.1:8086;
}

server {
    listen 18086;
    proxy_pass_request_headers off;  # handle gziped capture <https://github.com/openresty/lua-nginx-module/issues/12>

    location / {
        proxy_pass http://influxdb/query;
    }


    location /query {
        # 由于外层的nginx配置默认的`content-type`为`application/octet-stream;`，所以为了确保返回为json，又在`/query`中明确的设置了content-type
        add_header Content-Type 'application/json; charset=utf-8';  # make sure content-type is json
        content_by_lua_block {
            local method_map = {
                GET = ngx.HTTP_GET,
                POST = ngx.HTTP_POST,
            }
            ngx.req.read_body()
            local method = method_map[ngx.req.get_method()]
            local data = ngx.req.get_body_data()
            local args = ngx.req.get_uri_args()
            local res = ngx.location.capture("/", {method=method, data=data, args=args})
            local cjson = require "cjson"
            local new_body = {}
            if res.status ~= ngx.HTTP_OK then
                new_body["code"] = res.status
            else
                new_body["code"] = 0
            end
            new_body["msg"] = ''
            new_body["data"] = cjson.decode(res.body)
            ngx.say(cjson.encode(new_body))
        }
    }
}
```

influxdb的api在8086端口，所有直接的请求返回结果为：

```sh
# curl -Gs '100.113.164.108:8086/query?db=data&q=SELECT+*+FROM+cpu+limit+1&pretty=true'
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "cpu",
                    "columns": [
                        "time",
                        "cpu",
                        "host",
                        "usage_guest",
                        "usage_guest_nice",
                        "usage_idle",
                        "usage_iowait",
                        "usage_irq",
                        "usage_nice",
                        "usage_softirq",
                        "usage_steal",
                        "usage_system",
                        "usage_user"
                    ],
                    "values": [
                        [
                            "2017-12-08T08:30:00Z",
                            "cpu-total",
                            "Tencent-SNG",
                            0,
                            0,
                            96.92993315559063,
                            0.4456548651008583,
                            0,
                            0,
                            0,
                            0,
                            1.8321366676828983,
                            0.7922753157508718
                        ]
                    ]
                }
            ]
        }
    ]
}
```

经过18086端口的转发后得到的结果为：

```sh
# curl -Gs '100.113.164.108:18086/query?db=db_zhiyun&q=SELECT * FROM cpu limit 1&pretty=true' | python -m json.tool
{
    "code": 0,
    "data": {
        "results": [
            {
                "series": [
                    {
                        "columns": [
                            "time",
                            "cpu",
                            "host",
                            "usage_guest",
                            "usage_guest_nice",
                            "usage_idle",
                            "usage_iowait",
                            "usage_irq",
                            "usage_nice",
                            "usage_softirq",
                            "usage_steal",
                            "usage_system",
                            "usage_user"
                        ],
                        "name": "cpu",
                        "values": [
                            [
                                "2017-12-08T08:30:00Z",
                                "cpu-total",
                                "Tencent-SNG",
                                0,
                                0,
                                96.929933155591,
                                0.44565486510086,
                                0,
                                0,
                                0,
                                0,
                                1.8321366676829,
                                0.79227531575087
                            ]
                        ]
                    }
                ],
                "statement_id": 0
            }
        ]
    },
    "msg": ""
}
```

实现中遇到的坑：

使用curl请求可以得到正确结果，但是通过API平台过来的请求全部报错，错误信息为在capture后拿到的body是一传乱码，并不是json。

使用`ngx.req.raw_header()`查看curl和API平台请求的header信息的区别，发现curl没有设置`Accept-Encoding`，
而API平台的请求都带有`Accept-Encoding: gzip, deflate`，导致capture子请求（即请求真实influxdb接口）的返回结果被gzip压缩后把json变成了乱码，
从而导致在后面cjson.decode的时候出错。

解决方式就是要让nginx对influxdb的接口不做gzip压缩，比较优雅的方式可以参考：<https://github.com/openresty/lua-nginx-module/issues/12>

方法-1: Placed in "location /cap2":

```nginx
more_clear_input_headers Accept-Encoding;
```


方法-2：Placed in "location /cap2":

```nginx
proxy_pass_request_headers off;
```

方法-3：Downloaded https://github.com/brimworks/lua-zlib :

```sh
make linux
cp zlib.so /usr/lib/lua/5.1/zlib.so
```

added following to conf:

```lua
local zlib = require("zlib")
local stream = zlib.inflate()
local inflated_body = stream(res1.body)
local res2 = ngx.location.capture("/cap2?"..inflated_body)
```

