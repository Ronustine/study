搭建：
采用docker，没啥注意的

获取版本号
cat /opt/gitlab/version-manifest.json | grep build_version

gitlab更改仓库指向后需要导入
gitlab-rake gitlab:import:repos