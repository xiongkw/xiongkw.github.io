---
layout: post
title: Java中的Observer
categories: [编程, java]
tags: [observer]
---


> `java`提供了观察者模式框架

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