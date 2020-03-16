---
layout: post
title: chrome插件使用tesseract.js识别图片文字
categories: [编程, javascript]
tags: [tesseract, ocr]
---


> `chrome`插件使用`tesseract.js`识别图片文字

#### 1. 目录结构

```
js
--|tesseract-core.wasm.js
--|tesseract.min.js
--|test.js
--|worker.min.js
traineddata
--|eng.trainedata.gz
background.html
icon.png
manifest.json
test.png
```

> 下载`tesseract.js`相关`js`库和`traineddata`

#### 2. 清单文件(manifest.json)

```json
{
    "manifest_version": 2,
    "name": "Demo",
    "version": "1.0.0",
    "description": "Demo",
    "background":
    {
        "page": "background.html"
    },
	"web_accessible_resources": [
        "js/worker.min.js",
        "js/tessearct-core.wasm.js",
        "traineddata/*.traineddata.gz",
        "test.jpg"
    ]
}
```

#### 3. 后台页面(background.html)

```html
<!DOCTYPE html>
<html lang="zh">
<head>
	<meta charset="utf-8" />
    <meta http-equiv="Content-Type" content="text/html;charset=utf-8" />
    <title>Demo</title>
    <script src="js/tesseract.min.js"></script>
    <script src="js/test.js"></script>
</head>
<body>
</body>
</html>
```

#### 4. test.js

```js
const { createWorker } = Tesseract;
const worker = createWorker({
    workerPath: chrome.extension.getURL('js/worker.min.js'),
    langPath: chrome.extension.getURL('traineddata'),
    corePath: chrome.extension.getURL('js/tesseract-core.wasm.js'),
});

const decode = async (img) => {
    const { data: { text } } = await worker.recognize(img, 'eng');
    console.log(text);
};

(async () => {
    await worker.load();
    await worker.loadLanguage('eng');
    await worker.initialize('eng');
    let url = chrome.extension.getURL("test.jpg");
    await decode(url);
})();

```

#### 5. 运行

安装插件，打开`background.html`，可在`console`中看到解析出的文字

#### 6. 参考

* [Tesseract.js](https://github.com/naptha/tesseract.js#tesseractjs)