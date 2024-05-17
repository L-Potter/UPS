## Network UPS Tools User Manual
https://networkupstools.org/docs/user-manual.chunked/

### qnap 
>ls /etc/config/ups
>>ups.conf  upsd.conf  upsdrv.map  upsd.users  upsmon.conf

* upsd.users
https://networkupstools.org/docs/man/upsd.users.html  
  Administrative user definitions for NUT upsd  

  Administrative commands such as setting variables and the instant commands are powerful, and access to them needs to be `restricted`.

  This file defines who may access them, and what is available.

  Each user gets its `own section`. The fields in that section set the parameters associated with that `user’s privileges`.

  `The section` begins with the `name of the user` in brackets, and continues until the next user name in brackets or EOF. These users are `independent` of /etc/passwd.


   中譯: 此user/pwd是用於限制upsd process使用權限的並非使用者帳密
```
[admin]
        password = mypass
        actions = set
        actions = fsd
        instcmds = all
[pfy]
        password = duh
        instcmds = test.panel.start
        instcmds = test.panel.stop
[upswired]
        password = blah
        upsmon primary
[observer]
        password = abcd
        upsmon secondary

---
password
Set the password for this user.

actions
Allow the user to do certain things with upsd. To specify multiple actions, use multiple instances of the actions field. Valid actions are:

SET
    change the value of certain variables in the UPS

    FSD
    set the forced shutdown flag in the UPS. This is equivalent to an "on battery + low battery" situation for the purposes of monitoring.

    The list of actions is expected to grow in the future.

instcmds
    Let a user initiate specific instant commands. Use "ALL" to grant all commands automatically. To specify multiple commands, use multiple instances of the instcmds field. For the full list of what your UPS supports, use "upscmd -l".

    The cmdvartab file supplied with the distribution contains a list of most of the known command names.

upsmon
    Add the necessary actions for a upsmon process to work. This is either set to "primary" or "secondary".

    Do not attempt to assign actions to upsmon by hand, as you may miss something important. This method of designating a "upsmon user" was created so internal capabilities could be changed later on without breaking existing installations.
```

* upscmd
```
Network UPS Tools upscmd 2.7.4

usage: upscmd [-h]
       upscmd [-l <ups>]
       upscmd [-u <username>] [-p <password>] <ups> <command> [<value>]

Administration program to initiate instant commands on UPS hardware.

  -h		display this help text
  -l <ups>	show available commands on UPS <ups>
  -u <username>	set username for command authentication
  -p <password>	set password for command authentication

  <ups>		UPS identifier - <upsname>[@<hostname>[:<port>]]
  <command>	Valid instant command - test.panel.start, etc.
  [<value>]	Additional data for command - number of seconds, etc.
```



* ups.conf
https://networkupstools.org/docs/man/ups.conf.html

ups.conf - UPS definitions for Network UPS Tools
* * driver
Required. This specifies which program will be monitoring this UPS. You need to specify the one that is compatible with your hardware. See nutupsdrv(8) for more information on drivers in general and pointers to the man pages of specific drivers.

* * port
Required. This is the `serial port` where the UPS is connected. On a Linux system, the first serial port usually is /dev/ttyS0. On FreeBSD and similar systems, it probably will be /dev/cuaa0. On Windows, the first serial port will be "\\\\.\\COM1" (note the escaped slashes).

* * desc
Optional. This allows you to set a brief description that upsd will provide to clients that ask for a list of connected equipment.


[qnapups]
driver = usbhid-ups
port = /dev/ttyS1
desc = "Workstation"
pollinterval=2

[qnapups] 是UPS名稱，可以依照需求取別的名稱，例如 [cyberpower1] 之類的。driver是連接的驅動程式，通常USB的UPS就寫上usbhid-ups就好，可以參考下方註解1。port的部分輸入auto即可。desc跟設定沒關係，不寫也可以。

[註解1]
lsusb
dmesg | grep -i "APC UPS NAME"
Hardware compatibility list: driver(using ups.conf)
https://networkupstools.org/stable-hcl.html

* Optional
* * maxretry 重試次數
Optional. Specify the number of attempts to start the driver(s), in case of failure, before giving up. A delay of retrydelay is inserted between each attempt. Caution should be taken when using this option, since it can impact the time taken by your system to start.

  The default is 1 attempt.

* * pollinterval 輪詢時間差
    Optional. The status of the UPS will be refreshed after a maximum delay which is controlled by this setting. This is `normally 2 seconds.` This setting may be useful if the driver is creating too much of a load on your monitoring system or network.

* * retrydelay 重試之間時間差
    Optional. Specify the delay between each restart attempt of the driver(s), as specified by maxretry. Caution should be taken when using this option, since it can impact the time taken by your system to start.

    The default is 5 seconds.






第二個帳號設定除了master之外還可以設定為slave，差別是電源不足的時候slave會先關機，master之後才關機。有些新版本的NUT可能會改叫做primary、secondary，理由應該跟Git的主要支線從master改成main一樣。

以下是NAS upsd.conf但是作為nut-client不用修改此處
```
#
# ACL <name> <ipblock>
# ACL myhost 10.0.0.1/32
#
# ACCEPT <aclname> [<aclname>...]
# REJECT <aclname> [<aclname>...]
#
# Define lists of hosts or networks with ACL definitions.
#
# ACCEPT and REJECT use ACL definitions to control whether a host is
# allowed to connect to upsd.
#
# This default configuration only gives access to localhost.  To allow
# other hosts or networks to connect, see the documentation and change
# these lines.

ACL all 0.0.0.0/0
ACL localhost 127.0.0.1/32

ACCEPT localhost
REJECT all

MAXAGE          20

# =======================================================================
# MAXAGE <seconds>
# MAXAGE 15
#
# This defaults to 15 seconds.  After a UPS driver has stopped updating
# the data for this many seconds, upsd marks it stale and stops making
# that information available to clients.  After all, the only thing worse
# than no data is bad data.
#
# You should only use this if your driver has difficulties keeping
# the data fresh within the normal 15 second interval.  Watch the syslog
# for notifications from upsd about staleness.
```


```
# Network UPS Tools: example upsmon configuration
#
# This file contains passwords, so keep it secure.

# --------------------------------------------------------------------------
# RUN_AS_USER <userid>
#
# By default, upsmon splits into two processes.  One stays as root and
# waits to run the SHUTDOWNCMD.  The other one switches to another userid
# and does everything else.
默认情况下，upsmon 分成两个进程。一个保持为 root，并等待运行 SHUTDOWNCMD。另一个切换到另一个用户 ID 并执行其他操作。
#
# The default nonprivileged user is set at compile-time with
#       'configure --with-user=...'.
#
# You can override it with '-u <user>' when starting upsmon, or just
# define it here for convenience.
#
# Note: if you plan to use the reload feature, this file (upsmon.conf)
# must be readable by this user!  Since it contains passwords, DO NOT
# make it world-readable.  Also, do not make it writable by the upsmon
# user, since it creates an opportunity for an attack by changing the
# SHUTDOWNCMD to something malicious.
默认的非特权用户在编译时设置为 'configure --with-user=...'。
您可以在启动 upsmon 时使用 '-u <user>' 来覆盖它，或者只是在这里定义它以方便起见。
注意：如果您计划使用重新加载功能，则此文件（upsmon.conf）必须由此用户可读取！由于它包含密码，不要使它对所有人可读取。此外，不要使其对 upsmon 用户可写入，因为这会为通过更改 SHUTDOWNCMD 到某些恶意内容的攻击提供机会。
#
# For best results, you should create a new normal user like "nutmon",
# and make it a member of a "nut" group or similar.  Then specify it
# here and grant read access to the upsmon.conf for that group.
#
# This user should not have write access to upsmon.conf.
#
# RUN_AS_USER nutmon
为了获得最佳效果，您应该创建一个新的普通用户，如 "nutmon"，并将其设置为 "nut" 组或类似组的成员。然后在这里指定它，并为该组授予对 upsmon.conf 的读取权限。
此用户不应具有对 upsmon.conf 的写入权限。

RUN_AS_USER admin
-rw-r--r--  1 admin administrators 11971 2023-11-18 20:58 upsmon.conf

# --------------------------------------------------------------------------
# MONITOR <system> <powervalue> <username> <password> ("master"|"slave")
#
# List systems you want to monitor.  Not all of these may supply power
# to the system running upsmon, but if you want to watch it, it has to
# be in this section.
列出您想要监视的系统。并不是所有这些系统都会向运行 upsmon 的系统提供电源，但如果您想观察它，它必须在此部分中。
#
# You must have at least one of these declared.
#
# <system> is a UPS identifier in the form <upsname>@<hostname>[:<port>]
# like ups@localhost, su700@mybox, etc.
<system> 是 UPS 标识符，格式为 <upsname>@<hostname>[:<port>]，例如 ups@localhost、su700@mybox 等。
#
# Examples:
#
#  - "su700@mybox" means a UPS called "su700" on a system called "mybox"
#
#  - "fenton@bigbox:5678" is a UPS called "fenton" on a system called
#    "bigbox" which runs upsd on port "5678".
#
# The UPS names like "su700" and "fenton" are set in your ``ups.conf``
# in [brackets] which identify a section for a particular driver.
UPS 名称（如 "su700" 和 "fenton"）在您的 ups.conf 中以 [括号] 的形式设置，用于标识特定驱动程序的部分。
e.g. ups.conf
[qnapups]
driver = usbhid-ups
port = /dev/ttyS1
desc = "Workstation"
pollinterval=2
#
# If the ups.conf on host "doghouse" has a section called "snoopy", the
# identifier for it would be "snoopy@doghouse".
#
# <powervalue> is an integer - the number of power supplies that this UPS
# feeds on this system.  Most computers only have one power supply, so this
# is normally set to 1.  You need a pretty big or special box to have any
# other value here.
#
# You can also set this to 0 for a system that doesn't supply any power,
# but you still want to monitor.  Use this when you want to hear about
# changes for a given UPS without shutting down when it goes critical,
# unless <powervalue> is 0.
<powervalue> 是一个整数，表示此 UPS 在此系统上供电的电源数量。大多数计算机只有一个电源供应器，因此通常设置为 1。除非 <powervalue> 为 0，否则不需要执行关机，但您仍然希望监视给定 UPS 的更改时，可以将其设置为 0。
#
# <username> and <password> must match an entry in ``that system's
# upsd.users.``  If your username is "monmaster" and your password is
# "blah", the upsd.users would look like this:
<username> 和 <password> 必须与该系统的 upsd.users 中的条目匹配。如果您的用户名是 "monmaster"，密码是 "blah"，则 upsd.users 将如下所示：

""" 重點是system's upsd.users 表示是nut server為帳號密碼而不是client的 """

#
#       [monmaster]
#               password  = blah
#               allowfrom =     (whatever applies to this host)
#               upsmon master   (or slave)
#
# "master" means this system will shutdown last, allowing the slaves
# time to shutdown first.
#
# "slave" means this system shuts down immediately when power goes critical.
"master" 表示此系统将在最后关闭，以便从属系统有足够的时间先关闭。
"slave" 表示此系统在电源临界时立即关闭。
#
# Examples:
#
# MONITOR myups@bigserver 1 monmaster blah master
# MONITOR su700@server.example.com 1 upsmon secretpass slave

MONITOR qnapups@localhost 1 admin 123456 master

以上都是localhost & master -> ups's usb pluged on NAS
upsd.users 
[admin]
                password = 123456
                allowfrom = localhost
                actions = SET
                instcmds = ALL
                upsmon master           # or upsmon slave
```


```
# --------------------------------------------------------------------------
# MINSUPPLIES <num>
#
# Give the number of power supplies that must be receiving power to keep
# this system running.  Most systems have one power supply, so you would
# put "1" in this field.
#
# Obviously you have to put the redundant supplies on different UPS circuits
# for this to make sense!  See big-servers.txt in the docs subdirectory
# for more information and ideas on how to use this feature.

MINSUPPLIES 1
MINSUPPLIES <num>
给出必须接收电源的电源数量以保持此系统运行。大多数系统只有一个电源供应器，因此在此字段中放置 "1"。
显然，您必须将冗余电源放置在不同的 UPS 电路上，以使此功能有意义！有关更多信息和如何使用此功能的想法，请参阅文档子目录中的 big-servers.txt。
```

```
# --------------------------------------------------------------------------
# SHUTDOWNCMD "<command>"
#
# upsmon runs this command when the system needs to be brought down.
#
# This should work just about everywhere ... if it doesn't, well, change it.
当系统需要关闭时，upsmon 运行此命令。
这应该在几乎所有地方都能正常工作……如果不行，那就改变它。

SHUTDOWNCMD "/sbin/shutdown -h +0"
```

```
# POLLFREQ <n>
#
# Polling frequency for normal activities, measured in seconds.
#
# Adjust this to keep upsmon from flooding your network, but don't make
# it too high or it may miss certain short-lived power events.

POLLFREQ 5

正常活动的轮询频率，以秒为单位。
调整此参数以防止 upsmon 过度占用网络，但不要设置得太高，否则可能会错过某些短暂的电源事件。
```

```
# POLLFREQALERT <n>
#
# Polling frequency in seconds while UPS on battery.
#
# You can make this number lower than POLLFREQ, which will make updates
# faster when any UPS is running on battery.  This is a good way to tune
# network load if you have a lot of these things running.
#
# The default is 5 seconds for both this and POLLFREQ.

POLLFREQALERT 5
POLLFREQALERT <n>
当 UPS 处于电池供电状态时的轮询频率，以秒为单位。
您可以将此数字设置为低于 POLLFREQ，这将使在任何 UPS 处于电池供电状态时更新更快。 如果您有很多这样的设备在运行，则可以通过这种方式来调整网络负载。
默认值为 5 秒，POLLFREQALERT 和 POLLFREQ 均为此值。
```


```
# FINALDELAY - last sleep interval before shutting down the system
#
# On a master, upsmon will wait this long after sending the NOTIFY_SHUTDOWN
# before executing your SHUTDOWNCMD.  If you need to do something in between
# those events, increase this number.  Remember, at this point your UPS is
# almost depleted, so don't make this too high.
#
# Alternatively, you can set this very low so you don't wait around when
# it's time to shut down.  Some UPSes don't give much warning for low
# battery and will require a value of 0 here for a safe shutdown.
#
# Note: If FINALDELAY on the slave is greater than HOSTSYNC on the master,
# the master will give up waiting for the slave to disconnect.

FINALDELAY 5
FINALDELAY - 在关闭系统之前的最后睡眠间隔
在主系统上，upsmon 在发送 NOTIFY_SHUTDOWN 后会等待这么长时间，然后才执行您的 SHUTDOWNCMD。 如果您需要在这两个事件之间执行某些操作，请增加此数字。 请记住，此时您的 UPS 几乎已耗尽，因此不要将其设置得太高。
或者，您可以将此值设置得非常低，以便在需要关闭时不要等待。 一些 UPS 对低电池警告的时间不是很长，可能需要在此处设置为 0 以进行安全关闭。
注意：如果从机的 FINALDELAY 大于主机的 HOSTSYNC，则主机将放弃等待从机断开连接。
FINALDELAY 5
是的，你可以通过修改 FINALDELAY 参数来控制从属 nut（Network UPS Tools）监控到电源故障后等待多长时间后再执行关机操作。默认情况下，该参数设置为5秒。你可以根据需要将其设置为更长的时间，以确保在执行关机操作之前充分等待其他操作完成。
```


QNAP:
sudo -i   # -u root 
/etc/init.d/usb_ups.sh restart

/etc/nut/ups.conf
[ups1]
	driver = usbhid-ups
	port = auto
	desc = "Cyber Power System, Inc. CP1500 AVR UPS"

/etc/nut/nut.conf
MODE=netserver

/etc/nut/upsd.conf
LISTEN 0.0.0.0 3493

/etc/nut/upsd.users
[admin]
        password = password
        actions = SET
        instcmds = ALL

[upsmst]
        password = password
        upsmon master

[upscln]
        password = password
        upsmon slave

[upsuser]
        password = password
        actions = SET
        instcmds = ALL


nutserver ip 192.168.0.100

client 
upsd.users
[upsuser]
	password = password
	actions = SET
	instcmds = ALL
	upsmon slave

ups.conf
[qnapups]
	driver = usbhid-ups
	port = 192.168.0.100:3551
	desc = "ups for qnap"

qnap
upsdrvctl --version
Network UPS Tools - UPS driver controller 2.7.4