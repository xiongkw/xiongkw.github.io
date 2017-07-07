---
layout: post
title: Java中Object的wait和notify
categories: [编程, java]
tags: [java, wait, notify, 多线程]
---

> 前台MM那里有很多办公用品，有人喜欢借胶水，而我现在要借剪刀。但MM同一时间只能接待一个码农，所以大家都要抢着锁定MM。   
> 不要问我为什么大家都喜欢找MM借东西。

```java
public class WaitNotifyDemo {
    //前台MM
    private static Object mm = new Object();
    //剪刀只有一把，所以需要确认有没有被人借走
    private static boolean isShearReturned = false;

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(()->{
            for(;;) {
                //锁定MM，防止被别人打扰
                synchronized (mm) {
                    if (!isShearReturned) {
                        try {
                            System.out.println("剪刀被人借走了，等一下再来问");
                            mm.wait();//回座位等，这里是我等，不是MM等
                            System.out.println("MM说她那里没人了，可以过去问她剪刀还回来没有(别问我为什么不直接在QQ上问)");
                            continue;
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("终于借到剪刀了，赶紧去开快递!");
                    break;
                }
            }
        });
        executorService.execute(()->{
            //锁定MM，防止被别人打扰
            synchronized (mm){
                try {
                    System.out.println("某A借胶水，顺便还和MM聊了一会，让我们其它人等了3秒.");
                    Thread.sleep(3000);
                    mm.notifyAll();//MM在QQ群里说，要找我的可以过来了
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        executorService.execute(()->{
            synchronized (mm){
                try {
                    System.out.println("某B还剪刀，又和MM聊天，花了2秒时间.");
                    Thread.sleep(2000);
                    WaitNotifyTest.isShearReturned = true;
                    mm.notifyAll();//MM在QQ群里说，要找我的可以过来了
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        executorService.shutdown();
    }

}
```