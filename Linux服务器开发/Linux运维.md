# 1 进程管理

进程是已启动的可执行程序的运行实例，进程有以下组成部分：

* 一个文件
* 被分配内存的地址空间
* 有权限限制
* 程序代码的一个或多个副本（也叫执行线程）
* 像人一样拥有状态

# 2 磁盘管理

## 2.1 分类

机械硬盘：即是传统普通硬盘，主要由：盘片，磁头，盘片转轴及控制电机，磁头控制器，数据转换器，接口，缓存，接口，缓存几个部分组成。

固态硬盘：固态硬盘是固态电子存储芯片阵列而制成的硬盘。

![](G:\images\Linux\image-20220830153321223.png)

## 2.2 命名

kernel对不同接口硬盘命名方式：

IDE（并口）：

1) /dev/hda
2) /dev/hdb

SATA(串口)：

1) **/dev/sda**	(/dev 设备文件目录 	s代表sata串口	d代表磁盘	**a代表第一块**)
2) **/dev/sdb     (/dev 设备文件目录 	s代表sata串口	d代表磁盘	**b代表第二块**)
3) /dev/sda1     (/dev 设备文件目录 	s代表sata串口	d代表磁盘	**a代表第一块**      1代表第一个分区)

## 2.3 分区方式

**MBR：**主引导记录（MBR，Master Boot Record）是位于磁盘最前边的一段引导

MBR支持最大的磁盘容量 < 2TB。设计时分配4个分区，如果希望超过4个分区，需放弃主分区，改为扩展分区和逻辑分区

**GPT：**全局唯一标识分区表，是一个实体硬盘的分区表的结构布局的标准。

GPT支持大于2T的硬盘，支持128个分区

## 2.4 磁盘管理

> 查看分区UUID：**blkid**

管理磁盘流程三部曲：分区（MBR或者GPT）——-> 格式化/文件系统Filesystem ———>挂载mount

> 命令	**lsblk 用于查看块设备文件详细信息**

![](G:\images\Linux\image-20220830171639879.png)

### 2.4.1 启动分区工具

> $	fdisk	/dev/sdb

### 2.4.2 进入会话模式

1) ![](G:\images\Linux\image-20220830162726680.png)

   2) 操作：敲击字母“n”键

      说明：请求划分分区

      ![](G:\images\Linux\image-20220830162947770.png)

   3) 操作：敲击字母“p”键

      说明：选择划分主分区，**MBR最多可以划分4个主分区**

      ![](G:\images\Linux\image-20220830163315913.png)

      ​	

   4) 操作：敲击数字“1”键

      说明：选择分区的编号，默认为1

      ![](G:\images\Linux\image-20220830163513772.png)

5) 操作：敲击回车（选择默认）

   说明：选择起始扇区，默认即可，默认是上一分区的末尾，磁盘开头会有MBR分区信息，故分区扇区号不是从0开始

   ![](G:\images\Linux\image-20220830164132731.png)

6) 操作：输入分区大小“+2G”后回车（+2表示2扇区，+2G则表示2G个空间对应的扇区）

   说明：选择磁盘分区结束的扇区，即分区大小

​		![](G:\images\Linux\image-20220830164740897.png)

7) 操作：输入“w”保存分区信息，或继续按“n"继续分区

   说明：保存并自动退出分区工具

   ![](G:\images\Linux\image-20220830164959225.png)

   8) 操作：执行命令：**partprobe	/dev/sdb**	（其实会自动刷新，该命令只是确保）

      说明：刷新分区表

      ​	

   9) 操作：执行命令：**fdisk    -l   /dev/sdb**     或      lsblk

      说明：查看磁盘分区信息

​			![](G:\images\Linux\image-20220830165538184.png)

### 2.4.3 创建文件系统（格式化）

操作：执行命令：mkfs.ext4	/dev/sdb1	(make file system extend4) 或 **mkfs	-t	ext4	/dev/sdb2**

![](G:\images\Linux\image-20220830170120689.png)

### 2.4.4 挂载

操作：创建挂载点

> mkdir	/mnt/disk1

操作：挂载

> mount	-t 	ext4	/dev/sdb1	/mnt/disk1

操作：查看挂载信息

> **df	-hT**	分区	**文件系统**	大小	占用	空闲	占比	**挂载点**		（diskfree	-h（人性化显示空间）-T(type)分区类型）

![](G:\images\Linux\image-20220830170717849.png)

> **MBR最多只能划分4个主分区，要想有更多的分区，必须放弃一个主分区，在该主分区，划分多个逻辑分区，一般放弃最后一个主分区。**

### 2.4.5 划分扩展分区（N个逻辑分区）

> 使用MBR，划分分区。数量上只能有如下选择：
>
> 1) 4个主分区
> 2) 3个主分区加1个扩展分区（N个逻辑分区）
> 3) 1个主分区加1个扩展分区（N个逻辑分区）

1) 操作：敲击“d”

​		说明：删除一个分区

2) 操作：敲击“m“

​		说明：查看命令说明

​		![](G:\images\Linux\image-20220831154031370.png)

3) 操作：敲击“p”

   说明：打印分区情况

**划分逻辑分区**

1) 操作：敲击“n"，后敲击”e“

   说明：**划分出一片扩展分区，只有在扩展分区里，才能划分逻辑分区。**扩展分区不能格式化挂载，需要划分逻辑分区。

2) 操作：敲击“n”

   说明：划分逻辑分区

   ![](G:\images\Linux\image-20220831154956537.png)

3) 操作：**使用mkfs.ext4和mount命令格式化和挂载逻辑分区**

## 2.5 交换分区管理 Swap

> 作用：“提升”内存的容量，防止OOM(Out Of Memory)
>
> 推荐：设置交换分区大小为内存的2倍
>
> 查看当前的交换分区：**free	-mh**

> **增加Swap分区，fdisk划分分区后/dev/sde1，将类型设置为82（按t)** //**类型也可以不改，默认也可以**
>
> (增加交换分区有两种方法，一种是新的交换分区来做swap，另外一种是在磁盘中创建一个大的文件来做swap）
>
> 刷新分区表：**partprobe	/dev/sdc**
>
> 格式化：**mkswap	/dev/sdc1**
>
> 挂载：**swapon	/dev/sdc1**
>
> 删除对应的Swap分区：**swapoff	/dev/sdc1**

## 2.6 卸载（umount）

【注意】**mount挂载是临时的，当关机重启后需要重新挂载，若要永久挂载，需要编辑系统配置文件	/etc/fstab** 

> 当挂载目录中已经有文件，则会隐藏掉这些文件，显示为**lost+found**
>
> **umount	设备文件名或挂载点**
>
> 卸载该区后，又能查看这些文件了。

**填充磁盘数据**

> watch	-n0.5	“**dd	if=/dev/zero	of=/mnt/disk4/1.txt	bs=1M	count=1000**”
>
> if（input flow）of(output flow)	bs(block size)

## 2.7 逻辑卷LVM

> 目的：管理磁盘的一种方式，性质与基本磁盘无异
>
> 特点：随意扩张大小
>
> PV：物理卷（Physical volume）
>
> VG：卷组（Volume Group）
>
> LV：逻辑卷（Logical Volume）

1) 操作：**pvcreate	/dev/sdf**

   说明：创建物理卷

2) 操作：**vgcreate    vg1    /dev/sdf**

   说明：创建卷组

3) 操作：**lvcreate    -L    200M    -n   lv2    vg1**

   说明：创建逻辑卷（从卷组中取空间创建逻辑卷）    -L 大小   -n卷名     vg1组名

4) 操作：mkdir    /mnt/lv1

   说明：创建挂载点

5) 操作：**mount    /dev/vg1/lv1    /mnt/lv1**

   说明：**挂载逻辑卷**    /dev/卷组名/逻辑卷名

### 2.7.1 VG管理

目标：/dev/vg1容量由4G扩容到6G

步骤1：创建PV。而后使用第二步，将PV增加到VG中。

> **pvcreate	/dev/sdd1**

步骤2：扩展VG，同时包含方法1.

> **vgextend	vg1	/dev/sdd1**
>
> Volume group "vg1" successfully extended

**查看物理卷**

> **pvs**

![](G:\images\Linux\image-20220831195801970.png)

**查询卷组**

> **vgs**

![](G:\images\Linux\image-20220831200002645.png)

**查询逻辑卷**

> **lvs**
>
> **lvreduce	-L	80G	/dev/vg1/lv1**	#缩小逻辑卷

![](G:\images\Linux\image-20220831200122515.png)

### 2.7.2  逻辑卷（lv）扩容

步骤1：查看VG空间，是否有剩余空间

> **vgs**

步骤2：扩容lv

> **lvextend	-L	+200M	/dev/vg1/lv1**
>
> 增加200M空间，给lv2。

步骤3：增加到文件系统FS

> **resize2fs	/dev/vg1/lv1**

### 2.7.3 删除

步骤1：卸载对应卷组的所有逻辑卷LV

> **umount	/dev/vg1/lv1**

步骤2：删除逻辑卷LV

> **lvremove	/dev/vg1/lv1**
>
> lvdisplay	|	grep	"/dev/vg1/lv1"	#查看是否成功删除

步骤3：删除卷组

> **vgremove	/dev/vg1**

步骤4：删除物理卷

> **pvremove	/dev/sda7**

步骤5：编辑/etc/fstab 删除对应的挂载点信息

## 2.8 文件系统

> 文件系统类型：
>
> Windows：FAT16	FAT32	NTFS
>
> Linux：EXT3	EXT4	XFS
>
> **EXT3/4文件系统：**
>
> 类型：索引文件系统
>
> 系统限制：
>
> * Ext3：文件系统最大16TB
> * Ext4：文件系统最大16TB
> * XFS：文件系统最大100TB
>
> 1) block：
>
>    **存储文件的实际基本单位**
>
>    **block大小默认为4KB**
>
>    **若文件较大，会占用多个block。但一个块只能存放一个文件**
>
> 2) inode：
>
>    记录文件的属性（文件的元数据metadata，**大小，权限，属主，属组，连接数，块数量，块的编号**）
>
>    一个文件占用一个inode，同时记录此文件数据所在的block number。
>
>    **inode大小为128bytes**
>
>    **df	-i	/	ls	-i 可以查看各文件系统inode信息，包括可以创建的inode上限。inode决定了文件系统中文件数量的上限。**
>
> 3) superblock：
>
>    block和inode的总量
>
>    未使用和已使用的inode/block的数量

## 2.9 RAID（廉价的磁盘冗余阵列）

> 作用：提高数据读取速率，备份数据
>
> RAID0	RAID1	RAID5（一块硬盘作校验盘）
>
> 类型：硬RAID	软RAID
>
> 创建软RAID：
>
> 1) 准备4块硬盘（RAID5：3块数据盘(其中一个校验盘) + 1块热备硬盘）
>
> 2) 创建RAID
>
>    安装mdadm，yum -y install mdadm
>
>    **mdadm	-C	/dev/md0	-l5	-x1	/dev/sd{d,e,f,g}**
>
>    **mdadm	-D	/dev/md0	//查看RAID信息**
>
>    -C	创建RAID
>
>    /dev/md0	命名RAID设备
>
>    -l5	RAID5
>
>    -n	RAID成员的数量
>
>    -x	热备磁盘的数量
>
> 3) 格式化
>
>    mkfs.ext4	/dev/md0
>
> 4) 挂载
>
>    mkdir	/mnt/raid5
>
>    mount	/dev/md0	/mnt/raid5/

# 3 软件安装和卸载

Linux安装软件和Windows安装软件的概念不太一样。

Windows总是把一个软件安装在一个指定的文件夹里面。而Linux不一样，Linux是把要安装的软件发布在整个系统的各个文件夹里面，比如所有软件的配置文件都安装在/etc下面，软件需要的库文件都安装在/lib下面，日志文件都在/var/log下面，/bin下是常用的程序，等等。

## 3.1 RPM包管理

**RPM包：**

> RPM Package Manager(原Red Hat Package Manager，现在是一个递归缩写)，由Red Hat公司提出，被众多Linux发行版所采用。
>
> RPM包也被称为二进制（binary code）包无需编译，可以直接使用，无法设定个人设置，开关功能。
>
> 软件包示例：mysql-community-common-5.7.12-1.el7.x86_64.rpm （**软件名称 版本号(主版本、次版本、修订号) 操作系统（enterprise linux 7） cpu架构**）
>
> 操作系统：el6 el5 fedora suse debin ubuntu
>
> x86_64：表示64位软件
>
> noarch：表示32、64位通用

**源码包：**

> **源代码包(source code)需要编译环境编译才能运行**
>
> 可以修改源码设定个人设置，如安装位置，开关功能，但配置复杂
>
> 软件包示例：nginx-1.8.1.tar.gz
>

### 3.1.1 YUM工具

> Yum(全称为 Yellow dog Updater，Modified)，是一个在Fedora和RedHat以及CentOs中的Shell前端软件包管理器。
>
> 基于RPM包管理，能够从指定的服务器自动下载RPM包并且安装，**可以自动处理依赖性关系，并且一次安装所有依赖的软件包，无需繁琐地一次次下载、安装。**
>
> YUM本地源：系统安装光盘：通知linux服务器，通过本机的系统光盘获得软件包，并安装软件，操作系统安装包中的Packages下有很多rpm包。
>
> 1) 设置yum核心配置：**/etc/yum.repos.d/**
> 2) 删除官方YUM库
> 3) **编写YUM库本地源**，从本地rpm包安装
>
> > ```shell
> > [dvd]
> > name=dvd
> > baseurl=file:///mnt/cdrom
> > gpgcheck=0
> > enabled=1
> > ```
> >
> > /etc/yum.repos.d是YUM下载地址库目录
> >
> > [dvd]是某一个仓库的名称，中括号[]是必须的
> >
> > name=dvd是库的说明，name是必须的
> >
> > baseurl=file:///mnt/cdrom下载库的具体地址
> >
> > gpgcheck=0是关闭校验
> >
> > enabled=1是下载库链接的开关
>
> 4) 挂载安装光盘
>
>    > 【注】**/dev/sr0时光驱的设备名，/dev/cdrom代表光驱**
>    >
>    > **cdrom时sr0的软链接。**
>    >
>    > mkdir 	/mnt/cdrom	#创建挂载点
>    >
>    > **mount 	/dev/cdrom	/mnt/cdrom**	#挂载光盘前，需确保光盘已连接
>    >
>    > 把mount 	/dev/cdrom	/mnt/cdrom命令写到/root/.bashrc文件中，开机自动挂载。

> **查看帮助手册**
>
> > yum –help
>
> **安装**
>
> 全新安装
>
> > **yum -y install httpd vsftpd**  # -y表示自动回复yes，可以不加，httpd，vsftpd为软件包
>
> 重新安装
>
> > **yum -y reinstall httpd**
>
> 升级安装
>
> > **yum -y update httpd**
>
> **查询**
>
> > **yum list httpd**	#**安装库列下带"@"表示该包已安装**
> >
> > yum list | more
> >
> > **yum provides vim # 查询文件属于哪个软件**
>
> **卸载**
>
> > **yum -y remove httpd**

### 3.1.2 RPM工具

> 管理rpm包的基本工具，**但不能自动处理依赖关系。**
>
> **安装：**
>
> > \# rpm -ivh 软件包名称 
> >
> > -i install 
> >
> > -vh verbose human 
> >
> >  **rpm -ivh lftp-4.0.9-1.el6.x86_64.rpm**
>
> **查询软件安装路径：**
>
> >  rpm -ql 软件名称
> >
> > -q query 
> >
> > -l list
> >
> > **rpm -ql lftp-4.0.9-1.el6.x86_64** 
>
> **查询软件的配置文件**
>
> >  rpm -qc 软件名称 
> >
> > rpm -ql 软件名称	//查询与软件相关的所有文件
>
> **查询某一个文件是哪个软件产生的**：
>
> > \# rpm -qf /etc/passwd 
>
> **查询软件详细信息 ：**
>
> > [root@qfedu.com ~]# **rpm -qi 软件名称** 
> >
> > [root@qfedu.com ~]# rpm -qi gedit-2.28.4-3.el6.x86_64 
> >
> > -i information
>
> **软件卸载**
>
> > [root@qfedu.com ~]# rpm -e 软件名称 
> >
> > -e erase 
>
> --force 在安装的时候用(强制安装) 
>
> **--nodeps 在卸载的时候用(卸载的时候不检查依赖关系)** 

## 3.2 源代码包

> 1) 下载源码包，如tengine
>
> 2) 安装编译工具：
>
>    > yum -y install gcc make zlib-devel pcre pcre-devel openssl-devel
>
> 3) 解压源码包
>
> 4) **配置**
>
>    > ./configure –user==xxx –group=xxx –prefix=/usr/local/nginx	#–prefix指定安装路径
>
> 5) **编译**
>
>    >  **make**
>
> 6) **安装**
>
>    > make install

## 3.3 配置yum阿里源

> 配置yum源的步骤
> 1.可以移除默认的yum仓库，也就是删除 /etc/yum.repos.d/底下所有的.repo文件（踢出国外的yum源）
>
> 1.配置yum源，找到阿里云的官方镜像源地址 https://opsx.alibaba.com/mirror
> 2.下载centos7的yum源
> wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
>
> 3.清空旧的yum缓存
> yum clean all
> 4.生成新的yum仓库缓存（这个缓存来自于阿里云的yum仓库，便于加速软件下载）
> yum makecache
> 5.配置一个第三方的 额外仓库源 （epel源），这个源的作用是，如果阿里云源找不到这个软件，就在这里找
> wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

![](G:\images\Linux\image-20220904153813621.png)

> 查询yum源库
>
> **yum repolist**

# 4 任务计划

## 4.1 一次性调度执行at

> 语法格式：at <timespace>
>
> <timespace>**示例：**
>
> * now +5min
>
> * teatime tomorrow(teatime is 16:00)
>
> * noon +4days	第四天中午
>
> * 5pm august 3 2029
>
> * 4:00 2019-11-27
>
> 【注意】**Ctrl + D结束编辑任务**

![](G:\images\Linux\image-20220904160901396.png)

## 4.2 循环调度执行cron

> cron的概念和crontab是不可分割的。
>
> crontab是一个命令，常见于Unix和Linux的操作系统之中。
>
> **用于设置周期性被执行的指令**
>
> 该命令从标准输入设备读取指令，**并将其存放于/var/spool/cron/crontab文件中**，以供之后读取和执行。
>
> **命令输出以邮件形式发送给用户。**可以用mail命令查看。
>
> 
>
> 查看crond程序是否在执行状态：**systemctl status crond.service	或ps aux | grep crond**
>
> crond程序运行时计划任务执行的根本。一般默认开机自动执行。/usr/sbin/crond

> 1) 创建计划
>
>    **crontab -e	Edit jobs for the current user**
>
> 2) 查询计划
>
>    **crontab -l	List the jobs for the current user**
>
>    管理员可以使用-u username，去管理其它用户的计划任务
>
> 3) 删除计划
>
>    **crontab -r Remove all jobs for the current users**



> **编辑格式**
>
> **\* \* \* \* * command** 
>
> 星号分别表示分，时，日，月，周（24小时制）周（0-6）0表示周日
>
> **如1 3 1 * * command 表示每个月的1日3时1分执行后面的命令，*号表示任意月都执行**
>
> 如*/5 \* \* * command 表示每5分钟执行一次，***/5表示每隔5分钟**
>
> **如0 2 1,4,6 * * 表示每月的1,4,6日的2时0分执行**
>
> **如 0 2 5-9 * * 表示每月的5-9日2时0分执行**
>
> **如 0 2 * * 5 command 表示每月的两点执行** 
>
> **如 0 2 2 6 5 command 表示6月的2日，和所有周五2时零分执行**（周与月日模式都生效）

# 5 日志管理

> **rsyslog系统日志管理程序。**
>
> 常见的日志文件：
>
> > 1) /var/log/messages	//系统主日志文件
> >
> >    tail -f /var/log/messages	//动态查看日志文件的尾部
> >
> > 2) /var/log/secure	//认证、安全
> >
> > 3) /var/log/yum.log   //yum相关
> >
> > 4) /var/log/dmesg    //和系统启动相关
>
> **rsyslogd配置：**
>
> 启动程序：
>
> **systemctl start rsyslog.service** //默认开机自启
>
> 配置文件
>
> rpm -qc rsyslog
>
> **/etc/rsyslog.conf** **（rsylogd的主配置文件）**
>
> **/etc/sysconfig/rsyslog** （rsylogd相关文件，定义级别）
>
> **/etc/logrotate.d/syslog**（和日志办轮转（切割）相关）
>
> 主配置文件：
>
> RELUS，即规则，是一套生成日志，以及存储日志的策略
>
> 规则由设备+级别+存放位置组成。同一类程序的统称为设备
>
> 如：
>
> ```shell
> authpriv.*				/var/log/secure
> mail.*					-/var/log/maillog(-表示异步的方式记录，有时间差)
> cron.*					/var/log/cron
> *.info;mail.none;authpriv.none;cron.none                /var/log/messages # none表示排除这些设备产生的日志
> ```
>
> kill -1 rsyslog 或systemctl reload rsyslog重启程序读取配置文件
>
> **kill -1对于普通进程就是杀死进程，而对于一般的守护进程在收到这个信号时重新加载配置文件。**

## 5.1 日志轮转

> 日志	记录了程序运行时各种信息。
>
> 通过日志可以分析用户行为，记录运行轨迹，查找程序问题。
>
> 日志轮转就像飞机里的黑匣子，记录的信息再重要也只能记录最后一段时间发生的事件。
>
> 为了节省空间和整理方便，日志文件经常需要按时间或大小等维度分成多份，删除时间久远的日志文件。

**按配置进行轮转：**

> **配置文件：**
>
> > **主配置文件：/etc/logrotate.conf（决定每个日志文件如何轮转）**
> >
> > 子配置文件夹：/etc/logrotate**.d**/* （自定义配置，由用户自定义配置）
>
> > 全局设置（主配置文件，程序启动必须读取的文件）
> >
> > weekly	//轮转的周期，一周轮转一次
> >
> > minsize 1M	//最小达到1M才轮转，weekly and minsize
> >
> > rotate 4	//保留4份
> >
> > create	//轮转后创建新文件，默认操作
> >
> > #create 0644 root group1 //轮转时创建新文件，并设置权限和属主
> >
> > dateext	//使用日期作为日志备份后缀
> >
> > #compress	//是否压缩
> >
> > **include /etc/logrotate.d	//包含子配置文件**
> >
> > /var/log/wtmp{	//对某日志文件设置轮转的策略
> >
> > missingok	//丢失不提示
> >
> > }

# 6 网络管理

网络接口名称

> 传统上，Linux中的网络接口被枚举为eth0、eth1、eth2等。
>
> 网卡配置文件：
>
> **/etc/sysconfig/network-scripts/ifcfg-ens32**
>
> 命令行配置网卡：
>
> > vim /etc/sysconfig/network-scripts/ifcfg-ens32
> >
> > nmcli
>
> 图形配置网卡：
>
> > nmtui
> >
> > nm-connection-editor

## 6.1 配置网卡

```shell
vim /etc/sysconfig/network-scripts/ifcfg-ens33

ONBOOT=yes	# 是否开机自启动网卡
BOOTPROTOCOL=none	# 获取ip地址的方式（dhcp：自动分发，static/none：静态，手动）
IPADDR=192.168.142.131
NETMASK=255.255.255.0	# 子网掩码
GATEWAY=192.168.142.2	# 网关地址，默认路由
DNS1=192.168.142.2	# 域名解析服务器
```

> systemctl restart network	# 重启网络，更新配置

**查看ip：**

> **ip	a**
>
> **ip	route	#查看网关**
>
> **ip	neigh	#查看通过信的邻居**
>
> **ifconfig**

查看端口

> **ss -tnl**

## 6.2 主机名

> 查看主机名：hostname
>
> 临时修改：hostname LiQiaosheng.com
>
> 配置主机名：hostnamectl set-hostname LiQiaosheng.com
>
> 查看和配置主机名：vim /etc/hostname
>
> 重启生效：reboot

## 6.3 防火墙

关闭防火墙：

> systemctl	stop	firewalld	# 临时关闭，重启后自动打开
>
> systemctl	disable	firewalld	#永久性关闭
>
> systemctl	status	firewalld	#查看是否开机自启，disable字样表示开机不自启
>
> firewall简介：保护互联网对服务器的影响
>
> selinux简介：保护服务器内部程序（如ftp）对内部文件（/var/ftp）的访问。限制内部程序对内部文件的访问。

关闭另外一道防火墙：

> setenforce	0	#临时关闭
>
> vim /etc/sysconfig/selinux
>
> SELINUX=disabled	#永久性关闭
>
> getenforce	(查看selinuc状态，enfiring开启， permissive放行，disable关闭)

> **yum install -y lrzsz sysstat elinks wget net-tools bash-completion**

# 7 网络管理

## 7.1 划分VLAN

> 1) 创建VLAN(2~1024)，交换机默认所有接口都属于vlan1
>    1) 在交换机命令行，执行**enable**和**config t**命令进入特权配置模式。
>    2) 执行vlan 编号，创建VLAN，如vlan 10
>    3) 执行命令interface fastethernet0/1 或 int f0/1，进入对应的接口配置
>    4) 执行命令switch access vlan 10 或sw ac vl 10，将该接口配置到10号vlan
>    5) 不同vlan将的主机ping不通
>    6) 执行两次exit退出配置模式，执行**show vlan**查看交换机vlan划分情况

【注意】**VLAN的划分与主机没有关系，只是交换机的工作模式，主机发送和接收的帧是没有VLAN标签的。**

**VLAN标签是由交换机粘贴和去除的，带有标签的以太网帧在交换机间传递。VLAN标签只存在于trunk线路，在一端加上，在另一端交换机接收后去除。**

## 7.2 trunk

> **trunk（干道），用于不同交换机上相同vlan的通信。trunk可以通过打有所有标签编号的vlan。**
>
> 分别在两个交换机命令行运行(将两台交换机的3号端口连接线路配置成trunk，即把端口3设置成trunk模式)：
>
> int f0/3	
>
> switch mode trunk

**配置静态路由**

> enable
>
> config t
>
> ip route 192.168.2.0  255.255.255.0 f0/1

# 8 文件服务

## 8.1 FTP Server

> **FTP(File Transfer Protocol，文件传输协议)，是TCP/IP协议族中的协议之一。**
>
> 提供文件共享服务。
>
> 软件包：**vsftpd**
>
> 控制端口：command 21/tcp
>
> 数据端口：data 20/tcp
>
> ftp服务器搭建：
>
> 1) 安装vsftpd:
>
> > **yum -y install vsftpd**
>
> 2) 准备分发的文件
>
>    > touch /var/ftp/abc.txt
>    >
>    > 【注意】FTP服务器的主目录："/var/ftp/"，是FTP程序分享内容的本机目录。
>
> 3) 启动服务
>
>    > systemctl start vsftpd	 #启动服务
>    >
>    > systemctl enable vsftpd	#设置开机自启
>
> 4) 关闭防火墙
>
>    > systemctl stop firewalld
>    >
>    > systemctl enable firewalld
>    >
>    > setenforce 0
>    >
>    > vim /etc/selinux/config
>    >
>    > 修改内容为：SELINUX=disabled

> 1) 在客户端安装**lftp**可以在终端进行访问：
>
> > yum -y install lftp
> >
> > lftp 服务器的IP地址
>
> 2) 下载文件至当前目录下：
>
> > get abc.txt
> >
> > mirror pub	#下载文件夹
>
> 3) 上传文件
>
>    > cd upload
>    >
>    > put 5.txt

**启动上传功能**

> 0) 用于设定FTP服务器的功能开启或关闭的文件**/etc/vsftpd/vsftpd.conf**
>
>    > 1) 检查禁用匿名账户登录开启：
>    >
>    >    anonymous_enable=YES	#默认开启，即可匿名访问
>    >
>    > 2) 开启上传文件的能力
>    >
>    >    anon_upload_enable=YES	#启动上传能力
>    >
>    >    anon_mkdir_write_enable=YES	#启动创建目录的能力
>    >
>    > 3) 创建上传目录
>    >
>    >    mkdir /var/ftp/upload
>    >
>    >    chmod 777 /var/ftp/upload

## 8.2 wget

> 使用wget下载FTP服务器中的文件：
>
> wget ftp://192.168.142.132/abc.txt	#/abc.txt	中的**/**代表的是服务器的FTP主目录，**一般为/var/ftp/**

> **-O 路径	#表示下载到指定目录**

> wget **-m**  #（mirror）下载文件夹

## 8.3 NFS

> NFS的工作原理是使用客户端/服务器架构，由一个客户端程序和服务器程序组成。服务器程序向其他计算机提供对文件系统的访问，其过程称为输出。NFS客户端程序对共享文件系统进行访问时，把它们从NFS服务器中“输送”出来。文件通常以块为单位进行传输。其大小是8KB（虽然它可能会将操作分成更小尺寸的分片）。NFS传输协议用于服务器和客户机之间文件访问和共享的通信，从而使客户机远程地访问保存在存储设备上的数据。

> NFS：Network File System 网络文件系统，Linux/Unix系统之间共享文件的一种协议
>
> NFS的客户端主要为Linux
>
> 支持多节点同时挂载，以及并发写入
>
> 作用：
>
> > 提供文件共享服务，为Web Server配置集群中的后端存储
>
> 实验：
>
> 环境
>
> > nas 192.168.122.241
> >
> > web1 192.168.122.150
> >
> > web2 192.168.122.8
> >
> > web3 192.168.122.194
>
> **nas（存储端）：**
>
> 1) 安装NFS服务器
>
>    > yum -y install nfs-utils
>    >
>    > mkdir /webdata	#存储网站数据
>    >
>    > echo "nfs test….." > /webdata/index.html	#放置测试页面
>
> 2) 配置NFS服务器**/etc/exports**
>
>    > vim /etc/exports
>    >
>    > /webdata	192.168.122.0/24(rw)	#/webdata指的是发布资源的目录，192.168.122.0/24指的是允许访问NFS的客户机，(rw)表示可读写，ro表示只读
>
> 3) 启动NFS
>
>    > systemctl start nfs-server
>    >
>    > systemctl enable nfs-server
>
> 4) 检查
>
>    > exportfs -v
>
> **客户端：**
>
> 1) 安装NFS客户端
>
>    > yum -y install nfs-utils
>    >
>    > yum -y install httpd
>
> 2) 查看存储端共享
>
>    > showmount -e 192.168.122.241	# 查询NFS服务器可用目录
>    >
>    > Export list for 192.168.142.241
>    >
>    > /webdata 192.168.142.0/24
>
> 3) 手动挂载
>
>    > **mount -t nfs 192.168.142.133:/webdata	/var/www/html/**  # /var/www/html/为httpd服务的主目录
>    >
>    > umount /var/var/www/html/	#取消挂载
>
> 4) 查看挂载
>
>    > cat /var/www/html/index.html
>    >
>    > nfs test…..
>    >
>    > df

## 8.4 ssh

> 远程连接
>
> ssh @root 10.0.0.204
>
> 1) 查询是否安装ssh
>
>    > rpm -qa | grep ssh
>    >
>    > yum install openssh-server #安装ssh
>
> 2) 启动服务（默认自己启动）
>
>    > systemctl start sshd
>    >
>    > netstat -antp | grep sshd	#22号端口提供者ssh服务
>    >
>    > systemctl enable sshd	#开机启动
>
> 3) 使用ssh远程管理
>
>    > **ssh 账户名@IP地址**

# 9 Apache

> Apache官网：www.apache.org
>
> 软件包名：httpd
>
> 服务端口：80/tcp(http)	443/tcp(https)
>
> 配置文件：/etc/httpd/conf/httpd.conf
>
> 子配置文件：/etc/httpd/conf.d/*.conf
>
> 主目录：/var/www/html/	#网站源代码默认位置

安装Apache

> yum -y install httpd	#安装
>
> systemctl start httpd	#启动
>
> systemctl status httpd	#查看服务状态
>
> systemctl enable httpd	#设置开机自启
>
> systemctl stop firewalld	#关闭防火墙
>
> httpd -v	#查看版本
>
> setenforce 0

## 9.1 虚拟主机

### 9.1.1 服务端配置

> 作用：在一台物理服务器上运行多个网站

> 配置虚拟主机目标：在一台服务器上实现两个网站的架设
>
> www.a.org	# /var/www/html/a.org
>
> www.b.org
>
> **www.a.org站点设置：**
>
> 1) 准备网站源码（网页）目录
>
>    > mkdir /var/www/html/a.org
>    >
>    > vim /var/www/html/a.org/index.html
>
> 2) 创建a.org的网站配置文件
>
>    ```html
>    # vim /etc/httpd/conf.d/a.org.conf
>    
>    <VirtualHost *:80>
>    
>    ServerName www.a.org # 服务器名字
>    
>    DocumentRoot /var/www/html/a.org #网站的根目录
>    
>    </VirtualHost>
>    ```
>
> 3) 检测配置文件语法，重启服务
>
>    > httpd -t
>    >
>    > systemctl restart httpd
>
> **www.b.org站点设置：**
>
> 1) 准备网站源代码目录
>
>    > mkdir /b.org
>    >
>    > vim /b.org/index.html
>
> 2) 创建b.org的网站配置文件
>
>    > vim /etc/httpd/conf.d/b.org.conf
>    >
>    > ```html
>    > <VirtualHost *:80>
>    > ServerName www.b.org
>    > DocumentRoot /b.org
>    > </VirtualHost>
>    > 
>    > <Directory "/b.org"> # /var/www/html/以外的目录需要以下内容进行授权
>    > Require all granted
>    > </Directory>
>    > ```
>
> 3) 检测配置文件语法，重启服务

### 9.1.2 客户端配置

> Linux客户端域名解析
>
> > vim /etc/hosts
> >
> > 10.0.128.118	www.a.rg	#填写web服务器的IP
> >
> > 10.0.128.118	www.b.org	#填写web服务器的IP
>
> 字符测试：
>
> > yum install -y elinks
> >
> > elinks http://www.a.org

## 9.2 部署论坛系统discuz

> discuz：开源论坛代码服务社区
>
> 基础环境：关闭防火墙
>
> 安装LAMP（Linux Apache MySql Php）：**yum -y install httpd mariadb-server mariadb(mysql) php php-mysql gd php-gd**
>
> 安装Discuz：
>
> 1) 导入discuz网站源码
>
>    > wget http://download.comsenz.com/DiscuzX/2.5/Discuz_X2.5_SC_UTF8.zip
>    >
>    > mkdir -p /webroot/discuz
>    >
>    > yum install -y unzip
>    >
>    > unzip Discuz_X3.4_SC_UTF8.zip
>    >
>    > chown -R apache.apache /webroot/discuz/
>
> 2) Apache配置虚拟主机
>
>    > vim /etc/httpd/conf.d/discuz.conf
>    >
>    > ```html
>    > <VirtualHost *:80>
>    > ServerName www.discuz.com
>    > DocumentRoot /webroot/discuz
>    > </VirtualHost>
>    > 
>    > <Directory "/webroot/discuz">
>    > Require all granted
>    > </Directory>
>    > ```
>    >
>    > systemctl restart httpd
>
> 3) 准备数据库
>
>    > mysql #进入mysql交互
>    >
>    > ```mysql
>    > create database discuz;
>    > 
>    > show databases;
>    > 
>    > ```

> 客户端测试：
>
> 1) 添加域名解析
>
>    > vim /etc/hosts
>    >
>    > 192.168.122.161 www.discuz.com discuz.com
>
> 2) 使用浏览器访问www.discuz.com，并设置数据库、站长等信息。

# 10 域名服务

> hosts文件：
>
> > 作用：实现域名解析，主要为本地主机名、集群节点提供快速解析
> >
> > 数据库：平面式结构，集中式数据库
> >
> > 缺点：不便于查询，更新
> >
> > Windows：C:\Windows\System32\drivers\etc\hosts
> >
> > Linux：/etc/hosts
>
> DNS：
>
> > DNS(Domain Name System，域名系统)
> >
> > 作用：实现域名解析
> >
> > 主机名.二级域名.顶级域名.（根域）
> >
> > 如：
> >
> > baidu.com
> >
> > www.baidu.com.
> >
> > www.icbc.com.cn

![](G:\images\1844px-DNS_schema.svg.png)

## 10.1 ISP(阿里)域名申请及解析

> 1) 准备阿里云服务器，启动网站功能
>
>    > 1) 搜索阿里云
>    > 2) 登录阿里云
>    > 3) 进入控制台
>    > 4) 选择云服务器(ECS)
>    > 5) 选择服务器配置
>    > 6) 选择带宽
>    > 7) 设置管理员密码
>    > 8) 设置标签（可有可无）
>    > 9) 确认订单
>    > 10) 登录ECS，启动网站服务
>
> 2) 通过控制台找到域名注册服务
>
> 3) 输入新域名，根据需要选购合适的域名
>
> 4) 选择购买并支付
>
>    ![](G:\images\image-20221004145833933.png)
>
> 5) 返回域名控制面板（解析），绑定IP地址
>
> 6) 客户端测试
>
> 7) 域名备案

## 10.2 DNS解析流程

> 例如客户端解析 www.126.com
>
> 1) 客户端查询自己的缓存（包含hosts中的记录），如果没有则将查询发送**/etc/resolv.conf**中的DNS服务器
> 2) 如果本地DNS服务器对于请求的信息具有权威性，会将（权威答案）发送到客户端。
> 3) 否则（不具有权威性），如果DNS服务器在其缓存中有请求信息，则将（非权威答案）发送到客户端
> 4) 如果缓存中没有查询到信息，DNS服务器将搜索权威DNS服务器以查找信息
>    1) 从根区域开始，按照DNS层次结构向下搜索，至至对于信息具有权威的名称服务器，为客户端答案，DNS将信息传递给客户端，并在自己的缓存中保留一个副本，以备以后查找
>    2) 转发到其他的DNS服务器

![](G:\images\image-20221004153720462.png)
