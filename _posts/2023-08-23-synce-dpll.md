---
layout: article
tags: Linux
title: SYNCE & DPLL
mathjax: true
key: Linux
---

[Reference1](https://www.h3c.com/cn/Service/Document_Software/Document_Center/Home/Switches/00-Public/Learn_Technologies/White_Paper/(SyncE)_WP-6W100/)
{:.info}

[Reference2](https://www.embedded.com/an-introduction-to-synchronized-ethernet/)
{:.info}

[Reference3](https://www.cisco.com/c/en/us/support/docs/ios-nx-os-software/ios-xr-software/217579-configure-ptp-and-synce-basics-with-cisc.html#anc7)
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
Digital Phase Locked Loop(DPLL)
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

## E810

### E810 Architecture
```
(output) U.FL1   U.FL2(input)    GNSS
	   ^       |              |
           |       |              |
           +       |              v
SMA1----SMA Logic<-|--------->DPLL(        ) -----------------+---+--+--+--+
                   |           ^  ^  ^   |R                   |   ^  |  ^  |REFCLK
              +----+           |  |R |R  |E                   |S  |S |S |S |
              |                |  |C |C  |F                   |D  |D |D |D |
              v                |  |L |L  |C                   |P  |P |P |P |
SMA2----SMA Logic<-------------+  |A |B  |L                   |2  |2 |2 |2 |
                                  |  |   |K                   |3  |2 |1 |0 |
                                  |  |   v                    v   |  v  |  v
SFP-------------------------------+--+-<-+                   PHC(                  ) 
SFP-------------------------------+--+                       |
SFP-------------------------------+--+-----------------------+
SFP-------------------------------+--+

The timing information from the DPLL (that can arrive from the GNSS module, SMA connectors,
E810 SDPs, or from the recovered SyncE signals) are routed to the E810 and can be used to
synchronize the PHC (PTP hardware clocks). All E810 ports share only one physical PHC, and this is
particularly useful in creating a Boundary Clock (BC) functionality like what is defined in ITU-T
G.8273.2 (but without SyncE).

DPLL0 is used for generating high stability clock signal and DPLL1 used for driving the output signals.
DPLL0用于内部生成内部使用的时钟信号，DPLL1用于向外发送1PPS信号？
ECC (DPLL0) driving the internal clocks and PPS (DPLL1) driving all 1PPS signals.
EEC - DPLL0 = Ethernet equipment clock source from DPLL0 for frequency adjustments.,
glitchless.
PPS - DPLL1 = 1 PPS generation from DPLL1 for phase adjustments. Glitches allowed. Slower
locking.

The Linux kernel provides the standard interface for controlling external synchronization pins
To check if the kernel has the required PTP and pin interface, run the following command:
# cat /proc/kallsyms | grep ptp_pin

In the following example, the driver exposes the ptp7 device:
#ls -R /sys/class/ptp/*/pins/
/sys/class/ptp/ptp7/pins/:
GNSS SMA1 SMA2 U.FL1 U.FL2

In the following example, the ens260f0 net interface exposes pins through the ptp7 interface:
#ls -R /sys/class/net/*/device/ptp/*/pins
/sys/class/net/ens260f0/device/ptp/ptp7/pins:
GNSS SMA1 SMA2 U.FL1 U.FL2

Users can also run ethtool -T <interface name> to show the PTP clock number.
# ethtool -T <interface name>
PTP Hardware Clock: 7
```

### 设置DPLL pin
```
The E810-XXVDA4T has four connectors for external 1PPS signals: SMA1, SMA2, U.FL1, and U.FL2
• SMA connectors are bidirectional and U.FL are unidirectional.
• U.FL1 is 1PPS output and U.FL2 is 1PPS input.
• SMA1 and U.FL1 connectors share channel one.
• SMA2 and U.FL2 connectors share channel two.

echo <function> <channel> > /sys/class/net/$ETH/device/ptp/*/pins/SMA1(SMA2,U.FL1,U.FL2)
function: 
  0 = Disabled
  1 = Rx
  2 = Tx
channel: 
  1 = SMA1 or U.FL1
  2 = SMA2 or U.FL2

channel 1:
1. SMA1 as 1PPS input:
# echo 1 1 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA1
2. SMA1 as 1PPS output:
# echo 2 1 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA1
channel 2:
1. SMA2 as 1PPS input:
# echo 1 2 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA2
2. SMA2 as 1PPS output:
# echo 2 2 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA2
```
### Recovered Clocks (G.8261 SyncE Support)  
```
Recovered clocks can be configured using a special sysfs interface that is exposed by every port
instance. Writing to a sysfs under a given port automatically enables a recovered clock from a given
port that is valid for a current link speed. A link speed change requires repeating the steps to enable the
recovered clock.

If a port recovered clock is enabled and no higher-priority clock is enabled at the same time, the DPLL
starts tuning its frequency to the recovered clock reference frequency enabling G.8261 functionality.
There are two recovered clock outputs from the C827 PHY. Only one pin can be assigned to one of the
ports. Re-enabling the same pin on a different port automatically disables it for the previously-assigned
port.

1. To enable a recovered clock for a given Ethernet device run the following:
# echo <enable> <pin(clock id)> > /sys/class/net/$ETH/device/phy/synce
where:
	ena: 0 = Disable the given recovered clock pin.
	     1 = Enable the given recovered clock pin.
	pin: 0 = Enable C827_0-RCLKA (higher priority pin).
	     1 = Enable C827_0-RCLKB (lower priority pin).

For example, to enable the higher-priority recovered clock from Port 0 and a lower-priority recovered clock from Port 1, run the following:
# export ETH0=enp1s0f0
# export ETH1=enp1s0f1
# echo 1 0 > /sys/class/net/$ETH0/device/phy/synce
# dmesg
[27575.495705] ice 0000:03:00.0: Enabled recovered clock: pin C827_0-RCLKA
# echo 1 1 > /sys/class/net/$ETH1/device/phy/synce
# dmesg
[27575.495705] ice 0000:03:00.0: Enabled recovered clock: pin C827_0-RCLKB

2.Disable recovered clocks:
# echo 0 0 > /sys/class/net/$ETH0/device/phy/synce
# dmesg
[27730.341153] ice 0000:03:00.0: Disabled recovered clock: pin C827_0-RCLKA
# echo 0 1 > /sys/class/net/$ETH1/device/phy/synce
# dmesg
[27730.341153] ice 0000:03:00.0: Disabled recovered clock: pin C827_0-RCLKB

Check recovered clock status:
You can add the current status of the recovered clock to the dmesg:
#echo dump rclk_status > /sys/kernel/debug/ice/0000:03:00.0/command
# dmesg
[311274.298749] ice 0000:03:00.0: State for port 0, C827_0-RCLKA: Disabled
[311274.300060] ice 0000:03:00.0: State for port 0, C827_0-RCLKB: Disabled
```

### External Timestamp Signals
```
The E810-XXVDA4T can use external 1PPS signals filtered out by the DPLL as its own time reference.
When the DPLL is synchronized to the GNSS module or an external 1PPS source, the ts2phc tool can be
used to synchronize the time to the 1PPS signal.

# export ETH=enp1s0f0
# export TS2PHC_CONFIG=/home/<user>/linuxptp-3.1/configs/ts2phc-generic.cfg
# ts2phc -f $TS2PHC_CONFIG -s generic -m -c $ETH
# cat $TS2PHC_CONFIG
[global]
use_syslog 0
verbose 1
logging_level 7
ts2phc.pulsewidth 100000000
#For GNSS module
#ts2phc.nmea_serialport /dev/ttyGNSS_BBDD_0 #BB bus number DD device number /dev/
ttyGNSS_1800_0
#leapfile /../<path to .list leap second file>
[<network interface>]
ts2phc.extts_polarity
rising
```

### Periodic Outputs From DPLL (SMA and U.FL Pins)
```
The E810-XXVDA4T supports two periodic output channels (SMA1 or U.FL1 and SMA2). Channels can be
enabled independently and output 1PPS generated by the embedded DPLL. 1PPS outputs are
synchronized to the reference input driving the DPLL1. Users can read the current reference signal
driving the 1PPS subsystem by running the following command:

# dmesg | grep "<DPLL1> state changed" | grep locked | tail -1
[ 342.850270] ice 0000:01:00.0: DPLL1 state changed to: locked, pin GNSS-1PPS

The following configurations of 1PPS outputs are supported:
1. SMA1 as 1PPS output:
# echo 2 1 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA1
2. U.FL1 as 1PPS output:
# echo 2 1 > /sys/class/net/$ETH/device/ptp/ptp*/pins/U.FL1
3. SMA2 as 1PPS output:
# echo 2 2 > /sys/class/net/$ETH/device/ptp/ptp*/pins/SMA2
```

###  Reading Status of the DPLL
```
# cat /sys/kernel/debug/ice/$PCI_SLOT/cgu
Found ZL80032 CGU
DPLL Config ver: 1.3.0.1
CGU Input status:
 | | priority |
 input (idx) | state | EEC (0) | PPS (1) |
 ---------------------------------------------------
 CVL-SDP22 (0) | invalid | 8 | 8 |
 CVL-SDP20 (1) | invalid | 15 | 3 |
 C827_0-RCLKA (2) | invalid | 4 | 4 |
 C827_0-RCLKB (3) | invalid | 5 | 5 |
 SMA1 (4) | invalid | 1 | 1 |
 SMA2/U.FL2 (5) | invalid | 2 | 2 |
 GNSS-1PPS (6) | valid | 0 | 0 |
EEC DPLL:
Current reference: GNSS-1PPS
Status: locked_ho_ack
PPS DPLL:
Current reference: GNSS-1PPS
Status: locked_ho_ack
Phase offset: -217

The first section of the log shows the status of CGU inputs (references) including its index number.
Active references currently selected are listed in Section 4.2. EEC Ethernet equipment clock (DPLL0)
skips the 1PPS signal received on the CVL-SDP20 pin.
The second section lists all internal DPLL units. ECC (DPLL0) driving the internal clocks and PPS (DPLL1)
driving all 1PPS signals.
```

### DPLL Monitoring
```
Enabling DPLL monitoring:
ethtool --set-priv-flags $ETH dpll_monitor on
Disabling DPLL monitoring:
ethtool --set-priv-flags $ETH dpll_monitor off
```

### pin_cfg User Readable Format
```
To check the DPLL pin configuration:
# cat /sys/class/net/ens4f0/device/pin_cfg
in
| pin| enabled| freq| phase_delay| esync| DPLL0 prio| DPLL1 prio|
| 0| 1| 1| 0| 0| 8| 8|
| 1| 1| 1| 0| 0| 15| 3|
| 2| 1| 1953125| 0| 0| 4| 4|
| 3| 1| 1953125| 0| 0| 5| 5|
| 4| 1| 1| 7000| 0| 1| 1|
| 5| 1| 1| 7000| 0| 2| 2|
| 6| 1| 1| 0| 0| 0| 0|
out
| pin| enabled| dpll| freq| esync|
| 0| 1| 1| 1| 0|
| 1| 1| 1| 1| 0|
| 2| 1| 0| 156250000| 0|
| 3| 1| 0| 156250000| 0|
| 4| 1| 1| 1| 0|
| 5| 1| 1| 1| 0|
In the “in” table. the pin numbers are referred from the DPLL Priority See Section 4.2, “DPLL Priority”.
In the “out” table pin 0 is SMA1 pin 1 is SMA2, all the other values do not modify.

Changing the DPLL priority list:
# echo "prio <prio value> dpll <dpll index> pin <pin index>" > \ /sys/class/net/<dev>/device/pin_cfg
where:
prio value = Desired priority of configured pin [0-14]
dpll index = Index of DPLL being configured [0:EEC (DPLL0), 1:PPS (DPLL1)]
pin index = Index of pin being configured [0-9]

Example:
Set priority 1 for pin 3 on DPLL 0:
# export ETH=enp1s0f0
# echo "prio 1 dpll 0 pin 3" > /sys/class/net/$ETH/device/pin_cfg

Changing input/output pin configuration:
# echo "<direction> pin <pin index> <config>" > /sys/class/net/<dev>/device/pin_cfg
where:
direction = pin direction being configured [“in”: input pin, “out”: output pin]
pin index = index of pin being configured [for in 0-6 (see DPLL priority section); for out 0:SMA1 1: SMA2]
config = list of configuration parameters and values:
[ "freq <freq value in Hz>",
 "phase_delay <phase delay value in ns>" // NOT used for out,
 "esync <0:disabled, 1:enabled>"
 "enable <0:disabled, 1:enabled>" ]

Note: The esync setting has meaning only with the 10 MHz frequency, you need to have esync to have the same setting in both ends of the SMA.
Example:
# export ETH=enp1s0f0
Set freq to 10 MHz on input pin 4: DPLL will lock only if 10 MHz signals arrive on SMA1 and it has been enabled for input.
# echo "in pin 4 freq 10000000" > /sys/class/net/$ETH/device/pin_cfg

```

### cgu_ref_pin/cgu_state Machine Readable Interface
```
To find out which pin the DPLL0 (EEC DPLL) is locked on, check the cgu_ref_pin:
# cat /sys/class/net/<dev>/device/cgu_ref_pin
To check the state of the DPLL0 (EEC DPLL) you can check the cgu_state:
# cat /sys/class/net/<dev>/device/cgu_state
DPLL_UNKNOWN = -1,
DPLL_INVALID = 0,
DPLL_FREERUN = 1,
DPLL_LOCKED = 2,
DPLL_LOCKED_HO_ACQ = 3,
DPLL_HOLDOVER = 4
The cgu_state interface used by synce4l as well.
```

### 1PPS Signals from E810 Device to DPLL
```
The E810-XXVDA4T implements two 1PPS signals coming out of the MAC (E810 device) to the DPLL.
They serve as the phase reference (CVL-SDP20) and as both phase and frequency reference (CVLSDP22) signals.
To enable a periodic output, write five integers into the file: channel index, start time seconds, start
time nanoseconds, period seconds, and period nanoseconds. To disable a periodic output, set all the
seconds and nanoseconds values to zero.

1. To enable the phase reference pin (CVL-SDP20):
# echo 1 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period

2. To enable the phase and frequency reference pin (CVL-SDP22):
# echo 2 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period

3. To disable the phase reference pin (CVL-SDP20):
# echo 1 0 0 0 0 > /sys/class/net/$ETH/device/ptp/ptp*/period

4. To disable the phase and frequency reference pin (CVL-SDP22):
# echo 2 0 0 0 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
```

### 1PPS Signals from the DPLL to E810 Device
```
The DPLL automatically delivers 2x 1PPS signals to the E810 device on pin 21 and 23. These signals can
be used to synchronize the E810 to the DPLL phase with the ts2phc program. The E810 will capture the
timestamp when the 1PPS signal arrives.

To ensure that the timestamps from the 1PPS signals are only used when the DPLL is locked to a signal,
you can enable the 1 PPS filtering option. By default, filtering is disabled, and all the timestamps are
passed along by the E810 ice driver.

The DPLL 1PPS filtering can be enabled (on) or disabled (off) by using the ethtool command in the Linux
kernel.

# export ETH=ens801f0
#ethtool --show-priv-flags $ETH
Private flags for ens801f0:
link-down-on-close : off
fw-lldp-agent : off
channel-inline-flow-director : off
channel-inline-fd-mark : off
channel-pkt-inspect-optimize : on
channel-pkt-clean-bp-stop : off
channel-pkt-clean-bp-stop-cfg: off
vf-true-promisc-support : off
mdd-auto-reset-vf : off
vf-vlan-pruning : off
legacy-rx : off
dpll_monitor : on
extts_filter : off
Enabling DPLL's 1 filtering:
ethtool --set-priv-flags $ETH extts_filter on
Disabling DPLL's 1 filtering:
ethtool --set-priv-flags $ETH extts_filter off
```

## From Intel DPLL TestPlan
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
通过pin-parent-pin中的pin-id关联某个RCLK input pin，然后可以disable和enable input pin

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
