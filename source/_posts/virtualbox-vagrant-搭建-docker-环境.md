---
title: virtualbox + vagrant 搭建 docker 环境
date: 2018-09-27 14:39:35
tags:
cover: https://knowledge-payment.oss-cn-beijing.aliyuncs.com/others/44e818397558c3770b38c6978f35cbb6.png
---

# 安装 Virtualbox

下载地址：https://www.virtualbox.org/wiki/Downloads

---

# 安装 Vagrant

下载地址： https://www.vagrantup.com/downloads.html

---

 vagrant 常用命令

| 命令行 | 说明 |
| -- | -- |
| vagrant init | 初始化 vagrant |
| vagrant up | 启动 vagrant |
| vagrant reload |  重启 vagrant |
| vagrant halt | 关闭 vagrant |
| vagrant ssh | 通过 SSH 登录 vagrant |
| vagrant provision | 重新应用更改 vagrant 配置 |
| vagrant destroy | 删除 vagrant |


# 配置 Vagrantfile

```vb
Vagrant.require_version ">= 1.6.0"

boxes = [
    {
        :name => "docker-host",
        :eth1 => "192.168.10.15",
        :mem => "1024",
        :cpu => "1"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.provider "vmware_fusion" do |v|
        v.vmx["memsize"] = opts[:mem]
        v.vmx["numvcpus"] = opts[:cpu]
      end
      config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
      end
      config.vm.network :private_network, ip: opts[:eth1]
    end
  end
  config.vm.synced_folder "F:\Learn", "/home/vagrant/learn"
  config.vm.provision "shell", privileged: true, path: "./setup.sh"
end
```

setup.sh 脚本

```bash
#/bin/sh

# install some tools
sudo yum install -y git vim gcc glibc-static telnet

# install docker
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh

# start docker service
sudo groupadd docker
sudo gpasswd -a vagrant docker
sudo systemctl start docker

rm -rf get-docker.sh
```

通过上述步骤，就可以成功搭建一台预装了 docker 的 centos7 虚拟机了。


# 参考

* [vagrant 官方文档](https://www.vagrantup.com/docs/)
* [Vagrant的配置文件Vagrantfile详解](https://blog.csdn.net/u011781521/article/details/80291765)
