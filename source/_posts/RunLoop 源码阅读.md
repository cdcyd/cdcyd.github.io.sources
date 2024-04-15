---
title: RunLoop 源码阅读
date: 2018-05-24 18:00:00
tags: "CoreFoundation"
categories: "源码阅读"
---

1. 获取runloop的函数
```
// 获取主线程的runloop
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

// 获取子线程的runloop
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```

2. 创建runloop的函数
```
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    // 判断t所指向的线程是否为空，如果为空，就获取当前的主线程进行赋值
    if (pthread_equal(t, kNilPthreadT)) {
        t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);

    // 全局__CFRunLoops，它是一个字典，用来存储所有线程的runloop
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        // 由于主线程默认就有runloop，所以再创建__CFRunLoops，要默认添加一个主线程的runloop
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }

    // 从__CFRunLoops中获取对应于当前线程的RunLoop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) {
        // 如果没有取到，说明是第一次，则创建一个runloop加入到全局字典中
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            // 存储当前线程和对应的RunLoop到全局字典__CFRunLoops中
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        __CFUnlock(&loopsLock);
        CFRelease(newLoop);
    }

    // 判断是否为当前线程，如果是就注册一个回调，当线程销毁时，也销毁其对应的 RunLoop
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```
3. 运行runloop的函数
```
void CFRunLoopRun(void) {
    // 只有运行完成或主动退出时，才会真正的退出
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {  
    CHECK_FOR_FORK();
    // 判断RunLoop是否正在被销毁，已被销毁就退出原因
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;

    __CFRunLoopLock(rl);

    // 获取当前 mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        // 如果当前 mode 和 runloop 的 mode 都为 nil，则退出
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }

    // 保存正在运行的RunLoop，把相关数据压进栈
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

    // 判断是否可以运行runloop，并通知观察者Observer，即将运行RunLoop
    if (currentMode->_observerMask & kCFRunLoopEntry ) 
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);

    // 运行RunLoop
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);

    // 判断是否要退出runloop，并通知观察者Observer，即将退出RunLoop
    if (currentMode->_observerMask & kCFRunLoopExit ) 
        __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}

/* rl, rlm are locked on entrance and exit 
   rl: runloop 
   rlm: runloop mode 
   seconds: 超时时间
   stopAfterHandle: 完成后是否停止 runloop
   previousMode: 前一次的 mode (主要因为 runloop 可以嵌套调用)
*/
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    // 当前内核时间
    uint64_t startTSR = mach_absolute_time();

    // 判断本次 runloop 或 mode 状态是否已经停止了
    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
	    return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    return kCFRunLoopRunStopped;
    }
    
    // 主线程消息端口
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    // 是否在主线程中
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    // 在主线程中 && 是主线程的runloop && mode类型为common
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) 
        // 获取GCD端口（4CF 表示 for Core Foundation）
        dispatchPort = _dispatch_get_main_queue_port_4CF();
    
// 是否使用了GCD timer
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    // GCD timer 消息端口
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
        if (!modeQueuePort) {
            CRASH("Unable to get port for run loop mode queue (%d)", -1);
        }
    }
#endif
    
    // 超时定时器
    dispatch_source_t timeout_timer = NULL;
    // 超时定时器的上下文
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { 
        // 如果超时时间<=0.0，立即就超时了，此时不需要定时器了
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
        // 如果超时时间在计时器的最大计时范围内，则初始化一个计时器并开始计时
        dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
        // 初始化计时器
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
        // 将计时器保存到上下文中
        timeout_context->ds = timeout_timer;
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        // 设置GCD的计时器上下文为当前超时计时器的上下文
        dispatch_set_context(timeout_timer, timeout_context); 
        // 设置超时source
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        // 设置超时取消source
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        // 设置超时时间
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        // 开始计时
        dispatch_resume(timeout_timer);
    } else { 
        // 如果超时时间很大，则认为是无限时间，即永远也不可能超时(这里的seconds值大概相当于317年)
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    // 函数的返回值 (return value)，默认为0
    int32_t retVal = 0;
    // do {} while() 这个循环就是runloop的核心代码块
    do {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
#endif  
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#elif DEPLOYMENT_TARGET_WINDOWS
        HANDLE livePort = NULL;
        Boolean windowsMessageReceived = false;
#endif
        // 申请一个存放消息的空间
        uint8_t msg_buffer[3 * 1024];

        __CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);

        // 1.通知 observer，即将处理 timers
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) 
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 2.通知 observer，即将处理 sources
        if (rlm->_observerMask & kCFRunLoopBeforeSources) 
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        
        // 3.处理 blocks
        __CFRunLoopDoBlocks(rl, rlm);

        // 4.处理 sources0
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        // 如果实际处理了 sources0，则再一次处理 blocks
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }
        
        // 处理完 sources0 或 超时时间为0
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
      
        // 5.处理主线程中的GCD事件，循环的第一次是不会进入的，因为didDispatchPortLastTime永远是true
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }
        didDispatchPortLastTime = false;

        // 6.没有处理完sources0 && 没有超时，通知 observer，即将进入睡眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting))    
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        // 7.设置标志，正在睡眠 (实际runloop还没有睡眠）
        __CFRunLoopSetSleeping(rl);
        // 8.将GCD端口加入所有监听端口集合中
        __CFPortSetInsert(dispatchPort, waitSet);
        
        __CFRunLoopModeUnlock(rlm);
        __CFRunLoopUnlock(rl);

        // 记录睡眠开始时间
        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        // 如果使用 GCD timer，获取消息后需要对GCD Timer做一些额外的处理
        do {        
            if (kCFUseCollectableAllocator) {
                memset(msg_buffer, 0, sizeof(msg_buffer));
            }
            msg = (mach_msg_header_t *)msg_buffer;
            
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                break;
            }
        } while (1);
#else
        if (kCFUseCollectableAllocator) {
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        
#elif DEPLOYMENT_TARGET_WINDOWS
        __CFRunLoopWaitForMultipleObjects(waitSet, NULL, poll ? 0 : TIMEOUT_INFINITY, rlm->_msgQMask, &livePort, &windowsMessageReceived);
#endif
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // 移除 GCD 端口
        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        //  处理完成 GCD 后，通知 observer，即将进入睡眠
        __CFRunLoopUnsetSleeping(rl);
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) 
            __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

        // 收到消息的标签
        // 1. 基于 port 的 Sources1 的事件
        // 2. Timer 到时间
        // 3. RunLoop 超时
        // 4. 被手动唤醒
        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);

#if DEPLOYMENT_TARGET_WINDOWS
        if (windowsMessageReceived) {
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);

            if (rlm->_msgPump) {
                rlm->_msgPump();
            } else {
                MSG msg;
                if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE | PM_NOYIELD)) {
                    TranslateMessage(&msg);
                    DispatchMessage(&msg);
                }
            }
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            
            __CFRunLoopSetSleeping(rl);
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            
            __CFRunLoopWaitForMultipleObjects(waitSet, NULL, 0, 0, &livePort, NULL);
            
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);            
            __CFRunLoopUnsetSleeping(rl);
        }  
#endif

        if (MACH_PORT_NULL == livePort) {
            // 不知道哪个端口唤醒的
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
        } else if (livePort == rl->_wakeUpPort) {
            // 被 CFRunLoopWakeUp 函数唤醒的
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
        }

#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            // 被 GCD timers 唤醒，处理 timers
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif

#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            // 被其它 timers 唤醒，处理 timers
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
        else if (livePort == dispatchPort) {
            // 被主线程 GCD 唤醒，处理 GCD
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
#if DEPLOYMENT_TARGET_WINDOWS
            void *msg = 0;
#endif
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            // 被 sources1 唤醒，处理 sources1
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
                mach_msg_header_t *reply = NULL;
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
#elif DEPLOYMENT_TARGET_WINDOWS
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls) || sourceHandledThisLoop;
#endif
            }
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
        } 
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
#endif

        // 再一次处理 blocks
        __CFRunLoopDoBlocks(rl, rlm);
         
        // 判断是否退出
        if (sourceHandledThisLoop && stopAfterHandle) {
            // 处理完事件后退出
            retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            // 超时退出
            retVal = kCFRunLoopRunTimedOut;
        } else if (__CFRunLoopIsStopped(rl)) {
            // 调用CFRunLoopStop()退出
            __CFRunLoopUnsetStopped(rl);
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            // 运行模式被终止时退出
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped;
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            // 没有要处理的事件时退出
            retVal = kCFRunLoopRunFinished;
        }
        
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```
Runloop 运行的核心函数`__CFRunLoopRun `大概有300多行代码，看似很多代码，其实其中有很多是为了适配不同平台的代码，如果我们将其删减一下，删掉那些平台的适配以及一些复杂的逻辑判断，我们就可以得到如下的runloop执行过程
```
staticint32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    do {
        // 1. 通知 observers，RunLoop 即将处理 Timers
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 2. 通知 observers，RunLoop 即将处理 Sources
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        // 3. 处理 sources0 事件
        __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        // 4. 处理 blocks
        __CFRunLoopDoBlocks(rl, rlm);
        // 5. 处理 sources1 事件
        goto handle_msg;
        // 6. 通知 observers, RunLoop 即将休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        // 7. RunLoop 休眠
        __CFRunLoopSetSleeping(rl);
        // 8. 线程进入休眠, 直到被下面某一个事件唤醒：
        // a. 基于 port 的 Source1 的事件
        // b. Timer 到时间了
        // c. RunLoop 启动时设置的最大超时时间到了
        // d. 被手动唤醒
        // 9. 当接收到第8步中的消息时，Runloop 醒来
        __CFRunLoopUnsetSleeping(rl);
        // 10. 通知 observers，RunLoop 已被唤醒
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
        // 11. 处理消息
        handle_msg:;
        if(MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
        } else if(livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
        } else if(modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            // 处理定时器事件
            CFRUNLOOP_WAKEUP_FOR_TIMER();
        } else if(livePort == dispatchPort) {
            // 处理主线程消息
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
        } else {
            // 处理 sources1
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
        } 
        // 12. 退出 RunLoop
    return retVal;
}
```