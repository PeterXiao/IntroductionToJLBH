
# JLBH - 引入 Java 延迟基准测试工具
[JLBH Examples 1 - Why Code Should be Benchmarked in Context](http://www.rationaljava.com/2016/04/jlbh-examples-1-why-code-should-be.html)

在这篇文章中：

* [使用 JMH 和 JLBH 进行日期序列化的并排示例](#首先是-jmh-基准测试)
* [在微基准测试中测量日期序列化](#我们使用-jlbh-运行测试得到几乎相同的结果)
* [测量日期序列化作为适当应用程序的一部分](#我们使用-jlbh-运行测试得到几乎相同的结果)
* [如何将探针添加到您的 JLBH 基准测试](#如上一篇文章所述jlbh-允许我们为代码的任何部分生成延迟配置文件我们将为第-3-阶段添加一个探针)
* [了解在上下文中测量代码的重要性](JLBH_Examples_2.md)


在上一篇文章“ JLBH 简介”中，我们介绍了 JLBH，这是Chronicle用来测试Chronicle-FIX 的延迟测试工具，现在可以开源使用。在接下来的几篇文章中，我们将查看一些示例应用程序： 示例的所有代码都可以在我的 GitHub 项目中找到：我在介绍 JLBH 时提出的要点之一是，对代码进行基准测试很重要语境。这意味着在尽可能接近现实生活中运行方式的环境中对代码进行基准测试。这篇文章在实践中证明了这一点。让我们看一个相对昂贵的 Java 操作——日期序列化——看看它需要多长时间：









## 首先是 JMH 基准测试：

'''

        package org.latency.serialisation.date;
        
        import net.openhft.affinity.Affinity;
        import net.openhft.chronicle.core.Jvm;
        import net.openhft.chronicle.core.OS;
        import org.openjdk.jmh.annotations.*;
        import org.openjdk.jmh.runner.Runner;
        import org.openjdk.jmh.runner.RunnerException;
        import org.openjdk.jmh.runner.options.Options;
        import org.openjdk.jmh.runner.options.OptionsBuilder;
        import org.openjdk.jmh.runner.options.TimeValue;
        
        import java.io.*;
        import java.lang.reflect.InvocationTargetException;
        import java.util.Date;
        import java.util.concurrent.TimeUnit;
        
        /**
        * Created to show the effects of running code within more complex code.
          * Date serialisation as a micro benchmark vs date serialisation inside a TCP call.
            */
            @State(Scope.Thread)
            public class DateSerialiseJMH {
            private final Date date = new Date();
        
            public static void main(String[] args) throws InvocationTargetException,
            IllegalAccessException, RunnerException, IOException, ClassNotFoundException {

         if (OS.isLinux())
             Affinity.setAffinity(2);

         if(Jvm.isDebug()){
             DateSerialiseJMH jmhParse = new DateSerialiseJMH();
             jmhParse.test();
         }
         else {
             Options opt = new OptionsBuilder()
                     .include(DateSerialiseJMH.class.getSimpleName())
                     .warmupIterations(6)
                     .forks(1)
                     .measurementIterations(5)
                     .mode(Mode.SampleTime)
                     .measurementTime(TimeValue.seconds(3))
                     .timeUnit(TimeUnit.MICROSECONDS)
                     .build();

             new Runner(opt).run();
         }
    }

    @Benchmark
    public Date test() throws IOException, ClassNotFoundException {
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(out);
    oos.writeObject(date);

         ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(out.toByteArray()));
         return (Date)ois.readObject();
      }
    }

  '''

在我的笔记本电脑 (MBP i7) 上运行这些是我得到的结果：

    Result "test":
    4.578 ±(99.9%) 0.046 us/op [Average]
    (min, avg, max) = (3.664, 4.578, 975.872), stdev = 6.320
    CI (99.9%): [4.533, 4.624] (assumes normal distribution)
    Samples, N = 206803
    mean =      4.578 ±(99.9%) 0.046 us/op
    min =      3.664 us/op
    p( 0.0000) =      3.664 us/op
    p(50.0000) =      4.096 us/op
    p(90.0000) =      5.608 us/op
    p(95.0000) =      5.776 us/op
    p(99.0000) =      8.432 us/op
    p(99.9000) =     24.742 us/op
    p(99.9900) =    113.362 us/op
    p(99.9990) =    847.245 us/op
    p(99.9999) =    975.872 us/op
    max =    975.872 us/op


    # Run complete. Total time: 00:00:21
    
    Benchmark                Mode     Cnt  Score   Error  Units
    
    DateSerialiseJMH.test  sample  206803  4.578 ± 0.046  us/op

操作的平均时间为 4.5us：

## 我们使用 JLBH 运行测试得到几乎相同的结果：


'''

        package org.latency.serialisation.date;
        import net.openhft.chronicle.core.jlbh.JLBHOptions;
        import net.openhft.chronicle.core.jlbh.JLBHTask;
        import net.openhft.chronicle.core.jlbh.JLBH;
        
        import java.io.*;
        import java.lang.reflect.InvocationTargetException;
        import java.util.Date;
        
        /**
        * Created to show the effects of running code within more complex code.
          * Date serialisation as a micro benchmark vs date serialisation inside a TCP call.
            */
        public class DateSerialisedJLBHTask implements JLBHTask {
        private Date date = new Date();
        private JLBH lth;
        public static void main(String[] args) throws InvocationTargetException, IllegalAccessException, IOException, ClassNotFoundException {
            JLBHOptions jlbhOptions = new JLBHOptions()
                        .warmUpIterations(400_000)
                        .iterations(1_000_000)
                        .throughput(100_000)
                        .runs(3)
                        .recordOSJitter(true)
                        .accountForCoordinatedOmmission(true)
                        .jlbhTask(new DateSerialisedJLBHTask());
            new JLBH(jlbhOptions).start();
        }
        @Override
                    public void run(long startTimeNS) {
            try {
                ByteArrayOutputStream out = new ByteArrayOutputStream();
                ObjectOutputStream oos = new ObjectOutputStream(out);
                oos.writeObject(date);
                ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(out.toByteArray()));
                date = (Date)ois.readObject();
                lth.sample(System.nanoTime() - startTimeNS);
            }
            catch (IOException | ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
        @Override
                    public void init(JLBH lth) {
            this.lth = lth;
        }
    }    
        
  '''

这些是结果：


    Warm up complete (400000 iterations took 2.934s)
    -------------------------------- BENCHMARK RESULTS (RUN 1) ---------
    Run time: 10.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,000,000)                         50/90 99/99.9 99.99/99.999 - worst was 4.2 / 5.8  352 / 672  803 / 901 - 934
    OS Jitter (13,939)                              50/90 99/99.9 99.99 - worst was 8.4 / 17  639 / 4,130  12,850 - 20,450
    --------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 2) ---------
    Run time: 10.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,000,000)                         50/90 99/99.9 99.99/99.999 - worst was 4.2 / 5.8  434 / 705  836 / 934 - 967
    OS Jitter (11,016)                              50/90 99/99.9 99.99 - worst was 8.4 / 17  606 / 770  868 - 1,340
    --------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 3) ---------
    Run time: 10.0s
    Correcting for co-ordinated:true
    Target throughput:100000/s = 1 message every 10us
    End to End: (1,000,000)                         50/90 99/99.9 99.99/99.999 - worst was 4.2 / 5.8  434 / 737  901 / 999 - 1,030
    OS Jitter (12,319)                              50/90 99/99.9 99.99 - worst was 8.4 / 15  573 / 737  803 - 901

--------------------------------------------------------------- SUMMARY (end to end)---------------

| Percentile | run1   | run2   | run3   | % Variation |
|------------|--------|--------|--------|-------------|
| 50:        | 4.22   | 4.22   | 4.22   | 0.00        |
| 90:        | 5.76   | 5.76   | 5.76   | 0.00        |
| 99:        | 352.26 | 434.18 | 434.18 | 0.00        |
| 99.9:      | 671.74 | 704.51 | 737.28 | 3.01        |
| 99.99:     | 802.82 | 835.58 | 901.12 | 4.97        |
| worst:     | 901.12 | 933.89 | 999.42 | 4.47        |
--------------------------------------------------------------------




操作的平均时间为 4.2us：

注意：在这种情况下，使用 JLBH 比使用 JMH 没有优势。我只是将代码作为比较。

现在我们将运行完全相同的操作，但在 TCP 调用中，代码将像这样工作：
1. 客户端通过 TCP 环回（本地主机）向服务器发送修复消息
2. 服务器读取消息
3. 服务器进行日期序列化
4. 服务器返回消息给客户端

## 如上一篇文章所述，JLBH 允许我们为代码的任何部分生成延迟配置文件。我们将为第 3 阶段添加一个探针。

'''

        package org.latency.serialisation.date;
        
        import net.openhft.affinity.Affinity;
        import net.openhft.chronicle.core.Jvm;
        import net.openhft.chronicle.core.jlbh.JLBHOptions;
        import net.openhft.chronicle.core.jlbh.JLBHTask;
        import net.openhft.chronicle.core.jlbh.JLBH;
        import net.openhft.chronicle.core.util.NanoSampler;
        
        import java.io.*;
        import java.net.InetSocketAddress;
        import java.nio.ByteBuffer;
        import java.nio.ByteOrder;
        import java.nio.channels.ServerSocketChannel;
        import java.nio.channels.SocketChannel;
        import java.util.Date;
        
        /**
        * Created to show the effects of running code within more complex code.
          * Date serialisation as a micro benchmark vs date serialisation inside a TCP call.
            */
           public class DateSerialiseJLBHTcpTask implements JLBHTask {
        private final static int port = 8007;
        private static final Boolean BLOCKING = false;
        private final int SERVER_CPU = Integer.getInteger("server.cpu", 0);
        private Date date = new Date();
        private JLBH lth;
        private ByteBuffer bb;
        private SocketChannel socket;
        private byte[] fixMessageBytes;
        private NanoSampler dateProbe;
        public static void main(String[] args) {
            JLBHOptions lth = new JLBHOptions()
                        .warmUpIterations(50_000)
                        .iterations(100_000)
                        .throughput(20_000)
                        .runs(3)
                        .recordOSJitter(true)
                        .accountForCoordinatedOmmission(true)
                        .jlbhTask(new DateSerialiseJLBHTcpTask());
            new JLBH(lth).start();
        }
        @Override
                    public void init(JLBH lth) {
            this.lth = lth;
            dateProbe = lth.addProbe("date serialisation ");
            try {
                runServer(port);
                Jvm.pause(200);
                socket = SocketChannel.open(new InetSocketAddress(port));
                socket.socket().setTcpNoDelay(true);
                socket.configureBlocking(BLOCKING);
            }
            catch (IOException e) {
                e.printStackTrace();
            }
            String fixMessage = "8=FIX.4.2u00019=211u000135=Du000134=3u000149=MY-INITIATOR-SERVICEu000152=20160229-" +
                                     "09:04:14.459u000156=MY-ACCEPTOR-SERVICEu00011=ABCTEST1u000111=863913604164909u000121=3u000122=5" +
                                     "u000138=1u000140=2u000144=200u000148=LCOM1u000154=1u000155=LCOM1u000159=0u000160=20160229-09:" +
                                     "04:14.459u0001167=FUTu0001200=201106u000110=144u0001n";
                     fixMessageBytes = fixMessage.getBytes();
                     int length = fixMessageBytes.length;
                     bb = ByteBuffer.allocateDirect(length).order(ByteOrder.nativeOrder());
                     bb.put(fixMessageBytes);
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
                             ByteBuffer bb = ByteBuffer.allocateDirect(32 * 1024).order(ByteOrder.nativeOrder());
                             for (; ; ) {
                                 bb.limit(12);
                                 do {
                                     if (socket.read(bb) < 0)
                                         throw new EOFException();
                                 } while (bb.remaining() > 0);
                                 int length = bb.getInt(0);
                                 bb.limit(length);
                                 do {
                                     if (socket.read(bb) < 0)
                                         throw new EOFException();
                                 } while (bb.remaining() > 0);
                                 long now = System.nanoTime();
                                 try {
                                     //Running the date serialisation but this time inside the TCP callback.
                                     ByteArrayOutputStream out = new ByteArrayOutputStream();
                                     ObjectOutputStream oos = new ObjectOutputStream(out);
                                     oos.writeObject(date);
                                     ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(out.toByteArray()));
                                     date = (Date)ois.readObject();
                                     dateProbe.sampleNanos(System.nanoTime() - now);
                                 } catch (IOException | ClassNotFoundException e) {
                                     e.printStackTrace();
                                 }
                                 bb.flip();
                                 if (socket.write(bb) < 0)
                                     throw new EOFException();
                                 bb.clear();
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
                @Override
                public void run(long startTimeNs) {
                bb.position(0);
                bb.putInt(bb.remaining());
                bb.putLong(startTimeNs);
                bb.position(0);
                writeAll(socket, bb);
                     bb.position(0);
                     try {
                         readAll(socket, bb);
                     } catch (IOException e) {
                         e.printStackTrace();
                     }
                     bb.flip();
                     if (bb.getInt(0) != fixMessageBytes.length) {
                         throw new AssertionError("read error");
                     }
                     lth.sample(System.nanoTime() - startTimeNs);
                }
                private static void readAll(SocketChannel socket, ByteBuffer bb) throws IOException {
                   bb.clear();
                   do {
                       if (socket.read(bb) < 0)  throw new EOFException();

                      }  while (bb.remaining() > 0);
                  }

               private static void writeAll(SocketChannel socket, ByteBuffer bb) {
                 try {
                        while (bb.remaining() > 0 && socket.write(bb) >= 0) ;

                     } catch (IOException e) {
                    e.printStackTrace();
                        }
                    }
                }

'''

这次的结果是这样的：

    Warm up complete (50000 iterations took 3.83s)

    -------------------------------- BENCHMARK RESULTS (RUN 1) ------------------------
    Run time: 6.712s
    Correcting for co-ordinated:true
    Target throughput:20000/s = 1 message every 50us
    End to End: (100,000)                           50/90 99/99.9 99.99 - worst was 822,080 / 1,509,950  1,711,280 / 1,711,280  1,711,280 - 1,711,280
    date serialisation  (100,000)                   50/90 99/99.9 99.99 - worst was 11 / 19  31 / 50  901 - 2,420
    OS Jitter (64,973)                              50/90 99/99.9 99.99 - worst was 8.1 / 16  40 / 1,540  4,850 - 18,350
    --------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 2) ---------
    Run time: 6.373s
    Correcting for co-ordinated:true
    Target throughput:20000/s = 1 message every 50us
    End to End: (100,000)                           50/90 99/99.9 99.99 - worst was 1,107,300 / 1,375,730  1,375,730 / 1,375,730  1,375,730 - 1,375,730
    date serialisation  (100,000)                   50/90 99/99.9 99.99 - worst was 11 / 19  29 / 52  901 - 1,670
    OS Jitter (40,677)                              50/90 99/99.9 99.99 - worst was 8.4 / 16  34 / 209  934 - 1,470
    --------------------------------------------------------------------
    -------------------------------- BENCHMARK RESULTS (RUN 3) ---------
    Run time: 5.333s
    Correcting for co-ordinated:true
    Target throughput:20000/s = 1 message every 50us
    End to End: (100,000)                           50/90 99/99.9 99.99 - worst was 55,570 / 293,600  343,930 / 343,930  343,930 - 343,930
    date serialisation  (100,000)                   50/90 99/99.9 99.99 - worst was 9.0 / 16  26 / 38  770 - 1,030
    OS Jitter (32,042)                              50/90 99/99.9 99.99 - worst was 9.0 / 13  22 / 58  737 - 934
    --------------------------------------------------------------------

-------------------------------- SUMMARY (end to end)---------------

| Percentile | run1       | run2       | run3      | % Variation |
|------------|------------|------------|-----------|-------------|
| 50:        | 822083.58  | 1107296.26 | 55574.53  | 92.66       |
| 90:        | 1509949.44 | 1375731.71 | 293601.28 | 71.07       |
| 99:        | 1711276.03 | 1375731.71 | 343932.93 | 66.67       |
| 99.9:      | 1711276.03 | 1375731.71 | 343932.93 | 66.67       |
| 99.99:     | 1711276.03 | 1375731.71 | 343932.93 | 66.67       |
| worst:     | 1711276.03 | 1375731.71 | 343932.93 | 66.67       |
--------------------------------------------------------------------
-------------------------------- SUMMARY (date serialisation )------

| Percentile | run1    | run2    | run3    | % Variation |
|------------|---------|---------|---------|-------------|
| 50:        | 11.01   | 11.01   | 8.96    | 13.22       |
| 90:        | 18.94   | 18.94   | 15.62   | 12.44       |
| 99:        | 31.23   | 29.18   | 26.11   | 7.27        |
| 99.9:      | 50.18   | 52.22   | 37.89   | 20.14       |
| 99.99:     | 901.12  | 901.12  | 770.05  | 10.19       |
| worst:     | 2424.83 | 1671.17 | 1032.19 | 29.21       |
|
--------------------------------------------------------------------


看出，完全相同的日期序列化时间是原来的两倍从 ~4.5us 到 ~10us。
这里并不是要详细说明为什么代码在上下文中运行时需要更长时间的原因，而是与在调用日期序列化之间填充 CPU 缓存有关。

当我们运行的（如在微基准测试中）是日期序列化时，它可以很好地放入 CPU 缓存中，永远不需要清除。但是，当对 Date 序列化的调用之间存在间隙时，
操作代码将被清除并需要重新加载。JLBH 允许您在上下文中对代码进行基准测试，这是延迟基准测试的一个重要部分。