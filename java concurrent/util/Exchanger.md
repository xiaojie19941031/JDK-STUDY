# Exchanger
定义

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并发工具类,用于两个线程之间进行数据交换,提供一个同步点,在这个同步点上两个线程可以交换数据[只能是两个线程之间交换数据]

应用场景

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当两个线程需要将自己的结果或者某个状态提交给另外一个线程时使用[例如校对方面的工作,两个线程去执行然后对比各自的结果]

原理还在研究中