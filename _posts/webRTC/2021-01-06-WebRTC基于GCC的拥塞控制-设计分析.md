---
layout: post
title: WebRTC基于GCC的拥塞控制(下)-实现分析
date: 2021-01-06
tag: webRTC
---

### 前言
从源代码实现角度对WebRTC的GCC算法进行分析。主要内容包括： RTCP RR的数据源、报文构造和接收，接收端基于数据包到达延迟的码率估计，发送端码率的计算以及生效于目标模块。


拥塞控制是实时流媒体应用的重要服务质量保证。通过本文和文章[1][2]，从数学基础、算法步骤到实现细节，对WebRTC的拥塞控制GCC算法有一个全面深入的理解，为进一步学习WebRTC奠定良好基础。

### 1 GCC算法框架再学习

本节内容基本上是文章[1]第1节的复习，目的是再次复习GCC算法的主要框架，梳理其算法流程中的数据流和控制流，以此作为后续章节的行文提纲。GCC算法的数据流和控制流如图1所示。

![2844879-91005d8cd84ba66e](http://image.smartjames.cn/mweb/20210105/16098246662333.png)

<center><font color="#808080" size="2">图1 GCC算法数据量和控制流</font></center>

对发送端来讲，GCC算法主要负责两件事：1)接收来自接收端的数据包信息反馈，包括来自RTCP RR报文的丢包率和来自RTCP REMB报文的接收端估计码率，综合本地的码率配置信息，计算得到目标码率A。2)把目标码率A生效于目标模块，包括PacedSender模块，RTPSender模块和ViEEncoder模块等。

对于接收端来讲，GCC算法主要负责两件事：1）统计RTP数据包的接收信息，包括丢包数、接收RTP数据包的最高序列号等，构造RTCP RR报文，发送回发送端。2）针对每一个到达的RTP数据包，执行基于到达时间延迟的码率估计算法，得到接收端估计码率，构造RTCP REMB报文，发送回发送端。

由此可见，GCC算法由发送端和接收端配合共同实现，接收端负责码率反馈数据的生成，发送端负责根据码率反馈数据计算目标码率，并生效于目标模块。本文接下来基于本节所述的GCC算法的四项子任务，分别详细分析之。


### 2 RTCP RR报文构造及收发
关于WebRTC上的RTP/RTCP协议的具体实现细节，可参考文章[3]。本节主要从RR报文的数据流角度，对其数据源、报文构造和收发进行分析。其数据源和报文构造如图2所示，报文接收和作用于码率控制模块如图3所示。

在数据接收端，RTP数据包从Network线程到达Worker线程，经过Call对象，VideoReceiveStream对象到达RtpStreamReceiver对象。在该对象中，主要执行三项任务：1)接收端码率估计；2) 转发RTP数据包到VCM模块；3)接收端数据统计。其中1)是下一节的重点，2)是RTP数据包进一步组帧和解码的地方；3)是统计RTP数据包接收信息，作为RTCP RR报文和其他数据统计模块的数据来源，是我们本节重点分析的部分。

在RtpStreamReceiver对象中，RTP数据包经过解析得到头部信息，作为输入参数调用ReceiveStatistianImpl::IncomingPacket()。该函数中分别调用UpdateCounters()和NotifyRtpCallback()，前者用来更新对象内部的统计信息，如接收数据包计数等，后者用来更新RTP回调对象的统计信息，该信息用来作为getStats调用的数据源。

![2844879-ed098a4958b2ee2c](http://image.smartjames.cn/mweb/20210105/16098246932212.png)
<center><font color="#808080" size="2">图2 RTCP RR报文数据源及报文构造</font></center>

RTCP发送模块在ModuleProcess线程中工作，RTCP报文周期性发送。当线程判断需要发送RTCP报文时，调用SendRTCP()进行发送。接下来调用PrepareReport()准备各类型RTCP报文的数据。对于我们关心的RR报文，会调用AddReportBlock()获取数据源并构造ReportBlock对象:该函数首先通过ReceiveStatistianImpl::GetStatistics()拿到类型为RtcpStatistics的数据源，然后以此填充ReportBlock对象。GetStatistics()会调用CalculateRtcpStatistics()计算ReportBlock的每一项数据，包括丢包数、接收数据包最高序列号等。ReportBlock对象会在接下来的报文构造环节通过BuildRR()进行序列化。RTCP报文进行序列化之后，交给Network线程进行网络层发送。

![2844879-ae80527d779c3402](http://image.smartjames.cn/mweb/20210105/16098247057859.png)
<center><font color="#808080" size="2">图3 RTCP RR报文接收及反馈</font></center>

在发送端(即RTCP报文接收端)，RTCP报文经过Network线程到达Worker线程，最后到达ModuleRtpRtcpImpl模块调用IncomingRtcpPacket()进行报文解析工作。解析完成以后，调用TriggerCallbacksFromRTCPPackets()反馈到回调模块。在码率估计方面，会反馈到BitrateController模块。ReportBlock消息最终会到达BitrateControllerImpl对象，进行下一步的目标码率确定。

至此，关于RTCP RR报文在拥塞控制中的执行流程分析完毕。

### 3 接收端基于延迟的码率估计
接收端基于数据包到达延迟的码率估计是整个GCC算法最复杂的部分，本节在分析WebRTC代码的基础上，阐述该部分的实现细节。

接收端基于延迟码率估计的基本思想是：RTP数据包的到达时间延迟m(i)反映网络拥塞状况。当延迟很小时，说明网络拥塞不严重，可以适当增大目标码率；当延迟变大时，说明网络拥塞变严重，需要减小目标码率；当延迟维持在一个低水平时，目标码率维持不变。其主要由三个模块组成：到达时间滤波器，过载检查器和速率控制器。

在实现上，WebRTC定义该模块为远端码率估计模块RemoteBitrateEstimator，整个模块的工作流程如图4所示。需要注意的是，该模块需要RTP报文扩展头部abs-send-time的支持，用以记录RTP数据包在发送端的绝对发送时间，详细请参考文献[4]。

![2844879-71027dc71720dd9c](http://image.smartjames.cn/mweb/20210105/16098247269312.png)
<center><font color="#808080" size="2">图4 GCC算法基于延迟的码率估计</font></center>

接收端收到RTP数据包后，经过一系列调用到RtpStreamReceiver对象，由该对象调用远端码率估计模块的总控对象RemoteBitrateEstimatorAbsSendTime，由该对象的总控函数IncomingPacketInfo()负责整个码率估计流程，如图4所示，算法从左到右依次调用子对象的功能函数。

总控函数首先调用InterArrival::ComputeDeltas()函数，用以计算相邻数据包组的到达时间相对延迟，该部分对应文章[1]的3.1节内容。在计算到达时间相对延迟时，用到了RTP报文头部扩展abs-send-time。另外，实现细节上要注意数据包组的划分，以及对乱序和突发时间的处理。

接下来算法调用OveruseEstimator::Update()函数，用以估计数据包的网络延迟，该部分对应文章[1]的3.2节内容。对网络延迟的估计用到了Kalman滤波，算法的具体细节请参考文章[2]。Kalman滤波的结果为网络延迟m(i)，作为下一阶段网络状态检测的输入参数。

算法接着调用OveruseDetector::Detect()，用来检测当前网络的拥塞状况，该部分对应文章[1]的3.2节内容。网络状态检测用当前网络延迟m(i)和阈值gamma_1进行比较，判断出overuse，underuse和normal三种网络状态之一。在算法细节上，要注意overuse的判定相对复杂一些：当m(i) > gamma_1时，计算处于当前状态的持续时间t(ou)，如果t(ou) > gamma_2，并且m(i) > m(i-1)，则发出网络过载信号Overuse。如果m(i)小于m(i-1)，即使高于阀值gamma_1也不需要发出过载信号。在判定网络拥塞状态之后，还要调用UpdateThreshold()更新阈值gamma_1。

算法接着调用AimdRateControl::Update()和UpdateBandwidthEstimate()函数，用以估计当前网络状态下的目标码率Ar，该部分对应文章[1]的3.3节。算法基于当前网络状态和码率变化趋势有限状态机，采用AIMD(Additive Increase Multiplicative Decrease)方法计算目标码率，具体计算公式请参考文章[1]。需要注意的是，当算法处于开始阶段时，会采用Multiplicative Increase方法快速增加码率，以加快码率估计速度。

此时，我们已经拿到接收端估计的目标码率Ar。接下来以Ar为参数调用VieRemb对象的OnReceiveBitrateChange()函数，发送REMB报文到发送端。REMB报文会推送到RTCP模块，并设置REMB报文发送时间为立即发送。关于REMB报文接下来的发送和接收流程，和第1节描述的RTCP报文一般处理流程是一样的，即经过序列化发送到网络，然后发送端收到以后，反序列化出描述结构，最后通过回调函数到达发送端码率控制模块BitrateControllerImpl。

至此，接收端基于延迟的码率估计过程描述完毕。

### 4 发送端码率计算及生效

在发送端，目标码率计算和生效是异步进行的，即Worker线程从RTCP接收模块经回调函数拿到丢包率和REMB码率之后，计算得到目标码率A；然后ModuleProcess线程异步把目标码率A生效到目标模块如PacedSender和ViEEncoder等。下面分别描述码率计算和生效过程。
![](http://image.smartjames.cn/mweb/20210105/16098247455890.png)
<center><font color="#808080" size="2">图5 发送端码率计算过程</font></center>

码率计算过程如图5所示：Worker线程从RTCPReceiver模块经过回调函数拿到RTCP RR报文和REMB报文的数据，到达BitrateController模块。RR报文中的丢包率会进入Update()函数中计算码率，码率计算公式如文章[1]第2节所述。然后算法流程进入CapBitrateToThreshold()函数，和配置的最大最小码率和远端估计码率进行比较后，确定最终目标码率。而REMB报文的接收端估计码率Ar则直接进入CapBitrateToThreshold()函数参与目标码率的确定。目标码率由文章[1]的3.4节所示公式进行确定。需要注意的是，RR报文和REMB报文一般不在同一个RTCP报文里。
![](http://image.smartjames.cn/mweb/20210105/16098247656332.png)
<center><font color="#808080" size="2">图6 发送端码率生效过程</font></center>

发送端码率生效过程如图6所示：ModuleProcess线程调用拥塞控制总控对象CongestionController周期性从码率控制模块BitrateControllerImpl中获取当前最新目标码率A，然后判断目标码率是否有变化。若是，则把最新目标码率设置到相关模块中，主要包括PacedSender模块，RTPSender模块和ViEEncoder模块。

对于PacedSender模块，设置码率主要是为了平滑RTP数据包的发送速率，尽量避免数据包Burst造成码率波动。对于RTPSender模块，设置码率是为了给NACK模块预留码率，如果预留码率过小，则在某些情况下对于NACK报文请求选择不响应。对于ViEEncoder模块，设置码率有两个用途：1)控制发送端丢帧策略，根据设定码率和漏桶算法决定是否丢弃当前帧。2)控制编码器内部码率控制，设定码率作为参数传输到编码器内部，参与内部码率控制过程。

至此，发送端码率计算和生效过程分析完毕。

### 5 总结
本文结合文章[1]，深入WebRTC代码内部，详细分析了WebRTC的GCC算法的实现细节。通过本文，对WebRTC的代码结构和拥塞控制实现细节有了更深层次的理解，为进一步学习WebRTC奠定良好基础。

参考文献  
[1] [WebRTC基于GCC的拥塞控制(上) – 算法分析](https://byronjia.github.io/2021/01/WebRTC%E5%9F%BA%E4%BA%8EGCC%E7%9A%84%E6%8B%A5%E5%A1%9E%E6%8E%A7%E5%88%B6-%E7%AE%97%E6%B3%95%E5%88%86%E6%9E%90/)

[2] [WebRTC视频接收缓冲区基于KalmanFilter的延迟模型.](http://www.jianshu.com/p/bb34995c549a)

[3] [WebRTC中RTP/RTCP协议实现分析](http://www.jianshu.com/p/c84be6f3ddf3)

[4] [abs-send-time.](https://webrtc.org/experiments/rtp-hdrext/abs-send-time/) 