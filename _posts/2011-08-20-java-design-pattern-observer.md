---
layout: post
title: Java中的观察者模式
categories: [编程, java]
tags: [observer, 设计模式, 观察者]
---


> 定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新

#### 一个例子

```java
public class Child extends Observable {
	private boolean wakeup;

	public void setWakeup(boolean wakeup) {
		this.wakeup = wakeup;
		setChanged();
		notifyObservers(new WakeupEvent());
	}

	public static void main(String[] args) {
		Child bean = new Child();
		bean.addObserver(new Mother());
		bean.setWakeup(true);
	}

}

class Mother implements Observer {

	public void update(Observable o, Object event) {
		if(event instanceof WakeupEvent){
			playWithChild();
		}
	}

	private void playWithChild() {
		
	}

}

class WakeupEvent{
	
}
```