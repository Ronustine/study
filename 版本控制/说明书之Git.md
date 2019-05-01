Linux(ubuntu)下使用：
1、安装
git，看系统是否安装了git。
如果没有，sudo apt-get install git。比较老的可能需要使用sudo apt-get install git-core，原因是以前也有一个软件叫GIT，后来才让步
安装完毕后，请用下面两个命令git config --global user.email "XXX@XXX.XXX"  和 git config --global user.name "XXX" 指明你是谁

2、创建公钥和私钥
ssh-keygen -t rsa -C "XXX@XXX.XXX" ->  确认公私钥存放的目录 -> 输入passphrase
cat .ssh/id_rsa.pub ->将公钥放在开源中国-码云下
ssh -T git@git.oschina.net -> 返回"welcom to Git@OSC, yourname!"，表示成功

3、创建版本库
新建一个目录，初始化Git仓库：git init 
将一个文件

4、 要查看删除的文件

git ls-files --deleted

使用命令checkout来恢复

git checkout -- file_name

     如果要恢复多个被删除的文件，可以使用批处理命令

git ls-files -d | xargs git checkout --
 
     如果要恢复被修改的文件，命令

git ls-files -m | xargs git checkout --

5、初始化仓库，以multipay为例

mkdir multipay
cd multipay
git init
touch README.md
git add README.md
git commit -m "first commit"
git remote add origin https://git.oschina.net/founder123/multipay.git
git push -u origin master

git remote add origin https://git.oschina.net/founder123/multipay.git
git push -u origin master
