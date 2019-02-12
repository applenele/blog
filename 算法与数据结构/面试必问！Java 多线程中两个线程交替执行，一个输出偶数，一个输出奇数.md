## 前言

楼主今天在面经上看到这个题，挺有意思，小小的题目对多线程的考量还挺多。大部分同学都会使用 synchronized 来实现。楼主今天带来另外两种优化实现，让你面试的时候，傲视群雄！

## 第一种 synchronized

```java
class ThreadPrintDemo2 {
  public static void main(String[] args) {
    final ThreadPrintDemo2 demo2 = new ThreadPrintDemo2();
    Thread t1 = new Thread(demo2::print1);
    Thread t2 = new Thread(demo2::print2);

    t1.start();
    t2.start();
  }

  public synchronized void print2() {
    for (int i = 1; i <= 100; i += 2) {
      System.out.println(i);
      this.notify();
      try {
        this.wait();
        Thread.sleep(100);// 防止打印速度过快导致混乱
      } catch (InterruptedException e) {
        // NO
      }
    }
  }

  public synchronized void print1() {
    for (int i = 0; i <= 100; i += 2) {
      System.out.println(i);
      this.notify();
      try {
        this.wait();
        Thread.sleep(100);// 防止打印速度过快导致混乱
      } catch (InterruptedException e) {
        // NO
      }
    }
  }
}
```

通过 synchronized 同步两个方法，每次只能有一个线程进入，每打印一个数，就释放锁，另一个线程进入，拿到锁，打印，唤醒另一个线程，然后挂起自己。循环反复，实现了一个最基本的打印功能。

但，如果你这么写，面试官肯定是不满意的。楼主将介绍一种更好的实现。

## 使用 CAS 实现

```java
public class ThreadPrintDemo {

  static AtomicInteger cxsNum = new AtomicInteger(0);
  static volatile boolean flag = false;

  public static void main(String[] args) {

    Thread t1 = new Thread(() -> {
      for (; 100 > cxsNum.get(); ) {
        if (!flag && (cxsNum.get() == 0 || cxsNum.incrementAndGet() % 2 == 0)) {
          try {
            Thread.sleep(100);// 防止打印速度过快导致混乱
          } catch (InterruptedException e) {
            //NO
          }

          System.out.println(cxsNum.get());
          flag = true;
        }
      }
    }
    );

    Thread t2 = new Thread(() -> {
      for (; 100 > cxsNum.get(); ) {
        if (flag && (cxsNum.incrementAndGet() % 2 != 0)) {
          try {
            Thread.sleep(100);// 防止打印速度过快导致混乱
          } catch (InterruptedException e) {
            //NO
          }

          System.out.println(cxsNum.get());
          flag = false;
        }
      }
    }
    );

    t1.start();
    t2.start();
  }
}
```

我们通过使用 CAS，避免线程的上下文切换，然后呢，使用一个 volatile 的 boolean 变量，保证不会出现可见性问题，记住，这个 flag 一定要是 volatile 的，如果不是，可能你的程序运行起来没问题，但最终一定会出问题，而且面试官会立马鄙视你。

这样就消除了使用 synchronized 导致的上下文切换带来的损耗，性能更好。相信，如果你面试的时候，这么写，面试官肯定很满意。

但，我们还有性能更好的。



## 使用 volatile

```java
class ThreadPrintDemo3{

  static volatile int num = 0;
  static volatile boolean flag = false;

  public static void main(String[] args) {

    Thread t1 = new Thread(() -> {
      for (; 100 > num; ) {
        if (!flag && (num == 0 || ++num % 2 == 0)) {

          try {
            Thread.sleep(100);// 防止打印速度过快导致混乱
          } catch (InterruptedException e) {
            //NO
          }

          System.out.println(num);
          flag = true;
        }
      }
    }
    );

    Thread t2 = new Thread(() -> {
      for (; 100 > num; ) {
        if (flag && (++num % 2 != 0)) {

          try {
            Thread.sleep(100);// 防止打印速度过快导致混乱
          } catch (InterruptedException e) {
            //NO
          }

          System.out.println(num);
          flag = false;
        }
      }
    }
    );

    t1.start();
    t2.start();

  }
}
```

我们使用 volatile 变量代替 CAS 变量，减轻使用 CAS 的消耗，注意，这里 ++num 不是原子的，但不妨碍，因为有 flag 变量控制。而 num 必须是 volatile 的，如果不是，会导致可见性问题。



## 引用

https://www.cnblogs.com/stateis0/p/9091254.html

