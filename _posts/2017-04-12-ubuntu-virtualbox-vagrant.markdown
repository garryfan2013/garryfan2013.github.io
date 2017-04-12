## 一、安装virtualbox
具体内容参考[virtualbox官方文档](https://www.virtualbox.org/wiki/Linux_Downloads)
### 1、添加virtualbox官方源
编辑/etc/apt/sources.list文件，加入以下内容：
```bash
deb http://download.virtualbox.org/virtualbox/debian xenial contrib
```
添加Oracle public key for apt-secure
```bash
sudo apt-key add oracle_vbox_2016.asc
sudo apt-key add oracle_vbox.asc
```
### 2、安装virtualbox
```bash
sudo apt-get install virtualbox
```

## 二、安装vagrant
具体内容参考[Vagrant官方文档](https://www.vagrantup.com/)
### 1、下载官方deb包
```bash
wget https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_x86_64.deb
```

### 2、安装vagrant
```bash
sudo dpkg -i vagrant_1.9.3_x86_64.deb
```

## 三、创建一个虚拟机

### 1、添加box文件
把制作好的centos6.box文件放在~/box目录下
```bash
vagrant box add centos6 ~/box/centos6.box
```

### 2、创建虚拟机目录
```bash
mkdir -p ~/vms
```

### 3、初始化
```bash
vagrant init centos6
```

### 4、启动虚拟机
```bash
vagrant up
```

