---
layout: post
title: JavaFX中通过WebView实现javascript和java的相互调用
categories: [编程, java]
tags: [javafx, webview, javascript]
---

> 2008年`sun`公司推出`JavaFX`，旨在取代`swing/awt`

#### 一个例子

`java`代码
```java
public class MyApp extends Application {
    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage stage) throws Exception {
        stage.setWidth(1366);
        stage.setHeight(768);
        stage.setTitle("WebView Test");
        stage.initStyle(StageStyle.UNDECORATED);
        stage.centerOnScreen();
        WebView browser = new WebView();
        WebEngine engine = browser.getEngine();
        engine.getLoadWorker().stateProperty().addListener(new ChangeListener<Worker.State>() {
            @Override
            public void changed(ObservableValue<? extends Worker.State> observable, Worker.State oldValue, Worker.State newValue) {
                if (newValue == Worker.State.SUCCEEDED) {
                    engine.executeScript("sayHello('java')");
                    JSObject window = (JSObject) engine.executeScript("window");
                    window.setMember("app", new MyApp());
                }
            }
        });
        URL url = MyApp.class.getResource("/index.html");
        engine.load(url.toString());
        Scene scene = new Scene(browser);
        stage.setScene(scene);
        stage.show();
    }

    public void sayHello(String by) {
        System.out.println("Hello WebView by " + by);
    }
}
```

`html`代码
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>WebView Test</title>
    <script>
        function sayHello(by) {
            document.getElementById("hhello").innerHTML = "Hello WebView by" + by;
        }
        
        function callJava() {
            app.sayHello('javascript')
        }
    </script>
</head>
<body>
<h1 id="hhello"></h1>
<button onclick="callJava()">Call java</button>
</body>
</html>
```

说明：

* 在`java`中通过`WebEngine.executeScript`执行`javascript`脚本
* 通过`engine.executeScript("window")`获取`window`对象
* 通过`window.setMember("app", new MyApp())`给`window`对象添加属性`app`
* `javascript`中通过`window.app`直接操作`app`对象