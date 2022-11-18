---
title: "Flask 通过 Nginx 代理后拿不到正确的 url_scheme"
author: "axiaoxin"
date: 2016-04-14T12:06:49+08:00
subtitle: ""
image: ""
tags: ["flask", "nginx"]
slug: ""
---

问题：Nginx 配置了 https 请求后，Flask 中的 request 中 `wsgi.url_scheme` 收到的仍然是 http，而不是 https。

当用户发起 https 请求时首先和 Nginx 建立连接，完成 SSL 握手，而后 Nginx 作为代理是以 http 协议将请求转给 gunicorn 处理的，
Nginx 再把 gunicorn 的输出通过 SSL 加密发回给用户，这中间是透明的，gunicorn 只是在处理 http 请求而已。

所以，即使请求时用的是 https，但是在 Flask中收到的仍然是 http，导致在其他 url 相关的地方的值都是 http 链接。

解决办法是在 Flask 中使用 **ProxyFix** ，并且确保 Nginx 配置中设置了 `Host` 和 `X-Forwarded-Proto`

Flask 使用 ProxyFix 示例：

```python
from flask import Flask
from werkzeug.contrib.fixers import ProxyFix

app = Flask(__name__)
app.wsgi_app = ProxyFix(app.wsgi_app)
```

Nginx 配置：

```nginx
location / {
    proxy_pass http://your_upstream/;
    proxy_set_header   Host             $host;
    proxy_set_header   X-Forwarded-Proto  $scheme;
}
```

参考文档:

- <https://groups.google.com/forum/#!topic/pocoo-libs/KAle_rNC1V8>
- <http://docs.jinkan.org/docs/flask/deploying/wsgi-standalone.html#deploying-proxy-setups>
- <http://werkzeug.pocoo.org/docs/contrib/fixers/#werkzeug.contrib.fixers.ProxyFix>
- <https://github.com/mitsuhiko/werkzeug/blob/master/werkzeug/contrib/fixers.py#L81>
