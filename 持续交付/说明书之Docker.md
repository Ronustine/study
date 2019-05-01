Linux 安装Docker某个版本

```bash
yum install docker-1.12.6*
```

卸载Docker

```bash
yum list installed | grep docker
->  xxxxx-xxx.x86_64    17.03.0.ce-1.el7.centos
yum -y remove xxxxxxxx
```

创建镜像

```bash
docker commit -m "镜像说明" -a "我是作者" 882i2jo1dw test:1.0
```

保存镜像

```bash
docker save -o test.tar test:0.1
```

载入镜像

```bash
docker load --input test.tar
```

