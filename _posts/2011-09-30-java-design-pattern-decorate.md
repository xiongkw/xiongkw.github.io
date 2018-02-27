---
layout: post
title: Java中的装饰模式
categories: [编程, java]
tags: [decorate, 设计模式, 装饰]
---


> 动态地给一个对象添加一些额外的职责。就增加功能来说，装饰器模式相比生成子类更为灵活

#### 1. 场景 

一支画笔，能设置其粗细属性，现在要求增加色彩属性

#### 2. Drawable接口
```java
public interface Drawable{
    void draw();
}
```

#### 3. 粗细画笔
```java
public class WeightDrawer implements Drawable {
    private int weight;
    
	public void draw() {
		// draw
	}

}
```

#### 4. 颜色画笔
```java
public class ColorDrawer implements Drawable {
	private String color;
	
	// 被装饰对象
	private Drawable drawable;

    public ColorDrawer(Drawable drawable, String color){
        this.color = color;
        this.drawable = drawable;
    }

    public void draw(){
        // set color
        
        // 调用被装饰对象
        drawable.draw();
    }
    
}

```

#### 5. main
```java
public static void main(String[] args){
    Drawable weightDrawer = new WeightDrawer(10);
    Drawable colorDrawer = new ColorDrawer(weightDrawer, "red");
    colorDrawer.draw();
}
```

#### 6. 总结

* `装饰模式`实际是通过组合方式包装被代理对象，以实现行为增强的功能
* `java IO`中大量使用了`装饰模式`
* `代理模式`和`装饰模式`实现上比较相似，其不同点是`代理`是原对象不方便处理而指定一个代理(比如我们办证时如果本从不方便去办证中心，可以通过公证指定某人代为办理)，而`装饰`则侧重于功能的增强