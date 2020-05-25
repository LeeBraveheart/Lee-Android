
| 姓名      | 联系方式   |  更新日期 | 
| --------- | -------- | -----: | 
|    向洋| 15013787307  |2020.04.28

# 经销商E助手RN与H5交互

_**背景：
经销商E助手使用react-native编写，同时运行android、ios两端
对于普通的多级页面，RN的webview在加载H5页面时，可以根据webview控制返回，关闭、title更新等操作，但是如果H5使用vue的单页面，webview则监听不到，所以才有了下面的一些交互。（当然，RN与H5的正常业务交互也是必不可少的）**_

## RN Webview 向H5注入JS
- RN向H5发送消息
`this.webview.injectJavaScript(receiveMessage(${responeMsg});true;)`
H5需定义`receiveMessage()`方法供RN调用，其中**responeMsg**为RN向H5传递的参数，格式为json

可参考：[React-Native Webview 和H5交互](https://www.jianshu.com/p/44365ec64e4a)

## H5发送消息给RN Webview
- H5向RN发送消息
```
const injectedJavascript = `(function() {
  window.postMessage = function(data) {
    window.ReactNativeWebView.postMessage(data);
  };
})()`;
```

**注意：参数data必须为String，且String里面的格式为json方便取值**

发送消息的参数双方通过json里的functionType字段作为功能号，方便交流

1. **functionType:  '10001'，**==分享好友：H5调用RN==
- title:  分享标题
- description:  分享描述,
- webpageUrl:  分享地址,

2. **functionType:  '10002'，**==关闭webview：H5调用RN==
- 无参数

3. **functionType:  '10003'，**==RN点击返回：RN调用H5==
- 无参数

4. **functionType:  '10004'，**==更新title：H5调用RN==
- title：网页标题

5. **functionType:  '10005'，**++特殊业务场景++---==跳转代理商收款账户管理：H5调用RN==
- 无参数

6. **functionType:  '10006'，**==获取app信息：H5调用RN==
- 无参数

7. **functionType:  '10007'，**==RN发送app信息：RN调用H5==
- loginName：登录名 
- appVersion: 版本包 
- sign:签名 

8. **functionType:  '10008'，** ++特殊业务场景++---==展示导航栏搜索：H5调用RN==
- showSearch：true展示 false不展示

9. **functionType:  '10009'，**++特殊业务场景++---==RN点击搜索：RN调用H5==
- 无参数

10. **functionType:  '10010'，**++特殊业务场景++==获取键盘高度：RN调用H5==
RN注入`receiveKeyboardMessage()`方法
- keyboardHeight:键盘高度

11. **functionType:  '10011'，**==H5调用支付：H5调用RN==
_参数都可以放在json里面_
- payType：0微信  1支付宝
- extData：额外参数
- payData:微信参数（json格式）
微信参数如下图（其中appId、packageValue app可以自己获取不用传）
![微信支付参数]($resource/%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98%E5%8F%82%E6%95%B0)

12. **functionType:  '10012'，**==RN返回支付结果：RN调用H5==

- errCode：错误码
![微信支付结果]($resource/%E5%BE%AE%E4%BF%A1%E6%94%AF%E4%BB%98%E7%BB%93%E6%9E%9C)

- errStr：错误信息
- type：PayReq.Resp 回调类型



13. **functionType:  '10013'，**==H5 reload webview：H5调用RN==

   - 无参数


14. **functionType:  '10014'，**==H5调用app埋点统计：H5调用RN==

- eventId：事件id
- eventDes：事件描述
- params:额外参数（json格式），非必传



# 说明：
- 点击android物理返回键或导航栏返回时，RN发送10003消息，H5收到返回后可以返回上一级，如果H5到了首页不能返回了则发送10002消息给RN，RN会关闭webview或者其他处理，消息监听建议H5全局处理。
- 更新title时，H5需要在每次路由发生变化时发送10004

# 注意点：
- 各个功能号RN向H5注入的方法都是`receiveMessage()方法`，但是H5获取键盘高度时注入的是`receiveKeyboardMessage()方法，当时H5说定义的方法有冲突`，其实我觉得用一个方法就可以搞定的，可能他们姿势不对
- ==（RN调用H5时）RN注入的方法如果H5没有定义该方法，在ios模拟器上会报红，android模拟器正常，真机ios、android均正常。如果加载H5报错，会导致RN与H5无法通信，这也是个弊端，特别处理返回时（比如如果某个H5页面报错，会无法返回）,要是想监听H5的路由变化需要修改android、ios webview的源码，灰常麻烦且不易维护，或许RN的版本升级看有没有解决这个问题，但是升级RN的版本会有很多兼容问题==
- H5输入框被键盘挡住问题（android）
     webview加载H5时，H5的输入框被键盘挡住问题
     1. 目前解决方案：H5调用RN方法获取键盘高度（已实现），H5根据高度调整布局（如果地方太多H5会很麻烦）
     2. 如果用android原生解决：
         - 调整AndroidManifest.xml中MainActivity的`windowSoftInputMode属性`,但是app是单Activity使用了沉浸式（全屏），该属性不生效
         - 原生统一在webview中根据键盘变化调整布局，`MainActivity`的`onCreate()`方法中使用`AndroidBug5497Workaround(已注释，需要可以打开看看效果，H5键盘问题可以解决)`但是由于app为单Activity的限制，调整布局会影响RN的布局，**具体可看看登陆界面和其他有输入框的RN界面的效果****，看产品能不能接受，使用该方法后H5获取键盘高度后调整自身布局的逻辑可以不用了
