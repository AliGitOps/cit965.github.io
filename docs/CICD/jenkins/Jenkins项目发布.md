# Jenkins项目发布

[TOC]

## 前言

- 随着软件开发需求及复杂度的不断提高，团队开发成员之间如何更好的协同工作以确保软件开发的质量已经慢慢成为开发过程各种不可回避的问题。Jenkins自动化部署可以解决集成、测试、部署等重复性的工作，工具集成的效率明显高于人工操作；并且持续集成可以更早的获取代码变更的信息，从而更早的进入测试阶段，更早的发现问题，这样解决问题的成本就会显著下降；持续集成缩短了从开发、集成、测试、部署各个环节的时间，从而也就缩短了中间出现的等待时间；持续集成也意味着开发、测试、部署得以持续。所以当配置完Jenkins持续集成持续交付环境后，可以把发布的任务交给集成服务器去打理了。使用Maven(Ant)等来实现自动化构建发布部署。这些工具可以帮助在构建过程中实现自动化发布、回滚等工作。
- 本次课程我们来学习Jenkins代码发布的各个基础操作，为Jenkins的更高阶学习提供基础。

拓扑图

![image-20240803080419577](W:\images\image-20240803080419577-17226434738821.png)

## 资源列表

| 操作系统   | 主机名  | 配置 | IP             |
| ---------- | ------- | ---- | -------------- |
| CentOS 7.9 | jenkins | 2C4G | 192.168.93.101 |
| CentOS 7.9 | gitlab  | 2C4G | 192.168.93.102 |
| CentOS 7.9 | web01   | 2C4G | 192.168.93.103 |
| CentOS 7.9 | web02   | 2C4G | 192.168.93.104 |
| CentOS 7.9 | dev     | 2C4G | 192.168.93.105 |

## 基础环境

- 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

- 关闭selinux

```bash
setenforce 0
sed -i "s/.*SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config
```

- 修改主机名

```bash
hostnamectl set-hostname jenkins
hostnamectl set-hostname gitlab
hostnamectl set-hostname web01
hostnamectl set-hostname web02
hostnamectl set-hostname dev
```

## 一、Jenkins发布静态网站

### 1.1、项目介绍

- 本案例部署了一个简单的静态网站，通过此操作过程，主要掌握代码发布的基本流程，以及在这个过程中我们需要注意的重点环节，也就是掌握Jenkins项目发布的入门级操作。在这些操作中，进一步学习Jenkins持续集成、持续部署流程。

### 1.2、部署Web

- 两台web节点都要操作

```bash
yum -y install httpd
systemctl start httpd
systemctl enable httpd
```

### 1.3、准备gitlab

- 在gitlab节点操作

```bash
[root@gitlab ~]# cat > /etc/yum.repos.d/gitlab-ce.repo << 'EOF'
[gitlab-ce]
name=gitlab-ce
baseurl=http://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key
EOF
```

```bash
[root@gitlab ~]# yum -y install gitlab-ce-16.7.0-ce.0.el7
```

### 1.4、配置gitlab

```bash
[root@gitlab ~]# vim /etc/gitlab/gitlab.rb
# IP地址替换为自己的IP地址，然后保存退出即可
external_url 'http://192.168.93.102'


# 加载gitlab
[root@gitlab ~]# gitlab-ctl reconfigure


# 查看密码，然后更改密码，此次省略
[root@gitlab ~]# grep "Password:" /etc/gitlab/initial_root_password 
```

### 1.5、创建项目

- 访问gitlab地址：http://192.168.93.102

![image-20240803082354694](W:\images\image-20240803082354694.png)

![image-20240803082442874](W:\images\image-20240803082442874.png)

![image-20240803082514004](W:\images\image-20240803082514004.png)

### 1.6、推送代码

- dev节点操作

```bash
# 安装git命令
[root@dev ~]# yum -y install git


# 解压源代码
[root@dev ~]# tar -zxvf BlueLight.git.tar.gz 


# 拉取代码仓库
[root@dev ~]# git clone http://192.168.93.102/root/demo.git


# 复制源代码到代码仓库
[root@dev ~]# mv -f BlueLight/* demo/
[root@dev ~]# cd demo/


# 往main分支进行第一次推送
[root@dev demo]# git config --global user.email "you@example.com"
[root@dev demo]# git config --global user.name "Your Name"
[root@dev demo]# git add .
[root@dev demo]# git commit -m "first commit"
[root@dev demo]# git push -u origin main


# 设置一个tag为v1.0并且推送
# v1.0没有index.html页面
[root@dev demo]# git tag v1.0
[root@dev demo]# git push -u origin v1.0


# 设置一个tag为v2.0并且推送
# v2.0没有index.html页面
[root@dev demo]# cp bl-first-index.html index.html
[root@dev demo]# git add .
[root@dev demo]# git commit -m "first v2.0"
[root@dev demo]# git tag v2.0
[root@dev demo]# git push -u origin v2.0
```

## 二、Jenkins中创建gitlab凭据

### 2.1、创建凭据

- 详细步骤省略

![image-20240803084746892](W:\images\image-20240803084746892.png)

### 2.2、在Jenkins中添加远程主机

- “Manage Jenkins”——>“System”——>“Publish over SSH”，点击SSH Servers的新增按钮。须填写的信息如下：
  - Name：为远程主机的起的名字
  - Hostname：远程主机的IP地址或域名
  - Username：远程主机的登录账号
  - Remote Directory：远程同步路径（如果要拷贝文件，此处添加远程主机接口文件的目录）
  - 点击高级按钮，并勾选“Use password authentication，or use different key”
  - 在Passphrase/Password中输入密码
  - 其他保持默认，并点击test按钮进行连接测试，测试结果为Success表示参数设置成功
  - 最后保存设置
  - 可以用同样的方式添加更多的主机

![image-20240803085335786](W:\images\image-20240803085335786.png)

![image-20240803085520017](W:\images\image-20240803085520017.png)

![image-20240803085602212](W:\images\image-20240803085602212.png)

![image-20240803085644324](W:\images\image-20240803085644324.png)

### 2.3、获取gitlab项目的URL地址

![image-20240803085816503](W:\images\image-20240803085816503.png)

### 2.4、在Jenkins中创建webtest项目

![image-20240803085858051](W:\images\image-20240803085858051.png)

![image-20240803085932603](W:\images\image-20240803085932603.png)

### 2.5、配置源码管理

- 在源码管理中选择Git，并且把gitlab中获取的仓库URL填写进去。注意在“指定分支”的地方，将分支名称修改为“*/main”。在git中我们创建一个新项目的时候，项目的分支由早期的“master”，修改为现在的“main“，使用的时候注意这个变化。

```bash
[root@jenkins ~]# yum -y install git
```

![image-20240803090325031](W:\images\image-20240803090325031.png)

### 2.6、配置构建过程

- 在本案例中，我们需要将web网站的代码文件同步到web01主机，需要同步文件，需要一个发送文件的构建步骤，具体操作如下：
- 增加构建步骤”Send files or execute commands over SSH“需要设置的关键参数如下：
  - Name：在下拉菜单中选择目标主机
  - Source files：选择源文件位置，注意这里是工作目录的相对路径，不要些解决路径。如果要同步此目录下所有内容，就填写”***/*“；如果要同步工作目录下的img目录下的所有文件，就填写”img/*”
  - Remove prefix：该操作是针对上面的source files目录，会移除匹配的目录。通常留空
  - Remote directory：远程主机的同步目录，注意这里也是相对路径。是相对于远程主机的同步目录的，我们在前面的远程主机中设置同步的目录是“/var/www/html”，此处就直接些“/”，代表将文件同步到远程主机的“/var/www/html”目录下
  - 如果需要将文件批量同步到更多的主机，可以继续增加构建步骤。

![image-20240803090928486](W:\images\image-20240803090928486.png)

![image-20240803091146699](W:\images\image-20240803091146699.png)

### 2.7、构建项目

- 点击Jenkins项目，点“Build Now”或“立即构建”，如果成功将会在左下角看到绿色的标识

![image-20240803091324761](W:\images\image-20240803091324761.png)

### 2.8、访问验证

- 访问地址：http://192.168.93.103/bl-first-index.html

![image-20240803091420556](W:\images\image-20240803091420556.png)

## 三、Jenkins发布带有参数的项目

- 在刚才的案例中，我们掌握了项目发布的基本步骤，在实际工作过程中，程序员往往要对代码进行不断的升级，这时就出现了不同的版本，如果针对不同的项目版本进行发布，这也是Jenkins的一项基本功能。不仅能帮助管理员灵活的、有针对性的版本发布，同时在新版本出现bug的时候，又能快速的将项目回退到之前的版本。

### 3.1、修改General参数

- 勾选“This project js parameterized”，并点击“添加参数”，添加“Git Parameter”参数。设置的参数如下：名称：Tag 默认值：origin/main

![image-20240803092231758](W:\images\image-20240803092231758.png)

![image-20240803092302835](W:\images\image-20240803092302835.png)

### 3.2、修改源码管理

![image-20240803092347282](W:\images\image-20240803092347282.png)

### 3.3、构建项目

- 点击“Build Now”立即构建。额可以看到此处需要选择对应的标签版本
- v1.0没有index.html页面，v2.0有index.html页面

![image-20240803092512834](W:\images\image-20240803092512834.png)

## 四、Jenkins项目实时自动触发

- 在配置Jenkins实现前端自动化构建的过程中，Git如何通知Jenkins对应Job的工作区实时构建呢？web开发过程中的webhook，是一种通过通常的callback，去增加或者改变web page或者web app行为的方法。这些callback可以由第三方用户和开发维持当前，修改，管理，而这些使用者与网站或者应用的原始开发并没有关联。
- webhook这个词是由Jeff Lindsay在2007年计算机科学hook项目第一次提出的。Webhooks是“user-defined HTTP回调”。它们通常由一些事件触发，例如“push”代码到repo，或者“post一个评论道博客”。因此，我们可以将Jenkins的某个项目的webhook放置到gitbal，当gitlab中对应的项目代码有更新时，就会向jenkins触发一个构建的事件，这样就完成了一个项目自动触发的流程。

![image-20240803093151458](W:\images\image-20240803093151458.png)

### 4.1、设置触发器

- 项目——>“配置”——>“构建触发器”，勾选项目的webhook
- 复制出里面的webhook URL

![image-20240803093356607](W:\images\image-20240803093356607.png)

### 4.2、生成token

- 在“构建触发器”中生成一个Token，并且把这个Token复制出来

![image-20240803093520719](W:\images\image-20240803093520719.png)

![image-20240803093545659](W:\images\image-20240803093545659.png)

### 4.3、gitlab触发

- 单击Menu——>“Admin”

![image-20240803093646671](W:\images\image-20240803093646671.png)

### 4.4、Outbound requests

- 在这里要设置gitlab允许利用钩子（webhook）发送请求到本地网络
- 设置如下：Menu——>”Admin“——>"Settings"——>”Network“——>”Outbound requests“

![image-20240803093854356](W:\images\image-20240803093854356.png)

![image-20240803093953998](W:\images\image-20240803093953998.png)

![image-20240803094032859](W:\images\image-20240803094032859.png)

### 4.5、设置项目的webhook

- 打开自己创建的项目，Settings——>”Webhooks“
- 粘贴前面步骤中生成的webhook的URL和Token
- 最后点击页面底部的Add Webhook按钮

![image-20240803094223178](W:\images\image-20240803094223178.png)

![image-20240803094243976](W:\images\image-20240803094243976.png)

![image-20240803094358656](W:\images\image-20240803094358656.png)

![image-20240803094411943](W:\images\image-20240803094411943.png)

### 4.6、触发测试

![image-20240803094448770](W:\images\image-20240803094448770.png)

![image-20240803094509509](W:\images\image-20240803094509509.png)

### 4.7、手动触发测试

- 也可以手动测试，修改代码以后，提交上去

```bash
# 把默认dev主机存在的demo目录删除，重新拉取一个
[root@dev ~]# rm -rf demo/


# 拉取代码仓库
[root@dev ~]# git clone http://192.168.93.102/root/demo.git


[root@dev ~]# cd demo/
[root@dev demo]# date > time.log
[root@dev demo]# git config --global user.email "you@example.com"
[root@dev demo]# git config --global user.name "Your Name"
[root@dev demo]# git add .
[root@dev demo]# git commit -m "测试自动触发Jenkins"
[root@dev demo]# git push -u origin main
```

![image-20240803094910844](W:\images\image-20240803094910844.png)

## 五、Jenkins+ansible+gitlab实现项目发布

- 在此案例中，我们将进一步学习Jenkins较为复杂一点的应用，本案例将ansible集成到了jenkins中，让jenkins利用ansible插件，向远程主机推送文件和指令，完成自动化的项目部署。

- 在web集群中，我们可能有很多后端需要发布，这时候我们可以利用ansible批量进行发布

### 5.1、安装Ansible

- Jenkins节点操作

```bash
[root@jenkins ~]# yum -y install epel-release.noarch
[root@jenkins ~]# yum -y install ansible


# 取消/etc/ansible/anibsle.cfg文件host_key_checking = False的注释
[root@jenkins ~]# vim /etc/ansible/ansible.cfg
host_key_checking = False
```

### 5.2、配置ansible主机清单

```bash
[root@jenkins ~]# cat >> /etc/ansible/hosts << EOF
[webservers]
# web01
192.168.93.103 ansible_ssh_user=root ansible_ssh_pass=wzh.2005
# web02
192.168.93.104 ansible_ssh_user=root ansible_ssh_pass=wzh.2005
EOF
```

### 5.3、Jenkins创建webansible项目

![image-20240803100300359](W:\images\image-20240803100300359.png)

### 5.4、配置General

![image-20240803100347208](W:\images\image-20240803100347208.png)

![image-20240803100428464](W:\images\image-20240803100428464.png)

### 5.5、配置源码管理

![image-20240803100543501](W:\images\image-20240803100543501.png)

### 5.6、配置Build Steps

- 在”Build Steps“中，点”增加构建步骤“——>”Invoke Ansible Ad-Hoc Command“，在这里设置的主要参数如下：
  - Host pattern：设置ansible中的主机组的名字，本案例中我们用的是”webservers“
  - Inventory：选择File or host list，添加的文件是ansible的主机清单/etc/ansible/hosts
  - Moundle：设置同步方式，此处使用”synchronize“的方式，表示使用rsync同步
  - Module arguments or command to execute：填写ansible的同步命令：命令如下

```bash
src=./ dest=/var/www/html rsync_opts=--exclude=.git delete=yes


# 备注：ansible命令解释
rsync_opts=--exclude=.git：同步时将.git文件除外，该文件不同步
delete=yes：使两边的内容一样（即以推送方为主）
```

![image-20240803101205333](W:\images\image-20240803101205333.png)

### 5.7、增加构建步骤

![image-20240803101252411](W:\images\image-20240803101252411.png)

```bash
# 使用ansible给web节点的网页重新授权
ansible webservers -m shell -a "chmod -R 755 /var/www/html"
```

```bash
# 安装所需同步软件
[root@jenkins ~]# yum -y install rsync
[root@web01 ~]# yum -y install rsync
[root@web02 ~]# yum -y install rsync
```

![image-20240803101416960](W:\images\image-20240803101416960.png)

### 5.8、构建项目

![image-20240803101645619](W:\images\image-20240803101645619.png)

### 5.9、验证

- 可以查看web01和web02节点的网页是否存在

```bash
[root@web01 ~]# ls /var/www/html/
aos             bl-aritical.html       bootstrap     highlight  LICENSE
bl-about2.html  bl-aritical-list.html  css           img        README.md
bl-about.html   bl-first-index.html    font-awesome  jquery     screenshots
```

```bash
[root@web02 ~]# ls /var/www/html/
aos             bl-aritical.html       bootstrap     highlight  LICENSE
bl-about2.html  bl-aritical-list.html  css           img        README.md
bl-about.html   bl-first-index.html    font-awesome  jquery     screenshots
```

- 也可以浏览器进行访问，两个网站内容一样

![image-20240803101913721](W:\images\image-20240803101913721.png)

![image-20240803101924196](W:\images\image-20240803101924196.png)