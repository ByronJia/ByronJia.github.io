---
layout: post
title: WebRTC GCC拥塞控制详解
date: 2021-01-11
tag: webRTC
---

webrtc版本： M74

### 接收端
主要实现:  
- RemoteBitrateEstimatorAbsSendTime：基于abs-send-time的接收端带宽估计，默认方式，本文只描述该方式；
- RemoteBitrateEstimatorSingleStream：基于RTP时间戳的接收端带宽估计；
- InterArrival：到达时间滤波器，计算包簇(帧)之间的时间差；
- OveruseEstimator：卡尔曼滤波器，用于估算网络延迟变化(拥塞)的最优值；
- OveruseDetector：过载检测器，检测链路是否过载；
- RateStatistics：输入码率统计；
- AimdRateControl：加增乘减码率控制器。

#### 5.1 基本流程
接收端的带宽预测原则是以输入码率为基准，根据链路的拥塞情况进行调整，而链路的拥塞情况用延迟的变化来体现

![](http://image.smartjames.cn/mweb/20210111/16103494197594.png)

如上图所示，RTP包进入InterArrival(到达时间滤波器)后，InterArrival会缓存最近的俩个5ms的包簇(TimestampGroup)，当缓存够俩个包簇后，计算着俩个包簇之间的发送时间差、到达时间差、尺寸差，输出给OveruseEstimator(卡尔曼滤波器)，在没有网络拥塞的情况下，网络拥塞值=到达时间差-发送时间差=0，  如果网络拥塞值>0,则可能是因为网络噪声造成了拥塞，或者数据尺寸变化造成了网络排队，如果网络拥塞值<0, 则网络拥塞情况在好转，所以OveruseEstimator就以InterArrival的网络拥塞值、数据尺寸差为观测值，结合预测值进行拟合估算，计算出网络拥塞值的最优值，输出给OveruseDetector(过载检测器)，OveruseDetector根据拥塞值（抖动）来判断当前网络是否处于拥塞状态，将拥塞状态反馈到OveruseDetector并输出给AimdRateControl(加增乘减码率控制器)， AimdRateControl使用输入码率和网络拥塞状态根据”加增乘减“原则来调整接收端当前的预估码率。

#### 5.2 到达时间滤波器  -  InterArrival 

在不考虑NTP时间同步问题的情况下，一个数据在网络上的传输时间t=到达时间tr-发送时间ts,如果网络没有阻塞，同样的数据在相同的链路下的传输时间基本是不会变化的，如果传输时间变化，可能是以下原因：
- 网络播放、噪声
- 数据大小变化、排队情况变化(缓冲区)

2个相同数据之间传输时间的差异可以被定义为拥塞值：
```
t_ts_delta = t2 - t1 = (tr2-ts2) - (tr1-ts1) = (tr2-tr1)-(ts2-ts1) = t_delta - ts_delta
```
也就说拥塞值等于2个相同数据的到达时间差-发送时间差，经过这样的变换后NTP同步问题被解决，接下来考虑：
- 数据处理的频率
- 突发数据的处理
- 数据大小变化的影响

为降低处理频率，interArrival以5ms的数据为一组存储成一个包簇(TimestampGroup),缓存最近的2个包簇，计算着2个包簇之间的发送时间差、接收时间差，并根据数据到达速度来判断数据突发，避免处理短时间到来的大量数据。为了消除数据大小变化的影响，InterArrival还输出2个包簇的尺寸差，将这些值作为输入参数传给卡尔曼滤波器处理。

包簇结构：

```
struct TimestampGroup {
    size_t size;  // 总大小
    uint32_t first_timestamp;  // 第1个包的发送时间戳
    uint32_t timestamp;  // 最新包的发送时间戳作为包簇时间戳
    int64_t first_arrival_ms;  // 第1个包的接收时间
    int64_t complete_time_ms;  // 最新包的接收时间作为包簇接收时间
    int64_t last_system_time_ms;  // 最新包的当前处理时间
  };
```

计算包簇发送时间差、接收时间差、尺寸差：

```
bool InterArrival::ComputeDeltas(uint32_t timestamp,				// 输入，时间戳(abs-send-time或者RTP时间戳)
                                 int64_t arrival_time_ms,			// 输入，RTP包接收时间
                                 int64_t system_time_ms,			// 输入，RTP包当前处理时间
                                 size_t packet_size,				// 输入，RTP包大小
                                 uint32_t* timestamp_delta,			// 输出，发送时间差
                                 int64_t* arrival_time_delta_ms,	// 输出，接收时间差
                                 int* packet_size_delta) {			// 输出，尺寸差
  assert(timestamp_delta != NULL);
  assert(arrival_time_delta_ms != NULL);
  assert(packet_size_delta != NULL);
  bool calculated_deltas = false;
  // 如果是第一个包
  if (current_timestamp_group_.IsFirstPacket()) {
    // We don't have enough data to update the filter, so we store it until we
    // have two frames of data to process.
    current_timestamp_group_.timestamp = timestamp;  // 第一个包时间戳作为包簇时间戳
    current_timestamp_group_.first_timestamp = timestamp;  // 第一个包时间戳
    current_timestamp_group_.first_arrival_ms = arrival_time_ms;  // 第一个包到达时间
  } else if (!PacketInOrder(timestamp)) {  // 如果包乱序了，根据时间戳来大致估算
    return false;  // 本次计算失败
  } else if (NewTimestampGroup(arrival_time_ms, timestamp)) {  // 如果1个包簇满了，要创建新的包簇
    // First packet of a later frame, the previous frame sample is ready.
    // 如果有上个包簇
    if (prev_timestamp_group_.complete_time_ms >= 0) {
      // 发送时间差 = 当前包簇发送时间戳 - 上个包簇发送时间戳
      *timestamp_delta =
          current_timestamp_group_.timestamp - prev_timestamp_group_.timestamp;
      // 接收时间差 - 当前包簇接收时间 - 上个包簇接收时间
      *arrival_time_delta_ms = current_timestamp_group_.complete_time_ms -
                               prev_timestamp_group_.complete_time_ms;
      // Check system time differences to see if we have an unproportional jump
      // in arrival time. In that case reset the inter-arrival computations.
      // 检查接收时间和处理时间是否有过大偏差(3s)，容错
      int64_t system_time_delta_ms =
          current_timestamp_group_.last_system_time_ms -
          prev_timestamp_group_.last_system_time_ms;
      if (*arrival_time_delta_ms - system_time_delta_ms >=
          kArrivalTimeOffsetThresholdMs) {
        RTC_LOG(LS_WARNING)
            << "The arrival time clock offset has changed (diff = "
            << *arrival_time_delta_ms - system_time_delta_ms
            << " ms), resetting.";
        Reset();  // 重置
        return false;
      }
      // 从Socket上报的数据先发后至？乱序？
      if (*arrival_time_delta_ms < 0) {
        // The group of packets has been reordered since receiving its local
        // arrival timestamp.
        ++num_consecutive_reordered_packets_;
        if (num_consecutive_reordered_packets_ >= kReorderedResetThreshold) {
          RTC_LOG(LS_WARNING)
              << "Packets are being reordered on the path from the "
                 "socket to the bandwidth estimator. Ignoring this "
                 "packet for bandwidth estimation, resetting.";
          Reset();
        }
        return false;
      } else {
        num_consecutive_reordered_packets_ = 0;
      }
      assert(*arrival_time_delta_ms >= 0);
      // 尺寸差 = 当前包簇尺寸 - 上个包簇尺寸
      *packet_size_delta = static_cast<int>(current_timestamp_group_.size) -
                           static_cast<int>(prev_timestamp_group_.size);
     // 本次计算成功
      calculated_deltas = true;
    }

	// 当前的包簇改为上个包簇
    prev_timestamp_group_ = current_timestamp_group_;
    // The new timestamp is now the current frame.
    // 重置当前包簇，记录当前包簇的第一个包的时间戳
    current_timestamp_group_.first_timestamp = timestamp;  // 第一个包发送时间戳
    current_timestamp_group_.timestamp = timestamp;  // 当前包簇发送时间戳
    current_timestamp_group_.first_arrival_ms = arrival_time_ms;  // 第一个包到达时间
    current_timestamp_group_.size = 0;  // 当前包簇尺寸清零
  } else {
  	// 如果不满1个包簇，设置当前包簇时间戳为较新的时间戳
    current_timestamp_group_.timestamp =
        LatestTimestamp(current_timestamp_group_.timestamp, timestamp);
  }
  // Accumulate the frame size.
  // 累计当前包簇大小
  current_timestamp_group_.size += packet_size;
  // 更新当前包簇的接收时间为最新包接收时间
  current_timestamp_group_.complete_time_ms = arrival_time_ms;
  // 更新当前包簇的最新包处理时间
  current_timestamp_group_.last_system_time_ms = system_time_ms;

  return calculated_deltas;
}
```

判断当前RTP包到达后当前包簇是否已满5ms并需创建新的包簇

```
bool InterArrival::NewTimestampGroup(int64_t arrival_time_ms,
                                     uint32_t timestamp) const {
  if (current_timestamp_group_.IsFirstPacket()) {  // 第一个包
    return false;  // 不创建新包簇
  } else if (BelongsToBurst(arrival_time_ms, timestamp)) {  // 突发，不创建新包簇
    return false;
  } else {
    // 当前时间戳 - 当前包簇的第一个包时间戳 = 当前包簇总时长，是否大于kTimestampGroupLengthTicks个时间戳单位(90 * 5，也就是5ms)
    uint32_t timestamp_diff =
        timestamp - current_timestamp_group_.first_timestamp;
    return timestamp_diff > kTimestampGroupLengthTicks;
  }
}
```

判断当前RTP包是否属于突发：

```
bool InterArrival::BelongsToBurst(int64_t arrival_time_ms,
                                  uint32_t timestamp) const {
  // 默认使能
  if (!burst_grouping_) {
    return false;
  }
  assert(current_timestamp_group_.complete_time_ms >= 0);
  // 当前包距当前包簇最后1个包的接收时间差
  int64_t arrival_time_delta_ms =
      arrival_time_ms - current_timestamp_group_.complete_time_ms;
  // 当前包距当前包簇最后1个包的发送时间差
  uint32_t timestamp_diff = timestamp - current_timestamp_group_.timestamp;
  // 发送时间差换算成ms
  int64_t ts_delta_ms = timestamp_to_ms_coeff_ * timestamp_diff + 0.5;
  // 时间戳几乎一致，是突发
  if (ts_delta_ms == 0)
    return true;
  // 传输时延 = 接收时间差 - 发送时间差
  int propagation_delta_ms = arrival_time_delta_ms - ts_delta_ms;
  if (propagation_delta_ms < 0 &&  // 如果传输时延 < 0
      arrival_time_delta_ms <= kBurstDeltaThresholdMs &&  // 如果接收时间差 < 5ms
      arrival_time_ms - current_timestamp_group_.first_arrival_ms <
          kMaxBurstDurationMs)  // 如果当前包簇还不满100ms
    return true;  // 满足这些条件为突发
  return false;
}
```

#### 5.3 过载估计其 - OveruseEstimator
OveruseEstimator是一个卡尔曼滤波器，建立一个线性系统，输入观测值，计算预测值，用观测值修正预测值得到最优值。
具体到webRTC 输入值为2个包簇的发送时间差与接收时间差的差。 t_ts_delta = t_delta - ts_delta 。 以及2个包簇的尺寸差、链路拥塞情况，输出为最优的到达时间间隔增量(拥塞值)。
![20200602221057991](http://image.smartjames.cn/mweb/20210111/16103783764777.png)

#### 5.4 过载检测器 - OveruseDetector
OveruseDetector根据OveruseEstimator输出的链路延迟变化的情况，与特点的阈值比较，来检测链路是否处于过载状态，并反馈给OveruseEstimator

```
BandwidthUsage OveruseDetector::Detect(double offset,		// 链路延迟变化(到达时间间隔增量)
                                       double ts_delta,		// 两个包簇发送时间差
                                       int num_of_deltas,	// 估算使用的采样数
                                       int64_t now_ms) {	// 当前时间
  if (num_of_deltas < 2) {
    return BandwidthUsage::kBwNormal;
  }
  // 最多kMaxNumDeltas = 60个采样，计算总的网络拥塞值 = 采样数 * 单个采样的到达时间间隔增量
  const double T = std::min(num_of_deltas, kMaxNumDeltas) * offset;
  BWE_TEST_LOGGING_PLOT(1, "T", now_ms, T);
  BWE_TEST_LOGGING_PLOT(1, "threshold", now_ms, threshold_);
  // 如果网络拥塞值超过阈值，拥塞在加剧
  if (T > threshold_) {
    if (time_over_using_ == -1) {
      // Initialize the timer. Assume that we've been
      // over-using half of the time since the previous
      // sample.
      // 刚开始过载，过载时间设置为两个包簇的发送时间差/2
      time_over_using_ = ts_delta / 2;
    } else {
      // Increment timer
      // 持续过载，增加过载时间
      time_over_using_ += ts_delta;
    }
    //  过载计数++
    overuse_counter_++;
    // 如果过载时间超过100ms，并且发生了持续的过载
    if (time_over_using_ > overusing_time_threshold_ && overuse_counter_ > 1) {
      // 如果到达间隔变大了，也就是数据来的越来越慢
      if (offset >= prev_offset_) {
        time_over_using_ = 0;
        overuse_counter_ = 0;
        // 设置为过载状态
        hypothesis_ = BandwidthUsage::kBwOverusing;
      }
    }
  } else if (T < -threshold_) {  // 如果拥塞值 < 负的阈值，网络处于欠载状态，数据来的越来越快
    time_over_using_ = -1;
    overuse_counter_ = 0;
    // 设置为欠载状态
    hypothesis_ = BandwidthUsage::kBwUnderusing;
  } else {
    time_over_using_ = -1;
    overuse_counter_ = 0;
    // 设置为正常状态
    hypothesis_ = BandwidthUsage::kBwNormal;
  }
  // 保存当前的到达间隔增量
  prev_offset_ = offset;

  // 更新阈值
  UpdateThreshold(T, now_ms);

  return hypothesis_;
}
```