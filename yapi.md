#### yapi

接口管理平台

#### 安装

安装node环境

参考官方的安装地址

https://github.com/YMFE/yapi

```shell
npm install -g yapi-cli --registry https://registry.npm.taobao.org
yapi server 
```

部署成功，请切换到部署目录，输入： "node vendors/server/app.js" 指令启动服务器, 然后在浏览器打开 http://127.0.0.1:9090 访问

#### 问题

1. Error: Cannot find module 'json-schema-faker' YAPI部署

 这个错误的原因在于 nodejs 的 运行权限和运行 yapi -server 的权限不一致。

​      解决：

```shell
chown -R root:root /nodejs安装目录(环境变量配置的目录)
```



####  后台启动

```shell
　　nohup node app.js > app.log &
```