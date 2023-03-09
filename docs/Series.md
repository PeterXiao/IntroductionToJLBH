JLBH（Java Latency Benchmark Harness）系列文章

# [A Series of Posts on JLBH (Java Latency Benchmark Harness)](http://www.rationaljava.com/2016/04/a-series-of-posts-on-jlbh-java-latency.html)



在 JLBH 上介绍一组 5 篇博客文章这些文章组对任何负责创建或基准测试真实 Java 应用程序的人都应该有用。
除了介绍开源 JLBH 基准测试工具之外，我们还通过实际代码示例探讨了支持 Java 中延迟基准测试的一些微妙之处。




1)  [JLBH - 引入 Java 延迟基准测试工具](Intro.md)
  
    * [什么是JLBH](Intro.md#什么是-jlbh-)
    * [JLBH 的动机是什么](Intro.md#我们为什么写-jlbh)
    * [JMH 和 JLBH 的区别](Intro.md#jmh-和-jlbh-之间的区别)
    * [快速入门指南](Intro.md#快速入门指南)

2) [JLBH 示例 1 - 为什么代码应该在上下文中进行基准测试](JLBH_Examples_1.md)
  * [使用 JMH 和 JLBH 进行日期序列化的并排示例](JLBH_Examples_1.md)
  * [在微基准测试中测量日期序列化](JLBH_Examples_1.md)
  * [测量日期序列化作为适当应用程序的一部分](JLBH_Examples_1.md)
  * [如何将探针添加到您的 JLBH 基准测试](JLBH_Examples_1.md)
  * [了解在上下文中测量代码的重要性](JLBH_Examples_1.md)

3) [JLBH 示例 2 - 协调遗漏的会计处理](JLBH_Examples_2.md)
  * [在考虑和不考虑协调遗漏的情况下运行 JLBH](JLBH_Examples_2.md)
  * [协调遗漏影响的数字示例](JLBH_Examples_2.md)
  * [关于流量控制的讨论](JLBH_Examples_2.md)

4) [JLBH 示例 3 - 吞吐量对延迟的影响](JLBH_Examples_3.md)
   * [关于吞吐量对延迟的影响的讨论](JLBH_Examples_3.md)
   * [如何使用JLBH测量TCP环回](JLBH_Examples_3.md)
   * [添加探针以测试 TCP 往返的两半](JLBH_Examples_3.md)
   * [观察增加吞吐量对延迟的影响](JLBH_Examples_3.md)
   * [了解您必须降低吞吐量才能在高百分位数下实现良好的延迟。](JLBH_Examples_3.md)

5) [JLBH 示例 4 - QuickFix 与 ChronicleFix 的基准测试](JLBH_Examples_4.md)
   * [使用 JLBH 对 QuickFIX 进行基准测试](JLBH_Examples_4.md)
   * [观察 QuickFix 延迟如何通过百分位数降低](JLBH_Examples_4.md)
   * [比较 QuickFIX 和 Chronicle FIX](JLBH_Examples_4.md)