# 现在有 T1、T2、T3 三个线程,怎样保证 T2 在 T1 执行完后执行T3 在 T2 执行完

问题：现在有T1、T2、T3三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行

实现：使用Thread中的join方法实现

分析：

1. `Thread`类中的`join`方法是用来同步的，底层其实是调用了 `wait`方法。先来看一下演示代码：

```java
package com.whh.concurrency;
/**
 *@description:
 * 问题：现在有 T1、T2、T3 三个线程,怎样保证 T2 在 T1 执行完后执行T3在T2执行完
 * 分析：使用join方法实现
 *@author:wenhuohuo
 */
public class MyJoin {
    public static void main(String[] args) {
        final Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("线程1");
            }
        },"t1");
        final Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t1.join();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println("线程2");
            }
        },"t2");
        final Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t2.join();
                }catch (Exception e){
                    e.printStackTrace();
                }
                System.out.println("线程3");
            }
        },"t3");
        t3.start();
        t2.start();
        t1.start();
    }
}
```

执行结果：

```
线程3
线程2
线程1
```

可以看到，我们让`t2`线程调用`t1.join`,`t3`调用`t2.join`，尽管是t3，t2，t1分别start，执行顺序还是t1，t2，t3。是因为`join`方法底层使用的是`wait`方法。

2. 查看`join`方法源码

```java
public final void join() throws InterruptedException {
        join(0); //传入的是毫秒值
    }
```

```java
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) { //isAlive()是native方法，判断线程是否还存活
                wait(0);		//wait(0),不计时间，一直等待，直到有notify()或notifyAll()来唤醒。
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);//传入时间，表示在时间值消耗完之前一直等待，直到过了等待时间。
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

1）从源码中，我们结合之前的代码分析，`t2.join()`和`t3.join()`,均没有传值，相当于`join(0)`,表示不计时间，`t2`会一直`wait`等待`t1`执行完成，`t3`会一直`wait`等待`t2`执行完成。所以执行结果顺序是t3，t2，t1。

2）当传入的毫秒值不为0时，就一直循环等待，直到过了等待时间(dalay<=0)，则执行break方法，那么将不再等待。

4. 改变`join()`传入的毫秒值，查看执行顺序并分析结果：

```java
public class MyJoin {
    public static void main(String[] args) {
        final Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //处理业务时间，模拟为8秒
                    Thread.sleep(8000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("线程1");
            }
        },"t1");
        final Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t1.join(4000); //t2等待t1线程4秒
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println("线程2");
            }
        },"t2");
        final Thread t3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    t2.join(2000); //t3等待t2线程2秒
                }catch (Exception e){
                    e.printStackTrace();
                }
                System.out.println("线程3");
            }
        },"t3");
        t3.start();
        t2.start();
        t1.start();
    }
}
```

执行结果：

```java
线程3	//程序启动过了2秒执行t3
线程2 //过了4秒执行t2
线程1 //过了8秒执行t1
```

分析：我们让`t1` 睡眠8秒模拟业务执行时间，t2等待t1 的时间为4秒，t3等待t2的时间为2秒。那么当t1，t2，t3启动后，等待的时间，t3会因为t2的等待时间4秒太长而先与t2执行，t2会因为t1的8秒太长而先与t1执行。

