# # CocoaLumberjack 源码分析

github star 很多，作为一个很优秀的日志系统库，里面的代码、设计、注释都非常好，下面简单分析下。

先来看下调用方式，很简洁的方式

```
DDLogVerbose("Verbose")
DDLogDebug("Debug")
DDLogInfo("Info")
DDLogWarn("Warn")
DDLogError("Error")
```


### 设计架构如图
![](https://raw.githubusercontent.com/CocoaLumberjack/CocoaLumberjack/master/Documentation/CocoaLumberjackClassDiagram.png)

### DDLog
首先 DDLog 作为一个单例，
* 维护了一个数组 loggers 持有了各个 Logger 实例，通过 addLogger 来添加

```
+ (void)addLogger:(id <DDLogger>)logger
```
* 再看 DDLog 初始化方法

```        
_loggingQueue = dispatch_queue_create("cocoa.lumberjack", NULL);
_loggingGroup = dispatch_group_create();
dispatch_queue_set_specific(_loggingQueue, GlobalLoggingQueueIdentityKey, nonNullValue, NULL);
```
loggingQueue 作为一个全局的串行队列，保证 addLogger_removeLogger_quertLogMessage 都在线程上顺序执行。

dispatch_queue_set_specific 增加一个 key GlobalLoggingQueueIdentityKey 来标记当前线程，后面的代码可以通过 dispatch_get_specific 来检查是否在此线程上。

loggingGroup 是控制所有 logger 执行顺序，后面会详细说一下。

### DDLogger

每一个 logger 都有自己的一个 loggerQueue 和 DDLogFormatter
如类图描述，DDLogger 和 DDLogFormatter 作为一个抽象接口类，本身只提供了一个接口，实现的功能由子类来完成。

* DDOSLogger  
支持 console log 显示，iOS 10 以后使用的是 Apple os_log，性能比较好；iOS 10 之前是使用 DDASLLogger。
* DDFileLogger   
日志文件存储，支持时间大小限制，分页存储，文件回滚，文件排序，功能比较强大。
* DDAbstractDatabaseLogger  
数据库 logger 抽象类，如果有需要，可以继承此类，可以很方便接入，增加数据库的操作。

### 方法调用流程

这是 DDLog swift 对外提供的接口
```
@inlinable
public func DDLogInfo(_ message: @autoclosure () -> String,
                      level: DDLogLevel = DDDefaultLogLevel,
                      context: Int = 0,
                      file: StaticString = #file,
                      function: StaticString = #function,
                      line: UInt = #line,
                      tag: Any? = nil,
                      asynchronous async: Bool = asyncLoggingEnabled,
                      ddlog: DDLog = .sharedInstance) {
    _DDLogMessage(message(), level: level, flag: .info, context: context, file: file, function: function, line: line, tag: tag, asynchronous: async, ddlog: ddlog)
}
```

先生成 DDLogMessage ，判断线程（上面说过根据GlobalLoggingQueueIdentityKey 来区分），保证在 loggingQueue 中写入日志，注意这里有个信号量 wait，等待写入日志完成
```    
dispatch_block_t logBlock = ^{
        dispatch_semaphore_wait(_queueSemaphore, DISPATCH_TIME_FOREVER);
        // We're now sure we won't overflow the queue.
        // It is time to queue our log message.
        @autoreleasepool {
            [self lt_log:logMessage];
        }
};
if (asyncFlag) {
        dispatch_async(_loggingQueue, logBlock);
    } else if (dispatch_get_specific(GlobalLoggingQueueIdentityKey)) {
        // We've logged an error message while on the logging queue...
        logBlock();
    } else {
        dispatch_sync(_loggingQueue, logBlock);
    }
}
```

最终核心代码在这里 lt_log
```
- (void)lt_log:(DDLogMessage *)logMessage {
    // Execute the given log message on each of our loggers.

    NSAssert(dispatch_get_specific(GlobalLoggingQueueIdentityKey),
             @"This method should only be run on the logging thread/queue");

    if (_numProcessors > 1) {
        // Execute each logger concurrently, each within its own queue.
        // All blocks are added to same group.
        // After each block has been queued, wait on group.
        //
        // The waiting ensures that a slow logger doesn't end up with a large queue of pending log messages.
        // This would defeat the purpose of the efforts we made earlier to restrict the max queue size.

        for (DDLoggerNode *loggerNode in self._loggers) {
            // skip the loggers that shouldn't write this message based on the log level

            if (!(logMessage->_flag & loggerNode->_level)) {
                continue;
            }

            dispatch_group_async(_loggingGroup, loggerNode->_loggerQueue, ^{ @autoreleasepool {
                [loggerNode->_logger logMessage:logMessage];
            } });
        }

        dispatch_group_wait(_loggingGroup, DISPATCH_TIME_FOREVER);
    } else {
        // Execute each logger serially, each within its own queue.

        for (DDLoggerNode *loggerNode in self._loggers) {
            // skip the loggers that shouldn't write this message based on the log level

            if (!(logMessage->_flag & loggerNode->_level)) {
                continue;
            }

#if DD_DEBUG

```            // we must assure that we aren not on loggerNode->_loggerQueue.
            if (loggerNode->_loggerQueue == NULL) {
               tell that we can't dispatch logger node on queue that is NULL.
              NSLogDebug(@"DDLog: current node has loggerQueue == NULL");
            }
            else {
              dispatch_async(loggerNode->_loggerQueue, ^{
                if (dispatch_get_specific(GlobalLoggingQueueIdentityKey)) {
                   tell that we somehow on logging queue?
                  NSLogDebug(@"DDLog: current node has loggerQueue == globalLoggingQueue");
                }
              });
            }
#endif
             next, we must check that node is OK.
            dispatch_sync(loggerNode->_loggerQueue, ^{ @autoreleasepool {
                [loggerNode->_logger logMessage:logMessage];
            } });
        }
    }
    dispatch_semaphore_signal(_queueSemaphore);
}
```
因为每个 log 都有自己的线程，日志写入都是并行执行的，多核心时使用 dispatch_group 做线程同步，使每个日志写入都加到 group 内，处于等待状态，直到最后保证都执行完成；单核心就直接在每个 log 的线程执行；最后发送信号量表示日志写入完成。
这是一个日志写入的大致流程。

### 总结
架构设计非常好，极好的阐述了抽象接口的概念，职责分的很清楚，👍 ！~