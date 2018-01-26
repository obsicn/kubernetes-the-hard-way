创建云服务，手工制定ip，确保机器名和ip一致

	
IO优化型I2
	运行中
	
可用区A
	
内
10.0.10.6

选择缺省镜像，确保都安装了docker

采用默认ip，不用修改/etc/hosts文件

缺省镜像安装了kubelet（利用kubeadm安装方法），需要清除该服务。

ubuntu@vm10-0-10-200:~$ docker version
Client:
 Version:      1.13.1
 API version:  1.26
 Go version:   go1.6.2
 Git commit:   092cba3
 Built:        Thu Nov  2 20:40:23 2017
 OS/Arch:      linux/amd64

Server:
 Version:      1.13.1
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.6.2
 Git commit:   092cba3
 Built:        Thu Nov  2 20:40:23 2017
 OS/Arch:      linux/amd64
 Experimental: false
