# vconsole-vue-devtool-plugin

`vconsole-vue-devtool-plugin` 是一款`vConsole`插件，把`Vue.js`官方调试工具`vue-devtools`移植到移动端，可以直接在移动端查看调试`Vue.js`应用

![WechatIMG71.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68d77a8750fb459cab9aab3e09c3b9a0~tplv-k3u1fbpfcp-watermark.image)
### 为什么需要本插件：

1. 在Safari和移动端无法直接调试Vue.js
2. Electron版本的devtools安装和远程调试配置非常麻烦

### 功能

1. 移植了官方Vue-devtools的全部功能
2. 针对移动端优化了部分操作方式
3. 现已支持微信端内浏览器
### 使用方式

1. ```npm install vconsole-vue-devtool-plugin --dev-save ```

2. 在工程中入口文件 （如`src/main.js`）

```javascript
...
import VConsole from "vconsole";
import Devtools from 'vconsole-vue-devtool-plugin'
Devtools.initPlugin(new VConsole()); // 需要在创建Vue根实例前调用
...
```

### 高级用法

1. 只在开发环境下引入

   ```javascript
   new Vue({
     render: (h) => h(App),
   }).$mount("#app");
   
   // 在创建跟实例以后调用， 需要借助webpack的异步模块加载能力
   if(process.env.NODE_ENV === "development"){
      Promise.all([import("vconsole"), import("vconsole-vue-devtool-plugin")]).then(
        (res) => {
          if (res.length === 2) {
            Vue.config.devtools = true;
            window.__VUE_DEVTOOLS_GLOBAL_HOOK__.emit("init",Vue)
            const VConsole = res[0].default;
            const Devtools = res[1].default;
            Devtools.initPlugin(new VConsole());
          }
        }
      );
    }
   ```
### 更新日志

#### v0.0.2
1. 重要更新，解决iOS微信端浏览器兼容性问题
2. 解决iOS阿里mPass容器兼容性问题
3. 优化了打包体积

### TODO:

1. 支持Vue.js 3
2. 开发脱离vConsole版本
3. webpack plugin

**### Sample code**

[Github](https://github.com/CavinHuang/vconsole-vue-devtool-plugin/dev)

### 实现过程
实现过程
分析完Vue、Vue-devtools、vConsole源码以后，似乎可行。步骤如下：

剥离Vue-devtools @front部分
实现@backend 和 @front 部分通信
实现front注入iframe
iframe嵌入vConsole
制作npm包并发布

#### 1. 剥离front

首先一个问题就是 Vue-devtools并不是一个库，所以npm上没有它，无法引用，其次这个工程也不适合作为库输出，因为它的打包方式比较特殊，还有这个工程本身的目的就是打包成一个可执行的App或者chrome插件，所以如果想引用它里面的代码，我想到最简单的方式就是拷贝了。。。所以剥离frontend非常简单，在Vue-devtools工程的package.json中增加`script："buildvc": "cd packages/shell-dev && cross-env NODE_ENV=production webpack \--progress \--hide-modules"`, 同时在shell-dev里增加一个文件叫 inject.js
```js
import { initDevTools } from '@front'
import Bridge from '@utils/bridge'

const targetWindow = window.parent;
document.body.style.overflow = "scroll";
initDevTools({
  connect (cb) {
    cb(new Bridge({
      listen (fn) {
        window.addEventListener('message', evt => {
          fn(evt.data)
        })
      },
      send (data) {
        targetWindow.postMessage(data, '*')
      }
    }))
  },
  onReload (reloadFn) {
    reloadFn.call();
  }
})
```
当然shell-dev里的webpack.config.js里 增加一个入口配置 `inject: './src/inject.js'`

打出来的包就是我们要的front部分，最终嵌入iframe里。

#### 2.实现通信
上面的inject.js中已经包含了 front部分 接收和发送消息的代码了。接下来完成backend部分的消息发送和接收，
```js
import { initBackend } from '@back'
import Bridge from '@utils/bridge'

const initBackendWithTargetWindow = function(win,targetWindow){
  const bridge = new Bridge({
    listen (fn) {
      win.addEventListener('message', evt => {
        fn(evt.data)})
    },
    send (data) {
      targetWindow.postMessage(data, '*')
    }
  })
  
  initBackend(bridge)
}

export default { initBackendWithTargetWindow }

```

#### 3. front嵌入iframe
这个比较麻烦，也遇到了一些兼容性问题，最终方案是：
 - 把第一步打包的inject.js 重命名为 inject.txt
 - 增加一个rawloader规则，识别txt
 - 使用script.txt的方式插入到script标签中，然后插入iframe的body中
```js
import injectString from './inject.txt'

function inject (scriptContent, done) {
  const script = document.getElementById('vue-iframe').contentWindow.document.createElement('script')
  script.text = scriptContent
  document.getElementById('vue-iframe').contentWindow.document.body.appendChild(script)
}

inject(injectString)

```
#### 4. iframe嵌入vconsole
实现一个plugin类，继承VConsolePlugin，然后实现对应的方法即可，具体文档可以查看VConsole文档vConsole Readme
```js
class VConsoleVueTab extends VConsolePlugin {

  constructor(...args) {
    super(...args);
  }
  onRenderTab(cb){
    cb(`<iframe id="vue-iframe" style="width:100%;position:absolute;top:0;bottom:0;min-height:100%;"></iframe>`);
  }
  onReady() {
    target = document.getElementById('vue-iframe')
    targetWindow = target.contentWindow;
    be.initBackendWithTargetWindow(window,targetWindow);    
  }

  onShow() {    
    injectOnce(injectString)
  }
}

```

#### 5. 制作npm包并发布
这一步需要打包发布，并且优化

```js
const initPlugin = function(vConsole){
  var tab = new VConsoleVueTab('vue', 'Vue');
  vConsole.addPlugin(tab);
}
export default {
  initPlugin
}
```
