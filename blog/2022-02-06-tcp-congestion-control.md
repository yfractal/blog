# TCP Congestion Control Brief Description by Pseudo Code

## Introduction
   TCP 需要在异构，动态的网络环境下，通过不可靠的下层服务(IP)，提供可靠（reliable）的服务，而拥塞控制是 TCP 必不可少的机制。

   TCP 拥塞控制本身是一个非常困难，也非常有趣的问题。

   本文主要根据 RFC 2001[1] 对 TCP 拥塞控制的状态变化进行一个简单的描述。

## TCP Congestion Control Brief Description
   TCP 拥塞控制通过调整 cwnd(congestion window) 对流量进行控制，未经过确认(ack)的数据总和不能超过 cwnd 的大小。

   TCP 拥塞控制有三个状态，在不同的状态下，会使用不同的方式调整 cwnd 的大小。

### State Changes
#### Slow Start and Congestion Avoidance
slow start 和 congestion avoidance 这两个状态的转换由 cwnd 和 ssthresh(slow start threshold) 控制。

当 cwnd > ssthresh 则由 slow start 转换为 congestion avoidance。

在  slow start 状态下，当收到新的 ack 的时候，cwnd 会成指数增长，而 congestion avoidance 状态下，则成线性增长。

在当前的状态是 slow start 或者是 congestion avoidance 的时候，如果收到 duplicate ACK, 则会将 ssthresh 设置为 window size( min{cwnd, rwnd} )的一半。

#### Fast Recovery
当收到 3 个或者更多的重复的 ACK 的时候，则进入 Fast Recovery 状态，在收到新的 ACK 的时候，则 Fast Recovery 结束，进入 Congestion Avoidance。

当进入 Fast Recovery 的时候，会将 threshold 设置为 window size( min{cwnd, rwnd} )的一半，重传丢失的 segment，将 cwnd 设置为 ssthresh + 3 倍的 segment size。

#### Timeout
当 retransmit timer timeout 的时候，则会将 ssthresh 设置为 window size( min{cwnd, rwnd} )的一半，并将 cwnd 设置为 segment size。即从新开始 slow start。

### How Congestion Window Changes
图片截取自 MIT 6829 Computer Networking L8 [4]
![Screen Shot 2022-02-06 at 6 26 13 PM](https://user-images.githubusercontent.com/3775525/152676638-f346dd6d-7d2c-4d8c-984c-d1b618085d94.png)

### Pseudo Code
TCPCongestionControl#receive 在收到 ack 或者 retransmit timer timeout 的时候会被触发。

可以理解成一种 callbck，在有收到 ack 的时候会被调用，在 timeout 事件发生的时候，也会被调用。类似于 Erlang 的 receive。

```ruby
class TCPCongestionControl
  # rwnd receiver's advertised window
  def initialize(segment_size)
    # segment size announced by the other end, or the default,
    # typically 536 or 512
    @segment_size = segment_size
    @cwnd = segment_size
    @ssthresh =  65535 # bytes
    @in_fast_recovery = false
    # other logic ....
  end

  def receive
    # indicate congestion
    if current_algorithm == :slow_start && new_ack?
      # exponential growth
      @cwnd += @segment_size
      # may go_to congestion_avoidance state
      # Slow start continues until TCP is halfway to where it was when congestion
      # occurred (since it recorded half of the window size that caused
      # the problem in step 2), and then congestion avoidance takes over.
    elsif current_algorithm == :congestion_avoidance && new_ack?
      # linear growth
      @cwnd += segsize * segsize / @cwnd
    elsif three_or_more_duplicate_ack?
      # TCP does not know whether a duplicate ACK is caused by a lost segment or just a reordering of segments
      # it waits for a small number and assume it is just a a reordering of segments
      # 3 or more duplicate ACKs are received in a row,
      # it is a strong indication that a segment has been lost
      # go_to fast recovery
      @in_fast_recovery = true
      @ssthresh = current_window_size / 2
      retransmit_missing # retransmit directly without waiting timer
      @cwnd = @ssthresh + 3 * @segment_size
    # This inflates the congestion window by the number of segments that have left the network and which the other end has cached (3).
    elsif current_algorithm == :fast_recovery && new_ack?
      @cwnd = @ssthresh # go_to congestion_avoidance
      @in_fast_recovery = false
    elsif (current_algorithm == :slow_start || current_algorithm == :congestion_avoidance) && duplicate_ack?
      @ssthresh = current_window_size / 2 # go_to congestion avoidance
    elsif current_algorithm == :fast_recovery && duplicate_ack?
      @cwnd += @segment_size
      # This inflates the congestion window for the additional segment that has left the network.  Transmit a packet, if allowed by the new value of cwnd.
    elsif timeout?
      # slow_star and congestion_avoidance should come into here
      # not sure when timout occurs when fast_recevery should come into here too...
      @ssthresh = current_window_size / 2
      @cwnd = segment_size # go_to slow start
    end
    
    maybe_transmit_packet
  end

  private
  def current_algorithm
    return :fast_recovery if @in_fast_recovery

    @cwnd <= @ssthresh ? :slow_start : :congestion_avoidance
  end

  def current_window_size
    min(@cwnd, @rwnd)
  end
end
```

## Relative
   Congestion Avoidance and Control[2]，对用塞控制背后的原理有详细的解释，不过比较难读（我只读懂了部分）。

   Computer Networking: A Top Down Approach 和 Principles of Computer System 都对 TCP 拥塞控有所描述。更深入的可以参考 MIT 的 Computer Networking 课程[3]。

   在操作系统中，也有类似的问题。之前，在 package 超过系统负载能力的时候，CPU 都被用在了处理中断，而没有时间执行应用层的逻辑，导致没办法完整的处理一个 package。和 Congestion Avoidance and Control 有相似的思路，但是用了不同的方法[4]。

   工业上，类似的机制有：流量控制(rate limiter)，背压(Back pressure)。Facebook 在 memcache cluster 中有使用类似的机制[5]。

   Netfix 有开源 Hystrix[6]，但现在已经被 concurrency-limits[7] 所替代，而 concurrency-limits 使用类似 TCP congestion control 的机制。

## 参考
   - [1] W. Stevens. RFC2001: TCP Slow Start, Congestion Avoidance, Fast Retransmit, and Fast Recovery Algorithms, 1997
   - [2] V. Jacobson, Congestion Avoidance and Control
   - [3] MIT 6829 Computer Networking L8 End-to-End Congestion Control
     https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-829-computer-networks-fall-2002/lecture-notes/
   - [4] Jeffrey Mogul. Eliminating Receive Livelock in an Interrupt-driven Kernel, 1996
   - [5] Rajesh Nishtala. Scaling Memcache at Facebook, 2013
   - [6] Netflix Hystrix https://github.com/Netflix/Hystrix
   - [7] Netflix concurrency-limits https://github.com/Netflix/concurrency-limits




