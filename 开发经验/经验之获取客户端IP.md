> 须知：服务器只能从header头获取ip信息。但是匿名代理可以通过设置header头，完全隐藏客户端的IP，让服务器无法识别。
> REMOTE_ADDR HTTP_VIA HTTP_X_FORWARDED_FOR

## Header相关
REMOTE_ADDR: 直接调用服务的IP，即就算经过了很多代理，取的是最后一个
HTTP_VIA: 代理会把自己往这里加
HTTP_X_FORWARDED_FOR: 经过了多个代理，会记录（代理会记录上一个的ip，代理不记录就没有）

#### Java
REMOTE_ADDR: request.getRemoteAddr()
HTTP_VIA：不知道
HTTP_X_FORWARDED_FOR: request.getHeader("X-Forwarded-For");

## 未使用代理
> 即客户端直接调服务。在微服务中，其他服务可以作为客户端，直接调用二方服务

REMOTE_ADDR = 客户端 IP
HTTP_VIA = 没数值或不显示
HTTP_X_FORWARDED_FOR = 无数值或不显示

## 使用透明代理服务器（Transparent Proxies）：
REMOTE_ADD: 最后一个代理服务器 IP
HTTP_VIA: 代理服务器 IP
HTTP_X_FORWARDED_FOR: 客户端IP, 代理1 IP, 代理n IP

## 使用普通匿名代理服务器（Anonymous Proxies）：
REMOTE_ADDR = 最后一个代理服务器 IP
HTTP_VIA = 代理服务器 IP
HTTP_X_FORWARDED_FOR = 代理1 IP, ..., 代理n IP

此类代理服务器隐藏了客户端的真实IP，但是向访问对象能看出是使用代理服务器访问服务的。

## 使用欺骗性代理服务器（Distorting Proxies）：
REMOTE_ADDR = 代理服务器 IP
HTTP_VIA = 代理服务器 IP
HTTP_X_FORWARDED_FOR = 随机的 IP ，代理1 IP, ..., 代理n IP

访问对象能看出是使用代理服务器的，但能否验出是随机IP，看访问对象能力了

## 使用高匿名代理服务器（High Anonymity Proxies (Elite proxies)）：
REMOTE_ADDR = 代理服务器 IP
HTTP_VIA = 没数值或不显示
HTTP_X_FORWARDED_FOR = 没数值或不显示 ，经过多个代理服务器时，这个值类似如下：代理1 IP, ..., 代理n IP

此类代理服务器完全用代理服务器的信息替代了客户端的所有信息，就象是完全使用那台代理服务器直接访问对象一样。

## 关于Nginx
现在大多数的服务都使用代理服务器(如Nginx，代理服务器可以理解为用户和服务器之间的中介，双方都可信任。)，而用户对代理服务器发起的HTTP请求，代理服务器对服务集群中的真实部署的对应服务进行“二次请求”。
nginx配置这段：`location ~ ^/static {proxy_pass ....;proxy_set_header X-Forward-For $remote_addr ;}`会将请求自己的客户端IP往`X-Forward-For`写，自己占掉`remote_addr`。
所以在使用了反向代理的情况下，request.getRemoteAddr()获取的是反响代理在内网中的ip地址。

## 一些代理在header里常用的名字
> 看公司架构，代码按需用就好，没必要全用。如果要反爬虫，需要全部考虑了

- X-Real-IP：nginx的，可以获取真实ip
- Proxy-Client-IP：apache的
- WL-Proxy-Client-IP: weblogic的
- HTTP_X_FORWARDED_FOR：？
- HTTP_CLIENT_IP：？
- 等