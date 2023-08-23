---
layout: article
tags: Linux
title: synce dpll
mathjax: true
key: Linux
---

[Reference](https://www.h3c.com/cn/Service/Document_Software/Document_Center/Home/Switches/00-Public/Learn_Technologies/White_Paper/(SyncE)_WP-6W100/)
{:.info}

[Reference](https://www.embedded.com/an-introduction-to-synchronized-ethernet/)
{:.info}

[Reference](https://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-xr-software/217579-configure-ptp-and-synce-basics-with-cisc.html#anc7)
{:.info}

## 术语
```
同步以太网（SyncE，Synchronous Ethernet）
Primary Reference Clock (PRC)
Primary Reference Source (PRS)
Synchronization Supply Unit (SSU)
Building Integrated Timing Supply (BITS)
SDH Equipment Clock (SEC)
SONET Minimum Clock (SMC)
```

## 产生背景
```
在通信网络中，许多业务的正常运行都要求网络时间同步。时间同步包括频率和相位两方面的同步。通过时间同步可以使整个网络各设备之间的频率和相位差保持在合理的误差范围内。
同步以太网（SyncE，Synchronous Ethernet）是一种基于物理层码流携带和恢复频率信息的同步技术，能实现网络设备间高精度的频率同步，满足无线接入业务对频率同步的要求。
通常将SyncE和PTP技术配合使用，同时满足频率和相位的高精度要求，实现纳秒级的时间同步。
```

## 时间同步
```
频率同步也称为时钟同步。频率同步指两个信号的变化频率相同或保持固定的比例，信号之间保持恒定的相位差。
相位同步是指信号之间的频率和相位都保持一致，即信号之间相位差恒定为零。

SyncE只能实现频率同步，SyncE和PTP配合可实现时间同步。
```

## 时钟源类型
```
为设备提供时钟信号的设备叫做时钟源。根据时钟信号的来源不同，SyncE支持的时钟源包括：

·     BITS（Building Integrated Timing Supply System，通信楼综合定时供给系统）时钟源：时钟信号由专门的BITS时钟设备产生。设备通过专用接口（BITS接口）收发BITS时钟信号。
·     线路时钟源：由上游设备提供的、本设备的时钟监控模块从以太线路码流中提取的时钟信号，即开启SyncE功能的接口传递的时钟信号。线路时钟源精度比BITS时钟源低。
·     PTP时钟源：本设备从PTP协议报文中提取的时钟信号。PTP协议时钟源的精度比BITS时钟源低。
·     本地时钟源：本设备内部的晶体振荡器产生的38.88 MHz时钟信号，通常本地时钟源精度最低。
```

## 时钟源选择
```
当设备连接了多种时钟源，有多路时钟信号输入设备时，可以通过手动模式或自动模式选择一路优先级最高的时钟信号作为最优时钟（也称为参考源）。
·     手动模式：用户手工指定最优时钟。如果最优时钟的同步信号丢失，设备不会自动采用其他时钟源的时钟信号，而是使用设备上存储的已丢失的最优时钟的时钟参数继续运行。
·     自动模式：系统自动选择最优时钟（也称为自动选源）。如果最优时钟的同步信号丢失，设备会自动选择新的最优时钟，并和新的最优时钟保持同步。

自动选源参考因素
影响设备自动选择最优时钟的参考因素包括SSM（Synchronization Status Message，同步状态信息）级别和优先级。

1. SSM级别
SSM是ITU-T G.781在SDH（Synchronous Digital Hierarchy，同步数字系列）网络中定义的标识时钟源质量等级（QL，Quality Level）的一组状态信息。
SyncE也使用SSM级别来表示时钟源的好坏，并把SSM级别称为QL级别，本文中统称为SSM级别。

SyncE中支持的SSM级别按照其同步质量由高到低依次为：
·     PRC（Primary Reference Clock，基准参考时钟）级别：精度符合G.811协议要求。通常，BITS跟踪GPS或者北斗等卫星源后输出的时钟信号的质量等级为PRC。
·     SSU-A（primary level SSU，转接局时钟）级别：精度符合G.812中转接节点要求。通常，配置了铷钟的BITS在丢失GPS等卫星源后进入保持或者自由振荡时输出的时钟信号的质量等级为SSU-A。
·     SSU-B（second level SSU，本地局时钟）级别：精度符合G.812中本地节点要求。通常，配置了晶体钟的BITS在丢失GPS等卫星源后进入保持或者自由振荡时输出的时钟信号的质量等级为SSU-B。
·     SEC（SDH Equipment Clock，SDH设备时钟）/EEC（Ethernet Equipment Clock，以太网设备时钟）级别：精度符合G.813协议要求。
              通常，承载网设备（SDH设备SEC或者同步以太设备EEC）丢失参考源后进入保持或者自由振荡时输出的时钟信号的质量等级为SEC/EEC。
·     DNU（Do Not Use for synchronization，不应用作同步）级别：精度不符合时钟同步的要求。该质量等级的时钟源不可以作为参考源。
·     UNK级别：同步质量未知。


SSM级别对于自动选择最优时钟、时钟环路避免具有重要意义。设备间通过周期发送ESMC（Ethernet Synchronous Message Channel，以太网同步消息通道）报文传递时钟源的SSM级别。
ESMC报文有两种发送方式：
·     周期发送：设备在开启了SyncE功能的接口上每秒发送一次ESMC information报文，用于告知邻居设备本设备提供的时钟信号的SSM级别。
·     事件触发发送：当本设备选择的最优时钟变化时，立即发送携带新的最优时钟的SSM级别的ESMC event报文，以便尽快通知下游设备本设备提供的时钟信号的SSM级别发生变换。
                    与此同时，复位ESMC information报文的发送定时器，并周期性发送携带新SSM级别的ESMC information报文。

自动选源机制
(1)     SSM级别最高的时钟源优先当选为最优时钟。
(2)     如果用户配置了SSM级别不参与自动选源，或者SSM级别相同，则按照时钟源的优先级进行选择，优先级值最小的时钟源优先被选中。
(3)     如果时钟源的优先级相同，则按照时钟源类型进行选择，优先选用BITS时钟源，其次选用线路时钟源，然后选用PTP时钟源。
(4)     如果时钟源的类型也相同，继续比较时钟信号入接口的编号，编号最小的时钟源优先被选中。
(5)     当BITS时钟源、线路时钟源、PTP时钟源均不可用时，使用本地时钟源。

选举出最优时钟后，设备会通过ESMC报文将最优时钟的SSM级别传递给下游设备，进一步影响下游设备最优时钟的选择。

如果某接口收到的时钟信号当选为最优时钟，而该接口在5秒钟内未收到ESMC information报文，设备会认为最优时钟丢失或不可用，将自动按照上述原则重新选择最优时钟。
当原最优时钟源恢复时，系统自动立即切换回原最优时钟。
```

## 时钟同步原理
```
选出最优时钟后，设备开始锁定最优时钟，进行时钟同步（频率同步）。

数字通信网中传递的是对信息进行编码后得到的PCM（Pulse Code Modulation，脉冲编码调制）数字脉冲信号，每秒生成的脉冲个数即为脉冲的频率。
以太网物理层编码采用FE（百兆）和GE（千兆）技术，平均每4个比特就插入一个附加比特，这样在其所传输的数据码流中不会出现超过4个1或者4个0的连续码流，可有效地包含时钟信息。
利用这种信息传输机制，SyncE在以太网源端接口上使用高精度的时钟发送数据，在接收端恢复、提取这个时钟，并作为接收端发送数据码流的基准。
In fact, the old 10Mbps (10Base-T) Ethernet is not even capable of synchronization signal transmission over the physical layer interface because a 10Base-T transmitter stops sending pulses during idle periods.

A 10Base-T transmitter simply sends a single pulse (“I am alive” pulse) every 16 ms to notify its presence to the receiving end. Of course, such infrequent pulses are not sufficient for clock recovery at the receiver.

Idle periods in faster Ethernet flavors (100Mbps, 1Gbps and 10Gbps) are continuously filed with pulse transitions, allowing continuous high-quality clock recovery at the receiver–good candidates for synchronized Ethernet.


假设外接时钟源1比外接时钟源2更可靠，当选为最优时钟。Device1和Device2均同步外接时钟源1的频率，同步原理如下：

1. 发送方向同步机制
发送端携带并传递同步信息：
(1)     因为外接时钟源1的SSM级别最高，Device1选择外接时钟源1作为最优时钟。
(2)     Device1提取外接时钟源1发送的时钟信号，并将时钟信号注入以太网接口卡的PHY芯片中。
(3)     PHY芯片将这个高精度的时钟信息添加在以太网线路的串行码流里发送出去，向下游设备Device2传递时钟信息。

2. 接收方向同步机制
接收方向提取并同步时钟信息：
(1)     Device2的以太网接口卡PHY芯片从以太网线路收到的串行码流里提取发送端的时钟信息，分频之后上送到时钟扣板。
(2)     时钟扣板将接口接收的线路时钟信号、外接时钟源2输入的时钟信号、本地晶振产生的时钟信号进行比较，根据自动选源算法选举出线路时钟信号作为最优时钟，并将时钟信号发送给时钟扣板上的锁相PLL。
(3)     PLL跟踪时钟参考源后，同步本地系统时钟，并将本地系统时钟注入以太网接口卡PHY芯片往下游继续发送，同时将本地系统时钟输出给本设备的业务模块使用。
```

## 时钟工作状态
```
系统时钟存在三种工作状态，用户通过查看系统时钟的状态信息，可了解设备的时钟同步情况：
·     跟踪状态：当设备选择了一个最优时钟，并和最优时钟达到频率同步，将处于跟踪状态。系统时钟处于跟踪状态时，时钟芯片内部会不断保存最优时钟的相关数据。
·     保持状态：当最优时钟失效，不能继续提供时钟信号时，时钟芯片会根据之前存储的相关数据，在一定时间（最长不超过24小时）内保持之前最优时钟的频率特征，提供与原最优时钟相符的时钟信号。此时，系统时钟处于保持状态。
·     自由振荡状态：若保持状态超时，原最优时钟仍未恢复，系统时钟会进入自由振荡状态。此时，设备使用内部晶振作为最优时钟。
```

## dpll
```
Digital Phase Locked Loop(DPLL)

+--------------------------+     +-------------------------------------------------+
|device1                   |     |device2                                          |
|                          |     |                                                 |
|+------------+    +-----+ |     | +-----+                                         |
|| clock      | -> | phy | | --> | | phy |<---------------+------------>业务模块   |
|+------------+    +-----+ |     | +--+--+                |系统时钟                |
+--------------------------+     |    |recovery clock     ｜                       |
                                 |    +------------------->dpll                    |
                                 +-------------------------------------------------+

接收方向提取并同步时钟信息：
(1)     Device2的以太网接口卡PHY芯片从以太网线路收到的串行码流里提取发送端的时钟信息，分频之后上送到时钟扣板。
(2)     时钟扣板将接口接收的线路时钟信号、外接时钟源2输入的时钟信号、本地晶振产生的时钟信号进行比较，根据自动选源算法选举出线路时钟信号作为最优时钟，并将时钟信号发送给时钟扣板上的锁相PLL。
(3)     PLL跟踪时钟参考源后，同步本地系统时钟，并将本地系统时钟注入以太网接口卡PHY芯片往下游继续发送，同时将本地系统时钟输出给本设备的业务模块使用。


Any Gigabit or 10 Gigabit Ethernet PHY device should be able to support synchronized Ethernet, so long as it provides a recovered clock on one of its output pins. 
The recovered clock is cleaned by the PLL and fed to the 25MHz crystal oscillator input pin on the PHY device. 
Some new Ethernet PHY devices provide a dedicated pin for the synchronization input. 
The advantage of this approach is that frequency input can be higher than 25MHz–higher clock frequencies usually have lower jitter. 
In addition, this approach avoids any potential timing loop problems within the PHY device.

From the discussion so far, it appears that the only requirement for a PLL used in SyncE is to clean jitter from the recovered clock, which can be accomplished with general purpose PLLs. 
However, the PLL used in SyncE must provide additional functions beyond jitter cleaning.

Free-run accuracy: 
The accuracy of PLL output when it is not driven by a reference should be equal or better than +/-4.6 ppm (part per million) over a time period of one year. 
This is a very accurate clock relative to the clock accuracy for traditional Ethernet (+/-100 ppm).

Holdover: 
The PLL constantly calculates the average frequency of the locked reference. If the reference fails and no other references are available, 
the PLL goes into holdover mode and generates an output clock based on a calculated average value. 
Holdover stability depends on the resolution of the PLL averaging algorithm and the frequency stability of the oscillator used as the PLL master clock.

Reference monitoring: 
The PLL needs to constantly monitor the quality of its input references. If the reference deteriorates (disappears or drifts in frequency), 
then the PLL raises an alarm (interrupt) and switches to another valid reference.

Hitless reference switching: 
If the PLL's reference fails, then it will lock to another available reference without phase disturbances at its output.

Jitter and wander filtering: 
The PLL can be viewed as a jitter and wander filter. The narrower the loop bandwidth, the better the jitter and wander attenuation.

Jitter and wander tolerance: 
The PLL should tolerate large jitter and wander at its input and still maintain synchronization without raising any alarms.
```

## configuration
```
dpll pin 有三种，
1.input：
用来设置dpll的输入，可以把SMA,GPS或者RCLK作为dpll输入
看起来还可以把ptp作为input？CVL-SDP看起来就是这种。
pin-parent-device的id是dpll序号

2.output：
用来设置dpll的输出？和input类似？

3.synce-eth-port：
用来开启和关闭input pin？
通过pin-parent-pin中的pin-id关联某个input pin，然后可以disable和enable input pin

There are 2 recovery clocks on each device RCLKA and RCLKB

RCLKA can only be connected to one port at a time and RCLKB can only be connected to one port at
a time. Both can be assigned to the same port at the same time. Which one is connected is based on
priority of the pin

Enable RCLKA on pin-id 13 (RCLKA is ("pin-parent-pin":{"pin-id":2))
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":13, "pinparent-pin":{"pin-id":2, "pin-state":1}}'

Use the pin-get command to see the status:
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --dump pin-get

Verify RCLKA ‘input’ (pin id 2) is connected using the pin-get command:
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --dump pin-get

Verify dpll status changes from unlocked/holdover to locked/locked-ho-acq
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --dump device-get

Enable RCLKB on pin-id 15 (RCLKB is ("pin-parent-pin":{"pin-id":3))
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":15, "pinparent-pin":{"pin-id":3, "pin-state":1}}'

Notice RCLKB input pin id 3 will not be connected until the you change its priority on either
dpll 0 or 1 to be higher than RCLKA. Current priority is 8 for RCLKA for dpll 0 and 1 and
current priority for RCLKB for dpll 0 and 1 is 9. So change RCLKB prio to 7.
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":3, "pin-parentdevice":{"id":1, "pin-prio":7}}'

Should see RCLKB dpll 1 (pin-parent-device – id 1) show connected and RCLKA dpll 1 (pinparent-device – id 1) be selectable now. 
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-get --json '{"pin-id":3}'
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-get --json '{"pin-id":2}


Disable recovery on both port pins:
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":13,
"pin-parent-pin":{"pin-id":2, "pin-state":3}}'
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":15,
"pin-parent-pin":{"pin-id":3, "pin-state":3}}'

Verify recovery is disabled on both port pins:
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --dump pin-get
# 

Verify dpll is in holdover state:
./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --dump device-get

Enable recovery on both port pins again:
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":13,
"pin-parent-pin":{"pin-id":2, "pin-state":1}}'
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":15,
"pin-parent-pin":{"pin-id":3, "pin-state":1}}'

synce configuration:
server node external_input=1 internal_input=0
follower node external_input=0 internal_input=1
在synce配置文件里面添加dpll_state_file
在synce配置文件里面添加port，并为每个port设置recover_clock_file
启动synce4l on each side:
synce4l -f conifgs/synce.cfg -m

Go to /sys/class/net/<device name>/device/phy folder and execute the following command to enable
RCLKA as follows,
Note - do: echo <ENABLE (1) or DISABLE (0)> <WHICH RCLK, either 0 or 1>
# echo 1 0 > synce

Again DPLL status and RCLKA pin should be valid now,
# cat /sys/kernel/debug/ice/0000\:18\:00.0/dpll

turn off SW pin of dpll , it'll trigger DNU for quality signal and ext QL = 255 or 0xff
# echo 0 0 > synce
```
