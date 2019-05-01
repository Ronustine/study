org.apache.commons.net.ftp.FTPClient下的。

```java
FTPClient ftp = new FTPClient();
ftp.connect("ip地址","port端口");//连接
ftp.login("userName","password");//登录
ftp.setFileType(FTPClient.BINARY_FILE_TYPE);//设置文件类型
int reply = ftp.getReplyCode();//某个状态码是连接成功的
FileInputStream input=new FileInputStream(file2); //获取文件流
isUpload = ftp.storeFile(new String(destinationPath.getBytes("GBK"),"iso-8859-1"),input);//上传文件
ftp.logout();//登出
ftp.disconnect();//断开连接
```
