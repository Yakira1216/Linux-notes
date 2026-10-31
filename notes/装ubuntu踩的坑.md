# 第一次装 Ubuntu：我踩的坑

## 1. 我要做什么

在 Windows 上装一台能上网、能随时搞坏的 Ubuntu 虚拟机

## 2. 我的环境

- VirtualBox 7.2.22
- Ubuntu 22.04.5 LTS Desktop
- 内存 4096 MB / CPU 2 核 / 硬盘 40 GB（VDI 动态分配) / 显存128MB
- 网卡1 = NAT（上网）　网卡2 = 仅主机 Host-Only（宿主机直连）

## 3. 我做了什么

1. 装 VirtualBox，用默认路径（没改到 E 盘）
2. 下 Ubuntu 镜像放 E:\ISO
3. 新建虚拟机，类型改 Linux / Ubuntu 64-bit，取消无人值守安装
4. 挂 ISO 启动，选 English、Erase disk、时区 Shanghai
5. 装完先「设备 → 分配光驱 → 移除磁盘」，再重启
6. 打快照「干净系统」
7. 换阿里源
8. 装增强功能，重启验证，再打第二个快照

## 4. 我踩的坑

### 坑1：virtualbox拒接安装到E盘
- 现象：选好 E:\Program Files\Oracle\VirtualBox，点下一步弹窗：Invalid installation directory — does not meet the security requirements
- 原因：E 盘根目录的权限比系统盘松，E:\Program Files 继承了 BUILTIN\Authenticated Users:(I)(M)（所有登录用户可修改）。VirtualBox 7.x 会检查安装目录是否可被普通用户写入——能写就能被塞恶意 DLL，它直接拒绝。C:\Program Files 只有 Users:(RX)，所以默认路径能过。
- 解决：先用默认路径C盘，下载虚拟机时将安装包默认路径改为E盘

- ### 坑2：虚拟机建完就消失，注册表留下坏记录
- 现象：摘要页报“未能覆盖介质”，点完成 / 取消后，E:\VirtualBox\ubuntu22 整个目录被删空，但 VirtualBox 里还显示着 ubuntu22，双击报“无法保存虚拟电脑设置”“无法访问”。
-原因：向导中途出错会回滚、删掉半成品文件，但 VirtualBox.xml 里的 MachineEntry 注册记录没清；同时后台服务 VBoxSVC.exe 内存里还缓存着那个已经不存在的 VDI，于是“想删删不掉、想改改不了”。
-解决：彻底退出 VirtualBox（含右下角托盘图标）→ 任务管理器结束 VBoxSVC.exe → 重新打开 → 右键 ubuntu22 移除 → 重建。

### 坑3：安装器崩溃，但系统其实装好了
-现象：点完 Restart Now，弹出 Installer crashed，报错 ValueError: invalid literal for int() with base 10: ''，堆栈停在 copy_debconf。
-原因：安装的收尾阶段要把 debconf 数据库拷过去，debconf-communicate 返回了空字符串，int('') 直接崩。触发原因：在“安装完成”之后先强制弹出光盘，打断了这个收尾动作。
-解决：先别重装。回宿主机看虚拟磁盘大小：Get-Item E:\VirtualBox\ubuntu22\ubuntu22.vdi，如果已经 11–12 GB，说明系统早写完了。关机 → 确认光驱是“没有盘片” → 重新启动

###坑4：代理软件劫持了apt流量
-现象换到清华源后，报错在三种形态之间变：Connection failed [IP: 198.18.1.131 80]、Could not handshake: The TLS connection was non-properly terminated [IP: 198.18.1.131 443]、关掉代理后又变成 Could not resolve。
-原因：198.18.0.0/15 是基准测试保留网段，真实网络里不会出现。看到它就说明有 TUN 模式的代理软件（Clash 一类）在用 fake-ip 接管 DNS，把所有域名解析成假地址。80/443 都走它的隧道，TLS 握手被它打断。
-解决：退出代理 / 加速器 → 换一个镜像源 → 如果 DNS 还有缓存残留，重启虚拟机或执行 sudo resolvectl flush-caches。

###坑5：换源四五次都没有生效
-现象：sed 跑完没报错但文件没变；gedit 里 Replace All 明明改了，重启后又变回去；tee 追加的行写进了别的文件。
-原因：三个独立原因叠在一起：① 文件里的域名可能混进了形近字符，按域名匹配匹配不上；② gedit 改完没按 Ctrl+S，改动只在内存里，标题栏的星号就是提示；③ tee -a 的路径拼错，行写进了一个 apt 根本不读的新文件。
-解决：① 不要按域名匹配，改用行级操作：sudo sed -i /tuna/d /etc/apt/sources.list 直接删掉含关键词的整行；② 改完必须验证，用 grep；③ 路径拼不准时用通配符 sp*，让 shell 自己匹配。

## 5. 我现在会什么

- 能闭卷重装一台 Ubuntu（做过一次，算上加载时间约 90 分钟）
- 能配双网卡（NAT + 仅主机）
- 遇到报错先看第一个关键词，再用 `ls` / `which` 验证
- 会使用Linux的一些基础命令

- ##6.结论
- 这件事本身没有难点，在virtualBox装Ubuntu是一条走了十几年的标准路径，在官方文档中仅用四行就写完了，真正花时间的是判断问题出在哪里，如果说是新手那么装机中出现的所有问题都只能靠AI来解决，在排错中就会浪费很多时间，还不包括敲错命令，例如同样是 apt 报错：Could not resolve 是 DNS，Connection failed 是网络，Could not handshake 是 TLS，has no installation candidate 是源配置——四条报错，四个完全不同的方向。只要分不清层，就会一直在原地重试。
