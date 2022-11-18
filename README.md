# blog

<http://blog.axiaoxin.com>

## dev

```
git clone git@github.com:axiaoxin-com/blog.git
cd blog/themes/beautifulhugo
git submodule update --init --recursive
cd ../..
hugo server
```

## deploy

```
hugo # 生成 public
...
git push origin main # push 到 main 分支自动部署到服务器
```
