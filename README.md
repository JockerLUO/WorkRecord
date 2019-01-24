# WorkRecord
日常工作记录

## 1-23
### 目标:明确crash捕获逻辑,解决crash捕获不到问题
### 推测:集成多个第三方SDK污染造成
#### 预定分步目标:
1. 实现一份crash捕获demo
2. 理清sentry思路
3. 修改项目中crash捕获问题

### 实现步骤:
 1. 参考[漫谈iOS Crash收集框架](https://nianxi.net/ios/ios-crash-reporter.html)
 
 2. 学习 [ fishhook](https://github.com/facebook/fishhook) ,参考 [动态修改 C 语言函数的实现](https://github.com/draveness/analyze/blob/master/contents/fishhook/%E5%8A%A8%E6%80%81%E4%BF%AE%E6%94%B9%20C%20%E8%AF%AD%E8%A8%80%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0.md)
 
 3. 收获:fishhook类似OC的runtime,
 内部hook的是C的链接库,不可以直接hook已有的函数,
  `rebind_symbols` 其主要作用就是注册一个函数并在镜像加载时回调
  `rebind_symbols_image` 其主要作用重绑定符号
  可以使用fishhook获取到crash捕获注册方法`NSSetUncaughtExceptionHandler`的调用确定污染
  
 4. 调试demo发现:   
  阿里埋点SDK内部使用的是`PLCrashReporter`收集crash,通过`NSSetUncaughtExceptionHandler`注册异常处理程序,   
  高德地图内部在初始化时就通过`NSSetUncaughtExceptionHandler`收集异常,初始化后可以通过设置`crashReportEnabled`属性取消`NSSetUncaughtExceptionHandler`注册,高德地图SDK中有说明,   
  sentryt未发现通过`NSSetUncaughtExceptionHandler`注册异常处理,  
  暂未发现第三方SDK对sentry的crash收集造成影响,暂未确定原因    
  
5. 搜索发现`sentry`和`KSCrash`有很强的关联性,sentry相关资料较少,KSCrash资料较完整
  
6. 参考[KSCrash崩溃收集原理浅析](https://www.jianshu.com/p/34b98060965b)
  
7. 收获:sentrys有通过监听异常处理端口的方式捕获异常  
  在sentry源码中搜索打断点确定有相关方法,但在调试模式下不走该方法,进一步调试,确定是有宏定义判断造成的  
  相应的`NSSetUncaughtExceptionHandler`调用也是相同的原因造成没有调用,控制台没有输出,查看离线日志可知sentry调用了`NSSetUncaughtExceptionHandler`    
  阿里埋点SDK内部`crash`使用的是`PLCrashReporter`,检索源码没有发现`NSGetUncaughtExceptionHandler`  
  `PLCrashReporter`不会传递crash通知
  `sentry`的调用和阿里埋点crash收集的调用存在先后顺序造成`sentry`捕获crash异常
  
8. 实现测试demo   
  通过调用`NSSetUncaughtExceptionHandler`可以捕获signals异常,   
  signals异常已经Apple封装,可直接识别为OC方法crash的捕获,把crash信息存储到本地   
  更底层的c语言错误无法通过`NSSetUncaughtExceptionHandler`的函数捕获到,如     
  ``int* p = 0;``    
  ``*p = 0;``    
  需要使用更底层的监听异常处理端口接收mach异常,暂未实现
  
9. 更改项目中sentry调用顺序,把crash模块和连续闪退模块分开处理,连续闪退保护模块最新调用,crash模块最后调用,当发生连续闪退时保护模块直接调用crash模块,跳过中间其他模块
  
10. 连续闪退恢复APP时注意加入一个5秒的模拟进度,以供crash模块上传crash报告,5秒后之前清空沙盒文件