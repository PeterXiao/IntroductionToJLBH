
# JLBH 示例 2 - 协调遗漏的会计处理
[JLBH Examples 2 - Accounting for Coordinated Omission](http://www.rationaljava.com/2016/04/jlbh-examples-2-accounting-for.html)

在这篇文章中：

* [在考虑和不考虑协调遗漏的情况下运行 JLBH]()
* [协调遗漏影响的数字示例]()
* [关于流量控制的讨论]()


这是我在描述如果不考虑协调遗漏的情况下进行测量会是什么样子时使用的示例：假设您正在等火车，但由于前面的火车晚点而在车站耽误了一个小时。然后让我们想象一下，你上火车晚了一个小时，而火车通常需要半个小时才能到达目的地。
如果您不考虑协调遗漏，您将不会认为自己遭受了任何延误，因为即使您在出发前在车站等了一个小时，您的旅程也花费了正确的时间！
但这正是您在运行微基准测试时所做的。您为每个“旅程”计时，而不是等待时间。


事实是，这对于微型基准测试来说绝对没问题。但是当你想测量应用程序的延迟时，
它就不行了。默认情况下，JLBH 测量端到端时间，考虑协调遗漏，尽管您确实有一个设置来衡量它而不考虑协调遗漏。 
我写了这个简单的基准来展示协调遗漏的影响有多大。在这个例子中，我们在每 10k 次迭代后添加一个毫秒延迟：

'''

        package org.latency.spike;
        import net.openhft.chronicle.core.Jvm;
        import net.openhft.chronicle.core.jlbh.JLBH;
        import net.openhft.chronicle.core.jlbh.JLBHOptions;
        import net.openhft.chronicle.core.jlbh.JLBHTask;
        /**
        * A simple JLBH example to show the effects od accounting for co-ordinated omission.
          * Toggle the accountForCoordinatedOmission to see results.
          */
          public class SimpleSpikeJLBHTask implements JLBHTask {
          private int count = 0;
          private JLBH lth;
          public static void main(String[] args){
              JLBHOptions lth = new JLBHOptions()
                  .warmUpIterations(40_000)
                  .iterations(1_100_000)
                  .throughput(100_000)
                  .runs(3)
                  .recordOSJitter(true)
                  .accountForCoordinatedOmmission(true)
                  .jlbhTask(new SimpleSpikeJLBHTask());
              new JLBH(lth).start();
          }
          @Override
          public void run(long startTimeNS) {
                    if((count++)%10_000==0){
                    //pause a while
                    Jvm.busyWaitMicros(1000);
          }
                lth.sample(System.nanoTime() - startTimeNS);
          }
          @Override
          public void init(JLBH lth) {
                this.lth = lth;
            }

          }        
   

'''





如果您设置coordinatedOmission(false)那么您将获得此配置文件 - 正如预期的那样，毫秒延迟只能在最高百分位数上看到，
从第 99.99 个百分位数向上。或者这样说它只会影响每 10k 次迭代中的一次————————这并不奇怪。


    Warm up complete (40000 iterations took 0.046s)
    -------------------------------- BENCHMARK RESULTS (RUN 1) -----------
    Run time: 11.593s
    Correcting for co-ordinated:false
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.11 / 0.13  0.20 / 0.33  999 / 999 - 1,930
    OS Jitter (14,986)                              50/90 99/99.9 99.99 - worst was 8.4 / 15  68 / 1,080  3,210 - 4,330
    ----------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 2) -----------
    Run time: 11.49s
    Correcting for co-ordinated:false
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.11 / 0.13  0.16 / 0.28  999 / 999 - 999
    OS Jitter (13,181)                              50/90 99/99.9 99.99 - worst was 8.4 / 12  36 / 62  270 - 573
    ----------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 3) -----------
    Run time: 11.494s
    Correcting for co-ordinated:false
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.11 / 0.13  0.16 / 0.26  999 / 999 - 1,030
    OS Jitter (13,899)                              50/90 99/99.9 99.99 - worst was 8.4 / 13  42 / 76  160 - 541
----------------------------------------------------------------------
-------------------------------- SUMMARY (end to end)-----------------

| Percentile | run1    | run2   | run3    | % Variation |
|------------|---------|--------|---------|-------------|
| 50:        | 0.11    | 0.11   | 0.11    | 0.00        |
| 90:        | 0.13    | 0.13   | 0.13    | 0.00        |
| 99:        | 0.20    | 0.16   | 0.16    | 3.31        |
| 99.9:      | 0.33    | 0.28   | 0.26    | 3.88        |
| 99.99:     | 999.42  | 999.42 | 999.42  | 0.00        |
| 99.999:    | 999.42  | 999.42 | 999.42  | 0.00        |
| worst:     | 1933.31 | 999.42 | 1032.19 | 2.14        |

----------------------------------------------------------------------

But if you set coordinatedOmission(true)you see the true effect of this delay.


    Warm up complete (40000 iterations took 0.044s)
    -------------------------------- BENCHMARK RESULTS (RUN 1) -----------
    Run time: 11.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.11 / 0.17  385 / 1,930  4,590 / 5,370 - 5,370
    OS Jitter (13,605)                              50/90 99/99.9 99.99 - worst was 8.4 / 15  68 / 1,080  5,110 - 5,900
    ----------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 2) -----------
    Run time: 11.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.12 / 0.18  42 / 901  999 / 999 - 1,030
    OS Jitter (13,156)                              50/90 99/99.9 99.99 - worst was 8.4 / 13  38 / 68  209 - 467
    ----------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 3) -----------
    Run time: 11.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,100,000)                         50/90 99/99.9 99.99/99.999 - worst was 0.12 / 0.18  46 / 901  999 / 999 - 999
    OS Jitter (13,890)                              50/90 99/99.9 99.99 - worst was 8.4 / 14  44 / 80  250 - 1,870
----------------------------------------------------------------------

-------------------------------- SUMMARY (end to end)-----------------

| Percentile | run1    | run2    | run3   | % Variation |
|------------|---------|---------|--------|-------------|
| 50:        | 0.11    | 0.12    | 0.12   | 0.00        |
| 90:        | 0.17    | 0.18    | 0.18   | 0.00        |
| 99:        | 385.02  | 41.98   | 46.08  | 6.11        |
| 99.9:      | 1933.31 | 901.12  | 901.12 | 0.00        |
| 99.99:     | 4587.52 | 999.42  | 999.42 | 0.00        |
| 99.999:    | 5373.95 | 999.42  | 999.42 | 0.00        |
| worst:     | 5373.95 | 1032.19 | 999.42 | 2.14        |

----------------------------------------------------------------------



事实上，一百分之一（不是万分之一）的迭代在某种程度上受到影响。当您提高百分位数时，您还可以看到延迟的渐进效应。这在数字上清楚地证明了为什么协调遗漏必须成为基准测试的重要组成部分，尤其是当您无法在程序中施加流量控制时。流量控制是在您没有跟上时停止消费的能力，例如，如果您太忙，则将用户赶出您的站点。Fix Engines 不能施加流量控制，即你不能告诉市场放慢速度，因为你跟不上！施加流量控制的程序以消费者为中心，而不施加流量控制的程序以生产者为中心。





考虑协调遗漏与能够为定义的吞吐量设置延迟密切相关，这是我们将在下一个示例中看到的内容。
