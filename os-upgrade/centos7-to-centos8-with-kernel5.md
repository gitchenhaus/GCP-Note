# CentOS 7 升级 CentOS 8 包含 Kernel 5 脚本与说明

## 安装前环境确认
1. OS: CentOS 7.8.2003
2. Kernel: 4.19.113-300.el7.x86_64
```<bash>
cat /etc/redhat-release && uname -r
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.05.14%20PM.png)

## 开始执行升级步骤
1. 安装 epel yum Repository
```<bash>
yum install -y epel-release
```
![](../images/centos7-to-centos8-with-kernel5/4A4D5A1C-AAB5-4EDC-BF8A-1F96B1F6EC64.png)
截图中因为已经安装过，所以没有更新的档案

2. 安装升级过度用套件
```<bash>
yum install -y yum-utils rpmconf
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.39.34%20PM.png)

3. 清除已不再被依赖的套件
```<bash>
package-cleanup --leaves|tail -n +2|xargs yum remove -y
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.39.58%20PM.png)

4. 安装 dnf
```<bash>
yum install -y dnf
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.40.16%20PM.png)

5. 移除 yum
```<bash>
dnf remove -y yum yum-metadata-parser
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.40.31%20PM.png)

6. 安装 CentOS 8 RPM 
```<bash>
dnf install -y http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-repos-8.2-2.2004.0.1.el8.x86_64.rpm http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-release-8.2-2.2004.0.1.el8.x86_64.rpm http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/centos-gpg-keys-8.2-2.2004.0.1.el8.noarch.rpm
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.43.11%20PM.png)

7. 汇入 gpg key
```<bash>
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.43.33%20PM.png)

8. 升级 epel
```<bash>
dnf upgrade -y epel-release
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.44.21%20PM.png)

9. 清除旧的 kernel 与冲突套件
```<bash>
rpm -e `rpm -q kernel`
rpm -e --nodeps sysvinit-tools
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.44.41%20PM.png)

10. 移除 python3
```<bash>
dnf remove -y python3
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.44.55%20PM.png)

12. 升级 CentOS 8
```<bash>
dnf -y --releasever=8 --allowerasing --setopt=deltarpm=false distro-sync
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.49.55%20PM%20(2).png)

13. 升级 kernel RPM
```<bash>
dnf install -y http://repos.ord.lax-noc.com/elrepo/archive/kernel/el8/x86_64/RPMS/kernel-ml-core-5.7.12-1.el8.elrepo.x86_64.rpm
dnf install -y http://repos.ord.lax-noc.com/elrepo/archive/kernel/el8/x86_64/RPMS/kernel-ml-modules-5.7.12-1.el8.elrepo.x86_64.rpm
dnf install -y http://repos.ord.lax-noc.com/elrepo/archive/kernel/el8/x86_64/RPMS/kernel-ml-5.7.12-1.el8.elrepo.x86_64.rpm
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%2012.52.19%20PM.png)
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%201.44.22%20PM.png)
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%201.44.35%20PM.png)

14. 设定开机选单不倒数并重新产生开机设定
```<bash>
sed -i 's/GRUB_TIMEOUT=5/GRUB_TIMEOUT=0/g' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%201.45.14%20PM.png)

16. 将旧的 yum 资料夹备份
```<bash>
mv  /etc/yum /etc/yum.bak
```

17. 重新安装 yum
```<bash>
dnf install -y yum
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%201.45.24%20PM.png)

## 检查验证
1. OS: `CentOS 8.2.2004`
2. Kernel: `5.7.12-1.el8.x86_64`
```<bash>
cat /etc/redhat-release && uname -r
```
![](../images/centos7-to-centos8-with-kernel5/Screen%20Shot%202020-11-04%20at%202.05.22%20PM.png)

3. 确认无误之后重启机器
```<bash>
reboot
```

4. 透过 ssh 从新登入机器，确认 ssh 功能正常
```<bash>
ssh <username>@<hostname> -P <Port>
```