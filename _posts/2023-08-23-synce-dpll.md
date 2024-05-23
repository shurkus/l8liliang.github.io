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

[Reference4](https://www.ciscopress.com/articles/article.asp?p=3128857&seqNum=2)
{:.info}

## Clock
```
In everyday usage, the term clock refers to a device that maintains and displays the time of day and perhaps the date. 
In the world of electronics, however, clock refers to a microchip that generates a clock signal, 
which is used to regulate the timing and speed of the components on a circuit board. 
This clock signal is a waveform that is generated either by a clock generator or the clock itself—the most common form of clock signal in electronics is a square wave.
```

## PLL
```
A phase-locked loop (PLL) is an electronic device or circuit that generates an output clock signal that 
is phase-aligned as well as frequency-aligned to an input clock signal. 
As shown in Figure 5-3, in its simplest form, a PLL circuit consists of three basic elements, as described in the list that follows.
```

## Low-Pass and High-Pass Filters
```
A low-pass filter (LPF) is a filter that filters out (removes) signals that are higher than a fixed frequency, 
which means an LPF passes only signals that are lower than a certain frequency—hence the name low-pass filter. 
For the same reasons, sometimes LPFs are also called high-cut filters because they cut off signals higher than some fixed frequency. 
Figure 5-5 illustrates LPF filtering where signals with lower frequency than a cut-off frequency are not attenuated (diminished). 
The pass band is the range of frequencies that are not attenuated, and the stop band is the range of frequencies that are attenuated.
```

## Jitter and Wander
```
Noise is classified as either jitter or wander. As seen in the preceding section “Jitter and Wander,” by convention, 
jitter is the short-term variation, and wander is the long-term variation of the measured clock compared to the reference clock. 
The ITU-T G.810 standard defines jitter as phase variation with a rate of change greater than or equal to 10 Hz, 
whereas it defines wander as phase variation with a rate of change less than 10 Hz. In other words, slower-moving, low-frequency jitter (less than 10 Hz) is called wander.

https://www.ciscopress.com/articles/article.asp?p=3128857
如这里面的Figure5-8: jitter表示每个上升沿或者下降沿的时间差别，wander表示时间差别在变大还是缩小。

Jitter和Wander就在哪里，你可以用不同的方法去测量。
比如，MTIE和TDEV都属于一种Wander
```

## Frequency Error
```
While jitter and wander are both metrics to measure phase errors, the frequency error (or accuracy) also needs to be measured.

The frequency error (also referred to as frequency accuracy) is the degree to which the frequency of a clock can deviate from a nominal (or reference) frequency. 
The metric to measure this degree is called the fractional frequency deviation (FFD) or sometimes just the frequency offset. 
This offset is also referred to as the fractional frequency offset (FFO).

y(t)=(v(t)-v(nom))/v(nom)

where:

y(t) is the FFD at time t
v(t) is the frequency being measured and
v(nom) is the nominal (or reference) frequency

FFD is often expressed in parts per million (ppm) or parts per billion (ppb).

```

## Time Error
```
就是两个时钟的时间差
```

## max TE
```
max|TE|, which is defined as the maximum absolute value of the time error observed over the course of the measurement
```

## Time Interval Error
```
see https://www.ciscopress.com/articles/article.asp?p=3128857&seqNum=2

The time interval error (TIE) is the measure of the change in the TE over an observation interval. 

In comparison, the TE is the instantaneous measure between the two clocks; there is no interval being observed. 
So, while the TE is the recorded error between the two clocks at any one instance, the TIE is the error accumulated over the interval (length) of an observation. 
Another way to think of it is that the TE is the relative time error between two clocks at a point (instant) of time, 
whereas TIE is the relative time error between two clocks accumulated between two points of time (which is an interval).

TE是绝对error，TIE是intervel内积累的新error.

```

## Constant Versus Dynamic Time Error
```
Constant time error (cTE) is the mean of the TE values that have been measured. 

Dynamic time error (dTE) is the variation of TE over a certain time interval (you may remember, the variation of TE is also measured by TIE).
Additionally, the variation of TE over a longer time period is known as wander, so the dTE is effectively a representation of wander.
Another way to think about it is that dTE is a measure of the stability of the clock.

These two metrics are very commonly used to define timing performance, so they are important concepts to understand. 
Normally, dTE is further statistically analyzed using MTIE and TDEV; and the next section details the derivations of those two metrics.
```

## Maximum Time Interval Error
```
TIE是取x(n+k)-x(n)
但是这两个指（x(n+k)， x(n)）不一定是最大和最小值。
MTIE是找到这个interval之间的最大值和最小值，取差。

Note also that as the MTIE is the maximum value of delay variation, it is recorded as a real maximum value; 
not only for one observation interval but for all observation intervals. 
This means that if a higher value of MTIE is measured for subsequent observation intervals, it is recorded as a new maximum or else the MTIE value remains the same. 
This in turn means that the MTIE values never decrease over longer and longer observation intervals.
```

## Time Deviation
```
Whereas MTIE shows the largest phase swings for various observation intervals, time deviation (TDEV) provides information about phase stability of a clock signal. 
TDEV is a metric to measure and characterize the degree of phase variations present in the clock signal, primarily calculated from TIE measurements.

Unlike MTIE, which records the difference between the high and low peaks of phase variations, 
TDEV primarily focuses on how frequent and how stable (or unstable) such phase variations are occurring over a given time—note the importance of “over a given time.” 
```

## Noise
```
The metrics that are discussed in this chapter try to quantify the noise that a clock can generate at its output. 
For example, max|TE| represents the maximum amount of noise that can be generated by a clock. Similarly, cTE and dTE also characterize different aspects of the noise of a clock.

Noise is classified as either jitter or wander. As seen in the preceding section “Jitter and Wander,” by convention, jitter is the short-term variation, 
and wander is the long-term variation of the measured clock compared to the reference clock. 
The ITU-T G.810 standard defines jitter as phase variation with a rate of change greater than or equal to 10 Hz, 
whereas it defines wander as phase variation with a rate of change less than 10 Hz. In other words, slower-moving, low-frequency jitter (less than 10 Hz) is called wander.

# Noise Generation
Noise generation is the amount of jitter and wander that is added to a perfect reference signal at the output of the clock.

# Noise Tolerance
A slave clock can lock to the input signal from a reference master clock, and yet every clock (even reference clocks) generates some additional noise on its output. 
As described in the previous section, there are metrics that quantify the noise that is generated by a clock. 
But how much noise can a slave clock receive (tolerate) on its input and still be able to maintain its output signal within the prescribed performance limits?
It is simply a measure of how bad an input signal can become before the clock can no longer use it as a reference.

# Noise Transfer
This propagation of noise from input to output of a node is called noise transfer and can be defined as the noise that is seen at the output 
of the clock due to the noise that is fed to the input of the clock. 
The filtering capabilities of the clock determines the amount of noise being transferred through a clock from the input to the output. Obviously, less noise transfer is desirable
```

## Transient Response
```
The question arises, how should the clock behave when an input reference clock is lost and the clock decides to select the second-best clock source as the input reference? 
The behavior during this event is characterized as the transient response of a clock.
```

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

### eEEC and 1PPS
```
> Hello Pasi,
>
> Do you know why DPLL outputs two signals, i.e. DPLL0 outputting a
> clock signal and DPLL1 outputting 1PPS signal?
>
DPLL0只处理频率，来源包括RCLK和PTP，(当只有GNSS 1PPS输入的时候，也通过1PPS进行频率同步）。
DPLL1处理相位，来源包括GNSS和PTP。
Yes, DPLL0 is eEEC and deals with the frequency only (including input
sources other than GNSS, such as clocks recovered from Ethernet
interfaces or E810 when PTP is acting as source), while DPLL1 deals with
the phase only (i.e. it is syncronized to 1Hz pulse per second,
irrespective of the source), which is basically "exact" (to the extent
possible) location of the 1Second boundary, irrespective of it's source
(either GNSS receiver or PTP / E810).
> From my understanding the clock signal is for synchronizing frequency 
> and 1PPS signal is for synchronizing phase?
>
yes, see above
> But with 1PPS input from GNSS, DPLL can achieve both frequency and
> phase synchronization.
>
GNSS其实也输出频率，但是没有连接到DPLL。
如果只连接了1PPS，DPLL0和DPLL1都使用1PPS进行同步。
Yes, it can be debated whether this is optimal, but it works well enough
in this case. There is also freq output from GNSS module but it is not
physically wired to DPLL, so both frequency and phase aligment pieces
are sourced from 1PPS signal only. Both DPLLs in this case use 1PPS as
inputs.
> So it seems 1PPS is enough for frequency and phase synchronization,
> why DPLL outputs two signals?
>
没太看懂，DPLL会输出多个phase和frequency？
DPLL outputs way more than two signals on both phase (although in that
case they are generally all 1PPS phase aligned signals irrespective of
the destination). On frequency side there are many outputs as well, and
with different frequencies (10Mhz, 155+ Mhz, clocik to E810 etc.).
> What's the difference between clock signal and 1PPS signal?
clock就是固定频率的信号，不能表示ToD.
Clock is what it sounds like, a periodic signal with specific frequency
(e.g. 10Mhz clock has 10M transitions per second), clock signal can not
be signify time by itself, but high speed clocks are required for
various purposes, including ns level timekeeping as well as PHY clocks
etc. Clock signal transitions are not necessarily aligned in phase with
the 1PPS transition, although SOME of them are or can be, it dpends on
the specific clock and it's purpose.

ToD is ASCII string from GNSS module in GM case, (in WPC/LB case it's NMEA sentence) which tells what second the 1PPS lo-hi transition edge is associated with, 
as from the transition alone you only  know that it is start of new second, which is necessary but not sufficient 
to figure out the time which is what we are fundamentally trying to sychronize w/ 1PPSs.

Pasi
```

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

No, phase is not absolute time. There is phase synchronization, frequency synchronization and time synchronization. 
So two clocks can be running at the same speed (frequency) but it they don "tick" at the same time their phases are not aligned. 
That is what phase synchronization provides.

The output from the 1PPS from the GNSS receiver in the E810 can be configured to provide phase synchronization to the DPLL1.  
The T-GM use case leveraging time reference signal from GNSS satellite depends on that.
相位同步的前提是频率同步。1PPS即可以同步频率也可以同步相位.(实际上频率同步是通过计算相位变化速率实现的. 
DPLL可以接受任何引号，比如1pps或者RCLK的信号，去达到相位同步?
RCLK从网络上获取频率，并且把相应的频率信号发送给DPLL，当然所有这种信号都包含频率信息和相位信息，所以DPLL可以使用这个信号去lock自己?

https://www.nist.gov/pml/time-and-frequency-division/popular-links/time-frequency-z/time-and-frequency-z-p#:~:text=The%20time%20interval%20for%201,time%20shift%20of%20555%20picoseconds)

DPLL0 is used for generating high stability clock signal and DPLL1 used for driving the output signals.
DPLL0用于生成内部使用的时钟信号(frequency only)，DPLL1用于生成1PPS信号.
ECC (DPLL0) driving the internal clocks and PPS (DPLL1) driving all 1PPS signals.
EEC - DPLL0 = Ethernet equipment clock source from DPLL0 for frequency adjustments., glitchless.
PPS - DPLL1 = 1 PPS generation from DPLL1 for phase adjustments. Glitches allowed. Slower locking.
为什么DPLL1的1PPS输出不能用来同步frequency？DPLL就是同步到GNSS的1PPS输出啊？是因为网卡更容易通过时钟信号同步frequency？

Set periodic output on SDP20 (to synchronize the DPLL1 to the E810 PHC synced by ptp4l): # echo 1 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Or, if users want to set SDP22 (to synchronize the DPLL0 to E810 PHC synced by ptp4l): # echo 2 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Set the periodic output on SDP20 (to synchronize the DPLL1 to the E810 PHC synced by ptp4l): # echo 1 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Or, if users want to set SDP22 (to synchronize the DPLL0 and DPLL1 to the E810 PHC synced by ptp4l):
# echo 2 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period

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
In the default configuration, the E810-XXVDA4T driver enables monitoring of the DPLL events and reports state changes 
in the default system log (dmesg) with the WARN level independently for each of the DPLL units.
DPLLs start in a holdover mode and enter an unlocked and locked state when a valid reference input is enabled. 
If the current input becomes invalid, DPLLs change state to holdover. When the reference reappears (or a different valid input is present), 
the DPLL state changes to the unlocked state and locks to a new signal.

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

Set periodic output on SDP20 (to synchronize the DPLL1 to the E810 PHC synced by ptp4l): # echo 1 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Or, if users want to set SDP22 (to synchronize the DPLL0 to E810 PHC synced by ptp4l): # echo 2 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Set the periodic output on SDP20 (to synchronize the DPLL1 to the E810 PHC synced by ptp4l): # echo 1 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
Or, if users want to set SDP22 (to synchronize the DPLL0 and DPLL1 to the E810 PHC synced by ptp4l):
# echo 2 0 0 1 0 > /sys/class/net/$ETH/device/ptp/ptp*/period
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
用来设置dpll的输入，可以把CVL,SMA,U.FL2,GPS或者RCLK作为dpll输入
通过CVL-SDP可以把ptp作为input。
pin-parent-device的id是dpll序号

2.output：
输出引脚，用来设置dpll的输出.
包括REF-SMA1,REF-SMA2,U.FL1,PHY-CLK,MAC-CLK,CVL-SDP

3.synce-eth-port：
input pin中的RCLKA和RCLKB是multiplexer类型的pin。
synce-eth-port可以注册到这种multiplexer类型的pin，注册之后，他的parent就是multiplexer类型的pin。
一个synce-eth-port可以注册到多个mux pins。
多个synce-eth-port可以注册到同一个mux pins，但是同一时刻只有一个synce-eth-port pin可以作为mux pin的input。
可以通过 设置state来指定哪个pin作为input生效。
比如 Enable RCLKA on pin-id 13 (RCLKA is ("pin-parent-pin":{"pin-id":2))
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":13, "pinparent-pin":{"pin-id":2, "pin-state":1}}'

There are 2 recovery clocks on each device RCLKA and RCLKB

RCLKA can only be connected to one port at a time and RCLKB can only be connected to one port at
a time. Both can be assigned to the same port at the same time. Which one is connected is based on
priority of the pin

Enable RCLKA on pin-id 13 (RCLKA is ("pin-parent-pin":{"pin-id":2))
# ./cli.py --spec /usr/src/kernels/linux-dpll-net-next-dpllv11/Documentation/netlink/specs/dpll.yaml --do pin-set --json '{"pin-id":13, "pinparent-pin":{"pin-id":2, "pin-state":1}}'
上面的命令有问题，下面可以执行
# ./cli.py --spec /root/dpll.yaml --schema /root/genetlink.yaml --do pin-set --json '{"id":13, "parent-pin":{"parent-id":2, "state":1}}'
{'capabilities': 4,
  'clock-id': 5799633565436792966,
  'id': 13,
  'module-name': 'ice',
  'parent-pin': [{'parent-id': 2, 'state': 'connected'},
                 {'parent-id': 3, 'state': 'disconnected'}],
  'type': 'synce-eth-port'},

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
上面命令不能执行，下面可以执行
# ./cli.py --spec /root/dpll.yaml --schema /root/genetlink.yaml --do pin-set --json '{"id":6, "parent-device":{"parent-id":1, "prio":5}}'

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

## ts2phc
```
-s generic 
  Use the key word "generic" for an external 1-PPS without ToD information.
  1PPS from DPLL1
-s nmea
  Use the key word "nmea" for an external 1-PPS from a GPS providing ToD information via the RMC NMEA sentence.
  就是GNSS console ToD

如果用generic，就只需SDP21或者23
如果用nmea，那么还需要console ToD。

GNSS recvr syncs itself from the GNSS constellation(s) → 
GNSS syncs (DPLL0) and DPLL1 → 
syncs E810’s PHC (with ts2phc, using ToD NMEAs from the vSerial of GNSS + 1PPS out from DPLL1 / 1PPS in E810 SDP21|23) → 
phc2sys sync node clock fom PHC

The phc2sys tool should not be run at the same time as ts2phc using the generic source of ToD (-s generic). 
In the default configuration, ts2phc uses hardware-generated timestamps along with the system timer to create correction values. 
Running the tools in parallel can create a feedback that breaks time synchronization. 
The leapfile option is available but not necessary for the program to run. 
Also, the default .leap file is not compatible with ts2phc.

```

## synce4l
```
https://github.com/intel/synce4l

The lack of messages is considered to be a failure condition. 
The protocol behaviour is such that the quality level is considered QL-FAILED if no SSM messages are received after a five second period.

wait-to-restore:
The wait-to-restore time ensures that a previous failed synchronization source is only again considered as available by the selection process if it is fault-free for a certain time.
就是必须恢复正常多长时间之后才认为他真正有效。
recover_time和wait-to-restore是同样的意思。
停止master synce4l之后，slave side会因为5秒内未收到SSM而进入QL-FAILED状态。
再启动master synce4l之后，slave在收到SSM消息之后，要等待recover_time才能切换为lock状态

network_option:
Network option according to T-REC-G.8264. All devices in SyncE domain should have the same option configured.

tx_heartbeat_msec:
ESMC发送间隔

rx_heartbeat_msec:
ESMC poll间隔

use_syslog 1
message_tag                [synce4l]
vim /etc/rsyslog.conf 
:syslogtag, contains, "synce4l"                         /var/log/synce

input_QL                   0x1   // SSM code https://github.com/intel/synce4l
input_ext_QL               0x20  // Extended SSM code https://github.com/intel/synce4l

```
