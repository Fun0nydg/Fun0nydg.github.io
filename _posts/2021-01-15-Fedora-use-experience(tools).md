---
layout: post
title:  "Fedora使用心得-工具篇"
---
## 前言 
在去年10月份的时候，和一位前辈闲聊，聊到了保值技能，前辈建议我好好学习linux，熟练了linux不仅可以提高办公效率，而且也安全，更重要的是，你可以更好的去hack linux，于是，在前辈的推荐下，我将平时工作的操作系统换成了Fedora，开启了我的漫长踩坑之路


## 常用工具
安装好Fedora后，你的系统的工具少的可怜...放眼望去，浏览器只有firefox，办公软件只有难用的libreoffice...
所以，让我们开始完善我们的Fedora吧！当你搭建好了之后，有一种组装汽车的成就感～

#### Chrome
浏览器怎么能没有Chrome呢，毕竟谷歌大法好！
参考:  
<https://docs.fedoraproject.org/en-US/quick-docs/installing-chromium-or-google-chrome-browsers/>  

你可以选择安装chromium,dnf package自带
```bash
dnf install chromium
```
当然我没有安他，毕竟有时候还是要娱乐一下，看一看视频，这时候chromium就不行了，涉及到国外的一些开源许可以及法律法规，这时候我们就需要安装Chrome了，默认dnf package是没有的，我们需要前往  
<https://www.google.com/chrome/browser/desktop/index.html>  
选择rpm包下载，安装好rpm包后先去寻找有哪些版本:
```bash
dnf search google-chrome*
```
选择稳定版本
```bash
dnf install google-chrome-stable.x86_64
```

### Vmware
参考   
<https://docs.fedoraproject.org/en-US/quick-docs/how-to-use-vmware/>  
<https://ericclose.github.io/install-VMware-Workstation-on-Fedora-30.html>  
安装前的准备，确保make成功，切记！
```bash
sudo dnf install kernel-devel kernel-headers gcc gcc-c++ make git
```
下载Vmware的安装程序之后，赋予执行权限，这里x.y.z是版本号
```bash
sudo chmod +x ./VMware-Workstation-Full-x.y.z-nn.x86_64.bundle
```
运行安装程序
```bash
sudo ./VMware-Workstation-Full-x.y.z-nn.x86_64.bundle
```
当然这样安装肯定会出问题的，需要patch，下载patch,找到对应的版本号
```bash
wget https://github.com/mkubecek/vmware-host-modules/archive/workstation-x.y.z.tar.gz
```
解压并进入目录
```bash
tar -xzf workstation-x.y.z.tar.gz && cd vmware-host-modules-workstation-x.y.z
```
make
```bash
make
```
再安装
```bash
sudo make install
```
这时再运行就正常了

### 网易云
虽然网易云音乐只有deb包，但是没关系，我们有flatpak，当你遇到各种各样依赖问题的时候，flatpak安装软件是一个很好的解决方案，Fedora最新版默认是自带的，如果你的没有就去dnf安装一个,当然，这里面有好多是用户自己打包的，虽然flatpak会在沙箱里面运行，但是之前也有恶意的打包程序，我的推荐是，对于不支持rpm的，可以用flatpak，注意查看别人在github上的代码，多留个心眼
flathub地址
```html
https://flathub.org/
```
安装
```bash
flatpak install flathub com.netease.CloudMusic
```

运行的时候注意，这里会有输入法的问题，如果直接运行你可能输入不了中文，解决方法带个env参数，默认的输入法是ibus:

```bash
flatpak run --env=QT_IM_MODULE=ibus com.netease.CloudMusic
```
当然，如果你嫌命令太长，想要优雅的打开,你可以这样
修改.bashrc文件
```bash
vim ~/.bashrc
```
在最后添加
```bash
alias cloudmusic='flatpak run --env=QT_IM_MODULE=ibus com.netease.CloudMusic'
```
ctrl+c退出insert之后，shift+;输入wq
第二天的清晨，当你重启电脑之后，你就可以优雅的在bash中输入
```bash
cloudmusic
```
尽情享受音乐吧

### htop
top看起来很麻烦，不够直观，我们用htop
```bash
dnf install htop
```

### VSCode
参考官网
```html
https://code.visualstudio.com/docs/setup/linux
```
导入微软签名及微软的源
```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
```
更新并安装
```bash
sudo dnf check-update
sudo dnf install code
```

### WPS Office
虽然说windows版很多广告弹窗，但是linux版非常干净，官网地址：  
<https://www.wps.cn/product/wpslinux>  
选择```x64```的rpm包
下载好rpm包安装会提示缺少依赖的
我们需要安装mesa-libGLU
```bash
dnf install mesa-libGLU
```
之后再去安装wps就ok


### Docker
有时候搭建环境很麻烦，或者需要运一些不可靠的脚本，这时候docker可以很好解决这些顾虑，当然，docker也不是绝对安全的，也会有容器逃逸漏洞，
这时候我们可以选择rootless脚本，这样不需要root权限去启动docker，同时不要用host模式，docker默认是bridge模式，具体关于docker的保护
后续再说
那么我们来通过rootless脚本去安装docker,根据官网的教程便可以安装:  
<https://docs.docker.com/engine/security/rootless/>  
传统安装:  
<https://docs.docker.com/engine/install/fedora/>    
### Wechat
虽然说微信可以网页版登录，但是有不少用户出于安全策略都无法登录，比如说我...为了寻找解决方法，我在github上搜到了一个docker微信：  
<https://github.com/huan/docker-wechat>  
**如果说不担心被封号的同学，可以尝试docker微信，由于我自己没有使用过，所以不敢保证其稳定性**，我的解决方法是用一个Windows虚拟机装一个微信，虽然麻烦，但是比较稳定。  

### tmux
```bash
dnf install tmux
```
不用多说，神器。

### 010Editor
在linux下效率很高，十分推荐,官网地址:  
<https://www.sweetscape.com/010editor/>  
### KeePassXC
平常工作中我们会有很多帐号的密码，有些帐号可能不是特别重要但是又需要一定的密码强度，这时候我们就需要一个密码管理工具了，我推荐KeePassXC  
dnf管理包中就有，直接可以安装:
```bash
dnf install keepassxc
```

未完代续




