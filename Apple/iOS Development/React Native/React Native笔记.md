#React Native笔记
标签： 开发

##要点记录
###本地模块（[Native Modules][1]）
- **导出方法、导出静态变量、导出枚举**。
- **本地模块改变运行线程的方法**。
全局方法：重写属性methodQueue，如：
``` objectivec
- (dispatch_queue_t)methodQueue
{
  return dispatch_queue_create("com.facebook.React.AsyncLocalStorageQueue", DISPATCH_QUEUE_SERIAL);
}
```
个别方法：就是在调用回调的时候在外面包一层GCD，如：
``` objectivec
RCT_EXPORT_METHOD(doSomethingExpensive:(NSString *)param callback:(RCTResponseSenderBlock)callback)
{ 
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    // Call long-running code on background thread
    ...
    // You can invoke callback from any thread/queue
    callback(@[...]);
  });
}
```
- **发送事件给JavaScript**
Native代码通过RCTBridge的eventDispatcher发送事件：
``` objectivec
#import "RCTBridge.h"
#import "RCTEventDispatcher.h"
@(开发笔记)implementation CalendarManager

@synthesize bridge = _bridge;
- (void)calendarEventReminderReceived:(NSNotification *)notification
{
  NSString *eventName = notification.userInfo[@"name"];
  [self.bridge.eventDispatcher sendAppEventWithName:@"EventReminder" body:@{@"name": eventName}];
}
@end
```
JavaScript订阅事件：
``` javascript
import { NativeAppEventEmitter } from 'react-native';
var subscription = NativeAppEventEmitter.addListener(
  'EventReminder',
  (reminder) => console.log(reminder.name)
);
...
// Don't forget to unsubscribe, typically in componentWillUnmount
subscription.remove();
```
##本地UI组件（[Native UI Components][2])
- **本地的View都是通过`RCTViewManager`的子类来管理的，比如：`UIScrollView`会对应有一个`RCTScrollViewManager`，但这些`RCTViewManager`本质上是个单列，因为他们只会被bridge创建一次。`UIView`、`RCTViewManager`、`RCTUIManager`之间的关系如下图(不一定正确，需要研读代码做修正)**：
``` seq
UIView->RCTViewManager: UIView注册到RCTViewManager
RCTViewManager->RCTUIManager:提供UIView给
RCTUIManager-->RCTViewManager: 在更新UIView的属性时候通知它
RCTViewManager-->UIView: 更新或设置UIView的属性
```
- 当你想提供一个CustomView给JavaScript使用的时候要做的事情就是继承`RCTViewManager`创建一个`RCTCustomViewManager`，然后重写`- (UIView *)view`方法，同可以用宏`RCT_EXPORT_VIEW_PROPERTY`导出属性或者使用`RCT_CUSTOM_VIEW_PROPERTY`自定义属性，例如：
``` objectivec
@implementation RCTMapManager

RCT_EXPORT_MODULE()

- (UIView *)view
{
  RCTMap *map = [RCTMap new];
  map.delegate = self;
  return map;
}

RCT_EXPORT_VIEW_PROPERTY(showsUserLocation, BOOL)
RCT_CUSTOM_VIEW_PROPERTY(region, MKCoordinateRegion, RCTMap)
{
  [view setRegion:json ? [RCTConvert MKCoordinateRegion:json] : defaultView.region animated:YES];
}

...
@end
```
然后在JavaScript中就可以这一样使用了：
``` javascript
// MapView.js
import { requireNativeComponent } from 'react-native';
//requireNativeComponent automatically resolves this to "RCTMapManager"
<RCTMap showsUserLocation={false} />
module.exports = requireNativeComponent('RCTMap', null);
```
然而这并不是最好的方式，由于在Js中并不是很清楚的能看出来属性有什么，是什么类型，所以最好是将RCTMap再包一层，比如：

``` javascript
// MapView.js
import React, { requireNativeComponent } from 'react-native';

class MapView extends React.Component {
  render() {
    return <RCTMap {...this.props} />;
  }
}

MapView.propTypes = {
  /**
   * When this property is set to `true` and a valid camera is associated
   * with the map, the camera’s pitch angle is used to tilt the plane
   * of the map. When this property is set to `false`, the camera’s pitch
   * angle is ignored and the map is always displayed as if the user
   * is looking straight down onto it.
   */
  pitchEnabled: React.PropTypes.bool,
};

var RCTMap = requireNativeComponent('RCTMap', MapView);

module.exports = MapView;
```
##FLUX
**MVC模式**：
- Facebok 眼中的MVC
![Facebok 眼中的MVC](http://upload-images.jianshu.io/upload_images/1801567-736b93462451f44e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 网友眼中的MVC
![网友眼中的MVC](http://upload-images.jianshu.io/upload_images/1801567-da96d932bead788d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**FLUX**数据模型：（https://github.com/facebook/flux/）
![FLUX](http://upload-images.jianshu.io/upload_images/1801567-79c9daf37d2c9e9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Action:
- Dispatcher:
- Store:
- View:

All data flows through the dispatcher as a central hub. Actions are provided to the dispatcher in an action creator method, and most often originate from user interactions with the views. The dispatcher then invokes the callbacks that the stores have registered with it, dispatching actions to all stores. Within their registered callbacks, stores respond to whichever actions are relevant to the state they maintain. The stores then emit a change event to alert the controller-views that a change to the data layer has occurred. Controller-views listen for these events and retrieve data from the stores in an event handler. The controller-views call their own setState() method, causing a re-rendering of themselves and all of their descendants in the component tree.

**FLUX与MVC的区别**
- FLUX的Dispatcher与MVC的Controller的区别：Controller包含业务逻辑，而Dispatcher不包含业务逻辑，它可以在其他地方复用，主要职责是将事件分发给订阅者（Store）。

##ES6语法相关
- [**module**][3]
1. **实质：**ES6模块加载的机制，与CommonJS模块完全不同。CommonJS模块输出的是一个值的拷贝，而ES6模块输出的是值的引用。CommonJS模块输出的是被输出值的拷贝，也就是说，一旦输出一个值，模块内部的变化就影响不到这个值。
2. 循环加载问题，commonJS跟ES6的区别。
- [异步操作和Async函数](https://github.com/ruanyf/es6tutorial/blob/202f04bc43e0a9a74113338f0518847797071ae4/docs/async.md#异步操作和async函数)
1. [Promise](https://github.com/ruanyf/es6tutorial/blob/202f04bc43e0a9a74113338f0518847797071ae4/docs/async.md#promise)
2. [Generator](https://github.com/ruanyf/es6tutorial/blob/202f04bc43e0a9a74113338f0518847797071ae4/docs/async.md#generator函数)
使用`yield`作为关键字,每当程序运行到`yield`做修饰的代码，函数会暂停，并交出控制权给其他的协程，直到其协程返回控制权。调用next()方法执行下一个`yield`。
3. [Thunk](https://github.com/ruanyf/es6tutorial/blob/202f04bc43e0a9a74113338f0518847797071ae4/docs/async.md#thunk函数)
简单而言，就是将多参数的函数通过包装成高阶函数，将为只包含一个参数的函数。可使用`Thunkify`模块。安装方式为：`$ npm install thunkify`。
4. 编写自动执行器
当`Generator`和`Thunk`结合起来，即`Generator`函数调用多个`Thunk`函数，通过编写自动执行代码，可以实现一个自动执行器。[co模块](https://github.com/tj/co)就是一个自动执行器。实现自动执行器代码的过程一般是这样的：
>(1) 将要异步的函数转换成`Thunk`函数，如：读取文件`readFile`函数。
>(2) 使用关键字`yield`编写`Generator`函数。
>(3) 编写递归调用执行函数。
5. [ES7的`async`和`wait`关键字](https://github.com/ruanyf/es6tutorial/blob/202f04bc43e0a9a74113338f0518847797071ae4/docs/async.md#async-函数的用法)
`async`和`wait`关键字结合起来就实现了一个自动执行器。


##遇到的问题
1. 同时只能启动一个server。否则会报错：[error][tid:com.facebook.React.JavaScript] Application AwesomeProject has not been registered. This is either due to a require() error during initialization or failure to call AppRegistry.registerComponent.
2. Xcode 的 run script的运行路径是工程文件.xcodeproj所在目录。
3. ReactNative增量升级方案 http://react-china.org/t/reactnative/3932

[1]: http://facebook.github.io/react-native/docs/native-modules-ios.html#native-modules
[2]: http://facebook.github.io/react-native/docs/native-components-ios.html#content
[3]: https://github.com/ruanyf/es6tutorial/blob/8ad3c20f5f477a091282c764c15032732d386c48/docs/module.md
