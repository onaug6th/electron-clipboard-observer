# electron-clipboard-observer

## 前言

因为要完成剪贴板历史记录功能，经过查阅electron剪贴板文档。没有发现有剪贴板变化的钩子事件，于是决定自己实现一个。

`electron` 中有读取剪贴板内容API，可以借助此API封装监听定时器方法。具体逻辑如下：

使用定时器不断判断内容是否相同，如不相同。执行剪贴板改变回调

编写一剪贴板观察者类，实例化时传入监听的消息/图片变化回调、由于频繁获取剪贴板内容可能会带来性能负担，所以还需要支持传入自定义监听间隔时间。

返回的实例应该支持暂停和继续监听的方法。

## 编写剪贴板观察者类

首先定义构造函数参数的接口

```ts
import { clipboard, NativeImage } from 'electron';

interface Options {
  //  定时器获取内容判断的间隔
  duration?: number;
  //  文本变化回调
  textChange?: (text: string, beforeText: string) => void;
  //  图片变化回调
  imageChange?: (image: NativeImage, beforeImage: NativeImage) => void;
}
```

接着定义类，编写构造器中处理传参的逻辑与暴露的api

```ts
class ClipboardObserver {
  //  定时器
  timer: NodeJS.Timeout;
  //  变化前的文本（用于对比新获取的剪贴板文本）
  beforeText: string;
  //  变化去的图片（用于对比新获取的剪贴板图片）
  beforeImage: NativeImage;

  duration = 500;
  textChange: (text: string, beforeText: string) => void;
  imageChange: (image: NativeImage, beforeImage: NativeImage) => void;

  constructor(options: Options) {
    //  获取传入配置并保存到属性中
    const { duration, textChange, imageChange } = options;

    this.duration = duration;
    this.textChange = textChange;
    this.imageChange = imageChange;

    //  判断传入回调，默认开始执行定时器
    if (this.textChange || this.imageChange) {
      this.start();
    }
  }

  /**
   * 开始
   */
  start(): void {
    //  记录剪贴板目前的内容
    this.setClipboardDefaultValue();
    //  设置定时器
    this.setTimer();
  }

  /**
   * 暂停
   */
  stop(): void {
    //  清除定时器
    this.setTimer();
  }
}
```

`clearTimer`负责调用`clearInterval`来清除定时器，`setClipboardDefaultValue`负责设置实例中记录上一次变化的内容。

```ts
class ClipboardObserver {
  ...

  /**
   * 清除定时器
   */
  clearTimer(): void {
    clearInterval(this.timer);
  }

  /**
   * 设置剪贴板默认内容
   */
  setClipboardDefaultValue(): void {
    if (this.textChange) {
      //  electron剪贴板读取文本方法
      this.beforeText = clipboard.readText();
    }
    if (this.imageChange) {
      //  electron剪贴板读取图片方法
      this.beforeImage = clipboard.readImage();
    }
  }
}
```

`setTimer` 负责比较核心的逻辑，设置定时器，并在每一轮定时器回调中判断内容是否存在变化。如内容变化，执行内容变化回调。

```ts
class ClipboardObserver {
  ...

  /**
   * 设置定时器
   */
  setTimer(): void {
    //  设置定时器
    this.timer = setInterval(() => {
      if (this.textChange) {
        const text = clipboard.readText();
        //  判断内容是否与上次读取的内容不同
        if (this.isDiffText(this.beforeText, text)) {
          //  执行变化回调
          this.textChange(text, this.beforeText);
          //  记录此次内容
          this.beforeText = text;
        }
      }

      if (this.imageChange) {
        const image = clipboard.readImage();
        //  判断内容是否与上次读取的内容不同
        if (this.isDiffImage(this.beforeImage, image)) {
          //  执行变化回调
          this.imageChange(image, this.beforeImage);
          //  记录此次内容
          this.beforeImage = image;
        }
      }
    }, this.duration);
  }

  /**
   * 判断内容是否不一致
   * @param beforeText
   * @param afterText
   * @returns
   */
  isDiffText(beforeText: string, afterText: string): boolean {
    return afterText && beforeText !== afterText;
  }

  /**
   * 判断图片是否不一致
   * @param beforeImage
   * @param afterImage
   * @returns
   */
  isDiffImage(beforeImage: NativeImage, afterImage: NativeImage): boolean {
    return afterImage && !afterImage.isEmpty() && beforeImage.toDataURL() !== afterImage.toDataURL();
  }
}
```

## 打完收工

至此，剪贴板观察者类的逻辑已经完成了。我们在代码中可以这样食用

```ts
const clipboardObserver = new ClipboardObserver({
  textChange: (text: string, beforeText: string) => {
    //  处理文本变化的逻辑
  },
  imageChange: (image: Electron.NativeImage, beforeText: Electron.NativeImage) => {
    //  处理图片变化的逻辑
  }
});

//  也可以暂停
clipboardObserver.stop();

//  也可以再开始
clipboardObserver.start();
```

项目已开源到 [electron-clipboard-observer](https://github.com/onaug6th/electron-clipboard-observer) 中。可以将代码拷到项目中使用

也可以在项目中安装依赖 `electron-clipboard-observer` 安装后在项目内可直接引用使用。

```bash
npm install electron-clipboard-observer
```

```ts
import ClipboardObserver from "electron-clipboard-observer"

const clipboardObserver = new ClipboardObserver({ ... })
```

## 项目地址

[github electron-clipboard-observer](https://github.com/onaug6th/electron-clipboard-observer)

[npm electron-clipboard-observer](https://www.npmjs.com/package/electron-clipboard-observer)
