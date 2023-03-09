
# JLBH 示例 3 - 吞吐量对延迟的影响
[JLBH Examples 3 - The Affects of Throughput on Latency](http://www.rationaljava.com/2016/04/jlbh-examples-3-affects-of-throughput.html)


在这篇文章中：

关于吞吐量对延迟的影响的讨论
如何使用JLBH测量TCP环回
添加探针以测试 TCP 往返的两半
观察增加吞吐量对延迟的影响
了解您必须降低吞吐量才能在高百分位数下实现良好的延迟。


在这篇文章中，我们看到了考虑协调遗漏或衡量延迟一次迭代对后续迭代的影响的影响。直觉上我们理解吞吐量会影响延迟。当我们提高吞吐量时，我们也会提高延迟， 这似乎很自然。进入一个非常拥挤的商店会影响您选择和购买商品的速度。另一方面，考虑一家很少光顾的商店。可能是在这样的商店里，店主不在收银台喝茶休息，你的购买会被推迟，因为你要等他放下茶杯，走到他可以为你服务的柜台. 这正是您在运行基准测试和改变吞吐量时所发现的。







一般来说，当你提高吞吐量时，延迟会增加，但在某些时候，当吞吐量下降到某个阈值以下时，延迟也会增加。下面的代码通过环回对往返 TCP 调用进行计时。我们添加两个探针：





* client2server - 完成前半部分往返所花费的时间
* server2client - 完成行程的后半部分所花费的时间

这些探测不考虑协调遗漏——只有端到端时间才考虑协调遗漏。

这是基准测试的代码：

'''

        package org.latency.tcp;
        
        import net.openhft.affinity.Affinity;
        import net.openhft.chronicle.core.Jvm;
        import net.openhft.chronicle.core.jlbh.JLBHOptions;
        import net.openhft.chronicle.core.jlbh.JLBHTask;
        import net.openhft.chronicle.core.jlbh.JLBH;
        import net.openhft.chronicle.core.util.NanoSampler;
        
        import java.io.EOFException;
        import java.io.IOException;
        import java.net.InetSocketAddress;
        import java.nio.ByteBuffer;
        import java.nio.ByteOrder;
        import java.nio.channels.ServerSocketChannel;
        import java.nio.channels.SocketChannel;
        
        public class TcpBenchmark implements JLBHTask {
        private final static int port = 8007;
        private static final boolean BLOCKING = false;
        private final int SERVER_CPU = Integer.getInteger("server.cpu", 0);
        private JLBH jlbh;
        private ByteBuffer bb;
        private SocketChannel socket;
        private NanoSampler client2serverProbe;
        private NanoSampler server2clientProbe;
        
            public static void main(String[] args) {
                JLBHOptions jlbhOptions = new JLBHOptions()
                        .warmUpIterations(50000)
                        .iterations(50000)
                        .throughput(20000)
                        .runs(5)
                        .jlbhTask(new TcpBenchmark());
                new JLBH(jlbhOptions).start();
            }
        
            @Override
            public void init(JLBH jlbh) {
                this.jlbh = jlbh;
                client2serverProbe = jlbh.addProbe("client2server");
                server2clientProbe = jlbh.addProbe("server2clientProbe");
                try {
                    runServer(port);
                    Jvm.pause(200);
        
                    socket = SocketChannel.open(new InetSocketAddress(port));
                    socket.socket().setTcpNoDelay(true);
                    socket.configureBlocking(BLOCKING);
        
                } catch (IOException e) {
                    e.printStackTrace();
                }
        
                bb = ByteBuffer.allocateDirect(8).order(ByteOrder.nativeOrder());
            }
        
            private void runServer(int port) throws IOException {
        
                new Thread(() -> {
                    if (SERVER_CPU > 0) {
                        System.out.println("server cpu: " + SERVER_CPU);
                        Affinity.setAffinity(SERVER_CPU);
                    }
                    ServerSocketChannel ssc = null;
                    SocketChannel socket = null;
                    try {
                        ssc = ServerSocketChannel.open();
                        ssc.bind(new InetSocketAddress(port));
                        System.out.println("listening on " + ssc);
        
                        socket = ssc.accept();
                        socket.socket().setTcpNoDelay(true);
                        socket.configureBlocking(BLOCKING);
        
                        System.out.println("Connected " + socket);
        
                        ByteBuffer bb = ByteBuffer.allocateDirect(8).order(ByteOrder.nativeOrder());
                        while (true) {
                            readAll(socket, bb);
        
                            bb.flip();
                            long time = System.nanoTime();
                            client2serverProbe.sampleNanos(time - bb.getLong());
                            bb.clear();
                            bb.putLong(time);
                            bb.flip();
        
                            writeAll(socket, bb);
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("... disconnected " + socket);
                        try {
                            if (ssc != null)
                                ssc.close();
                        } catch (IOException ignored) {
                        }
                        try {
                            if (socket != null)
                                socket.close();
                        } catch (IOException ignored) {
                        }
                    }
                }, "server").start();
        
            }
        
            private static void readAll(SocketChannel socket, ByteBuffer bb) throws IOException {
                bb.clear();
                do {
                    if (socket.read(bb) < 0)
                        throw new EOFException();
                } while (bb.remaining() > 0);
            }
        
            @Override
            public void run(long startTimeNs) {
                bb.position(0);
                bb.putLong(System.nanoTime());
                bb.position(0);
                writeAll(socket, bb);
        
                bb.position(0);
                try {
                    readAll(socket, bb);
                    server2clientProbe.sampleNanos(System.nanoTime() - bb.getLong(0));
                } catch (IOException e) {
                    e.printStackTrace();
                }
        
                jlbh.sample(System.nanoTime() - startTimeNs);
            }
        
            private static void writeAll(SocketChannel socket, ByteBuffer bb) {
                try {
                    while (bb.remaining() > 0 && socket.write(bb) >= 0) ;
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        
            @Override
            public void complete() {
                System.exit(0);
            }
        }


'''

以下是在 20,000 次迭代/秒的吞吐量下运行时的结果：

Warm up complete (50000 iterations took 2.296s)
-------------------------------- BENCHMARK RESULTS (RUN 1) ---------Run time: 2.5s
Correcting for co-ordinated:true
Target throughput:20000/s = 1 message every 50us
End to End: (50,000)                            50/90 99/99.9 99.99 - worst was 34 / 2,950  19,400 / 20,450  20,450 - 20,450
client2server (50,000)                          50/90 99/99.9 99.99 - worst was 16 / 26  38 / 72  287 - 336
server2clientProbe (50,000)                     50/90 99/99.9 99.99 - worst was 16 / 27  40 / 76  319 - 901
OS Jitter (26,960)                              50/90 99/99.9 99.99 - worst was 9.0 / 16  44 / 1,340  10,220 - 11,800
--------------------------------------------------------------------
-------------------------------- BENCHMARK RESULTS (RUN 2) ---------
Run time: 2.5s
Correcting for co-ordinated:true
Target throughput:20000/s = 1 message every 50us
End to End: (50,000)                            50/90 99/99.9 99.99 - worst was 42 / 868  4,590 / 5,110  5,370 - 5,370
client2server (50,000)                          50/90 99/99.9 99.99 - worst was 20 / 27  38 / 92  573 - 2,560
server2clientProbe (50,000)                     50/90 99/99.9 99.99 - worst was 19 / 27  38 / 72  868 - 1,740
OS Jitter (13,314)                              50/90 99/99.9 99.99 - worst was 9.0 / 16  32 / 96  303 - 672
--------------------------------------------------------------------
-------------------------------- BENCHMARK RESULTS (RUN 3) ---------
Run time: 2.5s
Correcting for co-ordinated:true
Target throughput:20000/s = 1 message every 50us
End to End: (50,000)                            50/90 99/99.9 99.99 - worst was 34 / 152  999 / 2,160  2,290 - 2,290
client2server (50,000)                          50/90 99/99.9 99.99 - worst was 17 / 26  36 / 54  201 - 901
server2clientProbe (50,000)                     50/90 99/99.9 99.99 - worst was 16 / 25  36 / 50  225 - 1,740
OS Jitter (14,306)                              50/90 99/99.9 99.99 - worst was 9.0 / 15  23 / 44  160 - 184
---------------------------------------------------------------------------------------------------- SUMMARY (end to end)---------------
Percentile   run1         run2         run3      % Variation   var(log)
50:            33.79        41.98        33.79        13.91       
90:          2949.12       868.35       151.55        75.92       
99:         19398.66      4587.52       999.42        70.53       
99.9:       20447.23      5111.81      2162.69        47.62     99.99:      20447.23      5373.95      2293.76        47.24       
worst:      20447.23      5373.95      2293.76        47.24
--------------------------------------------------------------------
-------------------------------- SUMMARY (client2server)------------
Percentile   run1         run2         run3      % Variation   
50:            16.13        19.97        16.90        10.81       
90:            26.11        27.14        26.11         2.55       
99:            37.89        37.89        35.84         3.67       
99.9:          71.68        92.16        54.27        31.76       
99.99:        286.72       573.44       200.70        55.32       
worst:        335.87      2555.90       901.12        55.04
--------------------------------------------------------------------
-------------------------------- SUMMARY (server2clientProbe)-------
Percentile   run1         run2         run3      % Variation   
50:            16.13        18.94        16.13        10.43       
90:            27.14        27.14        25.09         5.16       
99:            39.94        37.89        35.84         3.67       
99.9:          75.78        71.68        50.18        22.22       
99.99:        319.49       868.35       225.28        65.55       
worst:        901.12      1736.70      1736.70         0.00
--------------------------------------------------------------------







应该发生的是：
client2server + server2client ~= endToEnd


这或多或少发生在第 50 个百分位 

出于本演示的目的进行第二次运行：

19.97 + 18.94 ~= 41.98

如果这就是您测量的全部，您可能会说运行 20k/s 的消息不会有问题我的机器。

然而，我的笔记本电脑显然无法处理这个体积，如果我们再次查看第 90 个百分位的第二次运行。

27.14 + 27.14 !~=  868.35

当你向上移动百分位数时，它变得越来越糟......

但是如果我将吞吐量更改为每秒 5k 条消息，
我会在第 90 个百分位数上看到这个：

32.23 + 29.38 ~= 62.46

所以我们看到如果你想在高百分位数上实现低延迟，你必须将吞吐量降低到正确的水平。





# 这就是为什么我们能够使用 JLBH 改变吞吐量如此重要。






