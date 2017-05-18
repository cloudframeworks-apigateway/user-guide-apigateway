# KONG命令行

## 注册API

```
curl -i -X POST \
      --url http://127.0.0.1:8001/apis/ \
      --data 'name=nginxfirst' \
      --data 'hosts=nginxfirst' \
      --data 'upstream_url=http://xx.xx.xx.xx:9001/'
```

通过返回数据查看注册是否成功

## API添加插件

```
curl -i -X POST \
  --url http://127.0.0.1:8001/apis/nginxfirst/plugins/ \
  --data 'name=key-auth'
```

通过命令行进行访问验证：

```
curl -H 'Host: nginxfirst' -H 'TT: e9da671f5c5d44d5bfdca95585283979' http://127.0.0.1:8000
```

<div align=center><img width="600" height="" src="./image/keyauthsucc.png"/></div>

```
curl -H 'Host: nginxfirst' http://127.0.0.1:8000
```

<div align=center><img width="600" height="" src="./image/keyauthfailed.png"/></div>
