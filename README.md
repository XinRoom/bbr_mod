# BBR 改版方案



## 推荐阅读

* TCP拥塞控制及BBR原理分析
* BBR是个什么鬼？系列
* TCP BBR算法的带宽敏感性以及高丢包率下的优化
* 来自Google的TCP BBR拥塞控制算法解析
* BBR及其在实时音视频领域的应用 （对bbr的一些问题给了点建议）



## 仅对主要参数修改

> 以应对较高丢包率的情形（loss 25%）



### ProbeRTT

```c
/* Window length of bw filter (in rounds): */
static const int bbr_bw_rtts = CYCLE_LEN + 2; 
/* Window length of min_rtt filter (in sec): */
static const u32 bbr_min_rtt_win_sec = 10;
```

拥塞排空之后会进入探测带宽阶段，探测最大带宽的方法是在10（**bbr_bw_rtts**）个RTT中观测到的最大带宽，以此数据作为最大带宽。如果10s（**bbr_min_rtt_win_sec**）没有得到最小RTT，超时之后需要继续探测最小RTT。探测最小RTT需要尽量避免网络拥堵，降低拥塞窗口，发送比较少的报文。**（BBR的ProbeRTT阶段只发4个包，发送速率下降太多会引发延迟加大和卡顿问题）**

**ProbeRTT: **探测物理RTTProp。当10s内没有发现最小RTTProp时，就要进入ProbeRTT状态。在ProbeRTT状态，仅发4pkts/RTT(接近停止发送)，从而排空链路上的数据包，测量真实的RTTProp

所以bbr_min_rtt_win_sec可以适当增加为15~20

```c
/* Window length of bw filter (in rounds): */
static const int bbr_bw_rtts = CYCLE_LEN + 4; 
/* Window length of min_rtt filter (in sec): */
static const u32 bbr_min_rtt_win_sec = 15;
```

也可以对ProbeRTT下固定降到4个包进行修改，变为降至50%：

https://github.com/google/bbr/blob/v2alpha/net/ipv4/tcp_bbr2.c#L321





### bbr_pacing_gain

```c
static const int bbr_pacing_gain[] = {
	BBR_UNIT * 5 / 4,	/* probe for more available bw */
	BBR_UNIT * 3 / 4,	/* drain queue and/or yield bw to other flows */
	BBR_UNIT, BBR_UNIT, BBR_UNIT,	/* cruise at 1.0*bw to utilize pipe, */
	BBR_UNIT, BBR_UNIT, BBR_UNIT	/* without creating excess queue... */
};
```

这是一组带宽增益数组，可以更激进一点：

```c
static const int bbr_pacing_gain[] = {
	BBR_UNIT * 11 / 8,	/* probe for more available bw */
	BBR_UNIT * 3 / 4,	/* drain queue and/or yield bw to other flows */
	BBR_UNIT * 9 / 8, BBR_UNIT, BBR_UNIT * 10 / 9,	/* cruise at 1.0*bw to utilize pipe, */
	BBR_UNIT, BBR_UNIT, BBR_UNIT	/* without creating excess queue... */
};
```







## 编译和生效

* 针对当前系统的内核，可以通过`apt install linux-source`在`/usr/src/`下获得当前版本的内核源代码压缩包，解压后，在`net/ipv4/tcp_bbr.c`原版文件的基础上进行适配
* 提示：
  * 本仓库tcp_bbr_mod为针对4.15.0内核版本的修改
  * 修改部分详见https://github.com/XinRoom/bbr_mod/commit/ba333386f128a596891bf96058bdbd876d2f9889

```bash
mkdir bbr_mod && cd bbr_mod

//++上传tcp_bbr_mod.c

echo 'obj-m:=tcp_bbr_mod.o' >./Makefile
make -C /lib/modules/$(uname -r)/build M=`pwd` modules
cp -rf ./tcp_bbr_mod.ko /lib/modules/$(uname -r)/kernel/net/ipv4
depmod -a
modprobe tcp_bbr_mod
lsmod |grep -q 'bbr'

//建议reboot
```





***也可参考其它名为 BBR魔改 的仓库代码***