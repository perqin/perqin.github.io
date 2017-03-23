---
title: 在 CentOS 6.8 上安装 Oracle Database 11g Release 2
date: 2017-03-23 23:43:12
tags: [oracle-database, centos]
---

Oracle Database 的安装一般需要图形化界面，因此这里我将使用 Response File 安装的方法记录下来以供参考。

## 获取 Response File

要通过静默安装的方式安装，我们需要一个 Response File，可以看作这个文件代替我们回答了图形化安装界面里的每个步骤的选项。我通过本地的虚拟机获得了该文件，也可以在安装目录里的`response/db_install.rsp`中看到默认的配置，这里我的配置如下（删除了注释）：

```
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=localhost
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/opt/oracle/oraInventory
SELECTED_LANGUAGES=en
ORACLE_HOME=/opt/oracle/oracle11g/product/11.2.0/dbhome_1
ORACLE_BASE=/opt/oracle/oracle11g
oracle.install.db.InstallEdition=EE
oracle.install.db.isCustomInstall=false
oracle.install.db.customComponents=
oracle.install.db.DBA_GROUP=dba
# Use wheel group which can issue sudo command
oracle.install.db.OPER_GROUP=wheel
oracle.install.db.CLUSTER_NODES=
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.globalDBName=
oracle.install.db.config.starterdb.SID=
oracle.install.db.config.starterdb.characterSet=
oracle.install.db.config.starterdb.memoryLimit=
oracle.install.db.config.starterdb.memoryOption=false
oracle.install.db.config.starterdb.installExampleSchemas=false
oracle.install.db.config.starterdb.enableSecuritySettings=true
oracle.install.db.config.starterdb.password.ALL=
oracle.install.db.config.starterdb.password.SYS=
oracle.install.db.config.starterdb.password.SYSTEM=
oracle.install.db.config.starterdb.password.SYSMAN=
oracle.install.db.config.starterdb.password.DBSNMP=
oracle.install.db.config.starterdb.control=DB_CONTROL
oracle.install.db.config.starterdb.gridcontrol.gridControlServiceURL=
oracle.install.db.config.starterdb.dbcontrol.enableEmailNotification=false
oracle.install.db.config.starterdb.dbcontrol.emailAddress=
oracle.install.db.config.starterdb.dbcontrol.SMTPServer=
oracle.install.db.config.starterdb.automatedBackup.enable=false
oracle.install.db.config.starterdb.automatedBackup.osuid=
oracle.install.db.config.starterdb.automatedBackup.ospwd=
oracle.install.db.config.starterdb.storageType=
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=
oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=
oracle.install.db.config.asm.diskGroup=
oracle.install.db.config.asm.ASMSNMPPassword=
MYORACLESUPPORT_USERNAME=
MYORACLESUPPORT_PASSWORD=
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false
# Must be true to avoid PROXY related error
DECLINE_SECURITY_UPDATES=true
PROXY_HOST=
PROXY_PORT=
```

其中需要注意的配置项已经在上面用注释说明。

## 配置依赖

Oracle Database 安装之前不仅要安装依赖，还需要调整各种内核参数。幸运的是，Oracle 已经为我们准备了一键配置包。

首先添加源：

```
$ wget http://public-yum.oracle.com/public-yum-ol6.repo
# mv public-yum-ol6.repo /etc/yum.repos.d/
```

然后获取公钥：

```
# wget http://public-yum.oracle.com/RPM-GPG-KEY-oracle-ol6 -O /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
```

接下来我们就可以安装依赖包了：

```
# yum install oracle-rdbms-server-11gR2-preinstall elfutils-libelf-devel unixODBC unixODBC-devel
```

## 准备安装用户和目录

安装好之后，Oracle 已经为我们添加了一个用户`oracle`和其用户组`oinstall`，我们接下来需要通过这个帐号安装（使用`root`运行安装程序会被拒绝），所以我们要先更改其密码，并事先创建好安装目录并配置好权限（如果选择安装到其他位置，前面的 Response File 需要相应地修改）：

```
# passwd oracle
# usermod -aG wheel oracle
# mkdir -p /opt/oracle
# chown oracle:oinstall /opt/oracle
```

## 准备安装包

以下步骤中美元符号开头的命令均使用`oracle`用户执行。

从官方网站下载了两个 zip 之后通过`unzip`命令解压，就能在目录中看到`database`目录，里面有安装所需一切文件：

```
$ unzip linux.x64_11gR2_database_1of2.zip
$ unzip linux.x64_11gR2_database_2of2.zip
```

##  增大 swap 分区

如果你的 CentOS 的 swap 分区没有启用或者不足 8G，安装将会失败。通过以下配置可以添加新的 swap 分区。创建分区文件也可以用`dd`命令，这里不详述：

```
# fallocate -l 8G /swapfile
# chmod 600 /swapfile
# mkswap /swapfile
# swapon /swapfile
```

然后编辑文件`/etc/sysctl.conf`，修改下面的一行如下：

```
vm.swappiness = 10
```

保存之后执行：

```
# sysctl -p
```

为了能够在服务器启动后自动挂载，可以在`/etc/fstab`中追加下面一行：

```
/swapfile            none                    swap    sw              0 0
```

## 开始安装

安装的时候只要用`oracle`用户执行一行命令。部分 log 会直接刷到标准输出，其中说明了日志的路径，只要静静等待安装完成时标准输出出现执行相关脚本的字眼即可。安装命令如下：

```
$ ./database/runInstaller -silent -noconfig -ignoreSysPrereqs -ignorePrereq -responseFile /home/oracle/db.rsp
```

注意修改上面的 Response File 的路径。

安装完成之后，**使用`root`用户**执行两个脚本：

```
# /opt/oracle/oraInventory/orainstRoot.sh
# /opt/oracle/oracle11g/product/11.2.0/dbhome_1/root.sh
```

## 配置环境变量

在`/etc/profile`中添加以下环境变量配置：

```
export ORACLE_BASE=/opt/oracle/oracle11g
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
export ORACLE_SID=orcl
export ORACLE_UNQNAME=orcl
export NLS_LANG=.AL32UTF8
export PATH=$PATH:$ORACLE_HOME/bin/:$ORACLE_HOME/lib64
```

## 完成

执行`. /etc/profile`或重新登入之后，就可以启动监听器了：

```
$ lsnrctl start
```

可以通过下面的命令验证 Oracle Database 已经可以使用：

```
$ sqlplus / as sysdba
```

由于之前配置好了`oracle`用户，所以可以直接登录而不询问密码。