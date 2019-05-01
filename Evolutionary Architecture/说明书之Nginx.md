CentOS yum源没有nginx源需要第三方yum源

```bash
# 获取yum仓库源
sudo rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
# 安装nginx
sudo yum install -y nginx
```

可使用epel源

```bash
# 安装epel-release源并安装
yum install epel-release
yum update
yum install nginx
```