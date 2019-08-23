---
title: Android之9.0Activity启动流程（二）
date: 2019-08-23 09:50:16
tags: 启动流程
categories: 四大组件
---

> 
>
> 注：下列源码均基于9.0，可通过下列方式下载本文相关源码到本地：
>
> - git clone https://aosp.tuna.tsinghua.edu.cn/platform/frameworks/base
>
> 参考博客：[如何下载和阅读Android源码](https://www.jianshu.com/p/14ade986d3a8)

# 前言

在上一篇文章[Android之Actvitiy启动流程（一）](https://jsyjst.github.io/2019/08/06/Android之Activity启动流程-一/)中我们已经分析了根Activity启动时所需的应用进程是如何创建的，并且当前应用进程已经启动了ActivityThread的main方法，所以这篇文章主要围绕下列内容展开来讲解：

- 应用进程绑定到AMS
- AMS发送启动Activity的请求
- ActivityThread的Handler处理启动Activity的请求

# 一、应用进程绑定到AMS

## 1. 时序图

![1566373261686](应用进程绑定到AMS.png)

## 2. 详细过程

在前面一篇我们知道当Zygote进程孵化出应用进程后会执行ActivityThread的main方法，所以我们先看看main方法里的代码。

> frameworks/base/core/java/android/app/ActivityThread.java

```java
    public static void main(String[] args) {
        ....
		//创建主线程的Looper以及MessageQueue
        Looper.prepareMainLooper();
        ...
        //AMS绑定ApplicationThread对象,即应用进程绑定到AMS
        thread.attach(false, startSeq);
		//开启主线程的消息循环
        Looper.loop();
        ...
    }
```

在这里我们就不再详细分析prepareMainLooper和loop方法，其主要功能就是准备好主线程的Looper以及消息队列，最后再开启主线程的消息循环。如果不懂的可以看看前面的博客[**Handler机制**](https://blog.csdn.net/qq_41979349/article/details/98107823)中对这两个方法的分析，在这里我们将重点分析attach这个方法。

> frameworks/base/core/java/android/app/ActivityThread.java

```java
    private void attach(boolean system, long startSeq) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            final IActivityManager mgr = ActivityManager.getService();
            try {
				//AMS绑定ApplicationThread对象
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            //垃圾回收观察者
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                            throw e.rethrowFromSystemServer();
                        }
                    }
                }
            });
        } 
        ...
    }
```

可以看到由于在ActivityThread的attach中我们传入的是false，故在attach方法中将执行!system里的代码，通过调用AMS的attachApplication来将ActivityThread中的内部类ApplicationThread对象绑定至AMS，这样AMS就可以通过这个代理对象 来控制应用进程。接着为这个进程添加垃圾回收观察者，每当系统触发垃圾回收的时候就在run方法中计算应用使用了多大的内存，如果超过总量的3/4就尝试释放内存。

> frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }


    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
			....
			    //将应用进程的ApplicationThread对象绑定到AMS，即AMS获得ApplicationThread的代理对象
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
          

        boolean badApp = false;
        boolean didSomething = false;
        // See if the top visible activity is waiting to run in this process...
        if (normalMode) {
            try {
				//启动Activity
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
                badApp = true;
            }
        }
        .....
    }

```

在AMS的attachApplication方法中调用了attachApplicationLocked进行绑定，从上面代码可以发现attachApplicationLocked中有两个重要的方法：thread.bindApplication和mStackSupervisor.attachApplicationLocked(app)。thread.bindApplication中的thread其实就是ActivityThread里ApplicationThread对象在AMS的代理对象，故此方法将最终调用ApplicationThread的bindApplication方法。而mStackSupervisor.attachApplicationLocked(app)主要是AMS启动Activity的作用（在下列AMS发送启动Activity的请求会分析到）。在这里我们先看看ApplicationThread的bindApplication方法。

> frameworks/base/core/java/android/app/ActivityThread.ApplicationThread

```java
       public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {

          	....
			//向H发送绑定ApplicationThread对象的消息
            sendMessage(H.BIND_APPLICATION, data);
        }

    void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        ...
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }

```

可以发现上面其实就是Handler机制的应用，mH其实就是H类型的，所以bindApplication主要就是向ActivityThread的Handler,即H发送绑定的消息。

> frameworks/base/core/java/android/app/ActivityThread.H

```java
	//线程所需要的Handler，内部定义了一组消息类型，主要包括四大组件的启动和停止过程
    class H extends Handler {
        public static final int BIND_APPLICATION        = 110;
        ...
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ...
        }
    }

```

这个H其实就是当前线程，也可以说是主线程ActivityThread的Handler，内部定义了一组消息类型，包括了四大组件的启动和停止过程。通过Handler机制我们知道在H的handleMessage中会处理发送给H的消息，在这里将调用handleBindApplication方法。

> frameworks/base/core/java/android/app/ActivityThread

```java
    @UnsupportedAppUsage
    private void handleBindApplication(AppBindData data) {

        ....
        final InstrumentationInfo ii;
        if (data.instrumentationName != null) {
            try {
                ii = new ApplicationPackageManager(null, getPackageManager())
                        .getInstrumentationInfo(data.instrumentationName, 0);
            } 
            ...
        } else {
            ii = null;
        }
        ....
        // Continue loading instrumentation.
        if (ii != null) {
            .....
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }
        ....
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;
            try {
                //调用Application的onCreate的方法
                //故Application的onCreate比ActivityThread的main方法慢执行
                //但是会比所有该应用Activity的生命周期先调用，因为此时的Activity还没启动
                mInstrumentation.callApplicationOnCreate(app);
            }
            ...
        } 
    }
```

# 二、AMS发送启动Activity的请求

应用进程绑定后，随之就得启动Activity，那么得向谁发送启动Activity得到请求呢？显而易见，当然是向主线程即ActivityThread发送启动Activity的请求。下面就让我们一探究竟！



## 1. 时序图

![1566459635853](AMS发送启动Activity的请求.png)

## 2. 详细过程

现在当前进程已经绑定到了AMS，绑定后呢？回忆一下，上面对AMS的attachApplicationLocked方法分析时，重点提到了两个方法，其中ApplicationThread的bindApplication方法我们分析应用进程是如何绑定到AMS的，没错！另外一个方法mStackSupervisor.attachApplicationLocked(app)就是用来启动Activity的，现在让我们来看看。

> frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

```java
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ...
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                ....
                for (int i = 0; i < size; i++) {
                    final ActivityRecord activity = mTmpActivityList.get(i);
                    if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                            && processName.equals(activity.processName)) {
                        try {
						    //真正启动Activity，也是普通Activity的启动过程
                            if (realStartActivityLocked(activity, app,
                                    top == activity /* andResume */, true /* checkConfig */)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                             ...
                        }
                    }
                }
            }
        }
        return didSomething;
    }
```

attachApplicationLocked会调用realStartActivityLocked方法，如下。

> frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

```java
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
                .....
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
			    //添加callback，首先会执行callback中的方法
                 //此时的ActivityLifecycleItem为LaunchActivityItem
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                //判断此时的生命周期是resume还是pause
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                //设置当前的生命周期
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // mService为AMS对象,getLifecycleManager得到ClientLifecycleManager
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
        	    ......
            } 
			....
        return true;
    }
```

在realStartActivityLocked方法中为ClientTransaction对象添加LaunchActivityItem的callback，然后设置当前的生命周期状态，由于我们是根Activity的启动，很显然这里的生命周期为ResumeActivityItem，最后调用ClientLifecycleManager.scheduleTransaction方法执行。我们继续追踪下去!

> frameworks/base/services/core/java/com/android/server/am/ClientLifecycleManager.java

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }
```

从上面代码中可以看出，由于transaction为ClientTransaction类型，故接下去将执行ClientTransaction的schedule方法

> frameworks\base\core\java\android\app\servertransaction\ClientTransaction.java

```java
    public void schedule() throws RemoteException {
        //1.mClient是IApplicationThread类型，
        //2.ActivityThread的内部类ApplicationThread派生这个接口类并实现了对应的方法
        //3.ActivityThread实际调用的是他父类ClientTransactionHandler的scheduleTransaction方法。
        mClient.scheduleTransaction(this);
    }
```

mClient是IApplicationThread类型，通过名字和阅读源码我们可以知道ApplicationThread会实现该类型，所以这里调用的应该是ApplicationThread的scheduleTransaction方法，我们知道ApplicationThread是ActivityThread的内部类，所以通过阅读ActivityThread源码你会发现在这个类中并没有实现scheduleTransaction这个方法，难道分析错了？？？？

聪明的你应该想到了，既然在当前类找不到这个方法，只能去找父类寻求帮助了，果然姜还是老的辣，在ActivityThread的父类ClientTransactionHandler中我们找到了这个scheduleTransaction方法。如下

> frameworks\base\core\java\android\app\ClientTransactionHandler.java

```java
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
    abstract void sendMessage(int what, Object obj);
```

看到这是不是突然觉得很熟悉，不急，我们一步一步来分析！在这个方法中会调用sendMessage方法，而在ClientTransactionHandler类中该sendMessage方法为抽象方法，其具体实现在子类ActivityThread中，如下

> frameworks/base/services/core/java/com/android/server/am/ActivityThread.java

```java
    void sendMessage(int what, Object obj) {
        sendMessage(what, obj, 0, 0, false);
    }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```

上面的代码很容易理解，在ActivityThread的sendMessage中会把启动Activity的消息发送给mH,而mH为H类型，其实就是ActivityThread的Handler。到这里AMS就将启动Activity的请求发送给了ActivityThread的Handler。

# 三、ActivityThread的Handler处理启动Activity的请求

## 1. 时序图

![1566484294523](ActivityThread的Handler处理启动Activity的请求.png)

## 2. 详细过程

既然发送了启动Activity的消息，那么就得ActivityThread当然得处理这个消息，我们知道Handler机制，如果是通过sendMessage的方法发送消息的话，应该是在Handler的handleMessage处理消息，在这里也同样如此，ActivityThread的Handler就是H，让我们来看看H的handleMessage,看看是否能找到EXECUTE_TRANSACTION消息的处理.

> frameworks/base/core/java/android/app/ActivityThread.H

```java
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
                ......
				//处理事务，处理活动的启动，生命周期的转化等	
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
					//切换Activity的状态
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
                ....
            }
            ...
        }
```

果真如此，我们H接受到AMS发来的EXECUTE_TRANSACTION消息后，将调用TransactionExecutor.execute方法来切换Activity状态。

> frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java

```java
    public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);
        //先执行callbacks，当根活动首次启动时会有callback，callback会执行onCreate方法
        //其它活动切换状态时没有callback
        executeCallbacks(transaction);
        //改变活动的生命周期状态
        executeLifecycleState(transaction);
        mPendingActions.clear();
        log("End resolving transaction");
    }
```

在execute方法中重点关注两个方法：executeCallbacks，由于我们现在分析的是根Activity的启动流程，从上面也可以知道此时的callback是不为null的，所以会执行executeCallbacks，最终会执行Activity的onCreate方法.此时根Activity就已经启动成功了。executeLifecycleState方法是用来改变活动的生命周期状态的，如果是根Activity的启动，最终将会执行onStart和onResume方法。下面来分析这两个方法

### 2.1 执行onCreate方法

来看看executeCallbacks这个方法。

> frameworks\base\core\java\android\app\servertransaction\TransactionExecutor.java

```java
    @VisibleForTesting
    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null) {
            // No callbacks to execute, return early.
            return;
        }
        ......
        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            //此时item为LaunchActivityItem
            final ClientTransactionItem item = callbacks.get(i);
            ......
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            ......
        }
    }
```

从上面的分析我们知道根Activity启动时这个callback的ClientTransactionItem为LaunchActivityItem，所以会执行LaunchActivityItem的execute方法。

> frameworks\base\core\java\android\app\servertransaction\LaunchActivityItem.java

```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client);
		//此client为ActivityThread
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

上面的client为ActivityThread，所以会继续调用ActivityThread的handleLaunchActivity方法，如下。

```java
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        mSomeActivitiesChanged = true;
        //启动Activity
        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            .....
        } else {
            //如果出现错误就通知AMS停止活动
            try {
				//停止Activity
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
```

在上述方法中将调用performLaunchActivity来启动活动，如下

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //ActivityInfo用于存储代码和AndroidManifes设置的Activity和receiver节点信息，
        //比如Activity的theme和launchMode
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
			//获取APK文件的描述类LoadedApk
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
		//获取要启动的Activity的ComponentName类，ComponentName类中保存了该Activity的包名和类名
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

		
        //创建要启动Activity的上下文环境
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();

		    //用类加载器来创建该Activity的实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        }
        ...

        try {
			//创建Application,makeApplication会调用Application的onCreate方法
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            ...
            if (activity != null) {
                ....
                //初始化Activity，创建Window对象（PhoneWindow）并实现Activity和Window相关联
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                ....
                //设置主题    
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //启动活动
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ....
            }

			//设置生命周期为onCreate
            r.setState(ON_CREATE);
        } 
        ...
        return activity;
    }
```

该方法代码很多，要干的事情也挺多的，并且都很重要，通过注释应该可以知道这个方法的任务，当干完所有事后得将当前的生命周期设置为ON_CREATE。我们重点关注启动活动的代码mInstrumentation.callActivityOnCreate，可以知道在这里将会调用Instrumentation的callActivityOnCreate来启动活动。

> frameworks\base\core\java\android\app\Instrumentation.java

```java
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

而callActivityOnCreate将调用Activity的performCreate，如下

> frameworks\base\core\java\android\app\Activity.java

```java
    final void performCreate(Bundle icicle) {
        performCreate(icicle, null);
    }

    @UnsupportedAppUsage
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        ....
		//调用Activity的onCreate，根Activity启动成功
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        ...
    }
```

看得到这眼泪是否哗啦啦啦的流下了，终于看到熟悉的东西了，在Activity的performCreate中将执行onCreate，也就是我们在开发中所熟悉的onCreate,“终于等到你，还好我们没放弃”，讲到这，根Activity就已经启动了，我们的万里长征也应该结束了，但是身为万里长征的首席指挥官，总不能一结束就撒手不管吧。所以让我们继续来看看结束后的维护工作。

### 2.2 执行onStart方法

从上面分析中我们知道onCreate已经被调用，且生命周期的状态是ON_CREATE，故executeCallbacks已经执行完毕，所以开始执行executeLifecycleState方法。如下

> frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java

```java
    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        if (lifecycleItem == null) {
            // No lifecycle request, return early.
            return;
        }
        ....
        // 执行当前生命周期状态之前的状态
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

        //切换状态
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```

在executeLifecycleState方法中，会先执行cycleToPath，从上面的分析我们已经知道当根Activity启动时，此时的lifecycleItem为ResumeActivityItem，故调用lifecycleItem.getTargetState时将得到ON_RESUME状态，让我们来瞧瞧cycleToPath方法。

```java
    private void cycleToPath(ActivityClientRecord r, int finish,
            boolean excludeLastState) {
        final int start = r.getLifecycleState();
        log("Cycle from: " + start + " to: " + finish + " excludeLastState:" + excludeLastState);
        final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
        performLifecycleSequence(r, path);
    }
```

由于onCreate方法已经执行，所以start为ON_CREATE,而finish为上面传递的ON_RESUME,excludeLastState是否移除最后的状态为true。让我们来看看getLifecyclePath这个方法

> frameworks/base/core/java/android/app/servertransaction/TransactionExecutorHelper.java

```java
    public IntArray getLifecyclePath(int start, int finish, boolean excludeLastState) {
        ...
        if (finish >= start) {
            // 添加start到finish之间的周期状态
            for (int i = start + 1; i <= finish; i++) {
                mLifecycleSequence.add(i);
            }
        } 
        ...
        // 根据需求移除最后的状态
        if (excludeLastState && mLifecycleSequence.size() != 0) {
            mLifecycleSequence.remove(mLifecycleSequence.size() - 1);
        }

        return mLifecycleSequence;
    }

public abstract class ActivityLifecycleItem extends ClientTransactionItem {
    .....
    public static final int UNDEFINED = -1;
    public static final int PRE_ON_CREATE = 0;
    public static final int ON_CREATE = 1;
    public static final int ON_START = 2;
    public static final int ON_RESUME = 3;
    public static final int ON_PAUSE = 4;
    public static final int ON_STOP = 5;
    public static final int ON_DESTROY = 6;
    public static final int ON_RESTART = 7;
    ...
}

```

在getLifecyclePath这个方法我们知道start为ON_CREATE,finish为ON_RESUME,那么如何知道start和finish的大小关系呢？在ActivityLifecycleItem中我们可以发现这些状态的定义，从定义中可以发现finish>=start，所以我们只关注这部分的逻辑处理，可以发现ON_CREATE和ON_RESUME中间还有ON_START这个状态，所以mLifecycleSequence将添加ON_START和ON_RESUME状态，但是又因为excludeLastState为true，所以最后会移除掉ON_RESUME状态，故返回的类型只包含ON_START状态。故cycleToPath方法中的path中将只包含ON_START状态，然后继续执行performLifecycleSequence方法。

```java
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            log("Transitioning to state: " + state);
            switch (state) {
                .....
                case ON_START:
                    mTransactionHandler.handleStartActivity(r, mPendingActions);
                    break;
                 ......
            }
        }
    }
```

因为path只包含ON_START状态，所以只执行ActivityThread的handleStartActivity方法。

> frameworks/base/core/java/android/app/ActivityThread.java

```java
	public void handleStartActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions) {
        ...
        // Start
        activity.performStart("handleStartActivity");
        r.setState(ON_START);
        ...
    }
```

> frameworks/base/core/java/android/app/Activity.java

```java
final void performStart(String reason) {
    ...
     mInstrumentation.callActivityOnStart(this);
    ...
}

```

> frameworks/base/core/java/android/app/Instrumentation.java

```java
    public void callActivityOnStart(Activity activity) {
        activity.onStart();
    }

```

从上面可以发现ActivityThread.handleStartActivity经过多次跳转最终会执行activity.onStart方法，至此cycleToPath方法执行完毕。

### 2.3 执行onResume方法

让我们回到executeLifecycleState这个方法，执行完cycleToPath方法后将执行lifecycleItem.execute，即ResumeActivityItem的execute方法。

> frameworks/base/core/java/android/app/servertransaction/ResumeActivityItem.java

```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
        //此client为ActivityThread
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward,
                "RESUME_ACTIVITY");
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
```

> frameworks/base/core/java/android/app/ActivityThread.java

```java
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...
        // TODO Push resumeArgs into the activity for consideration
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ...
        Looper.myQueue().addIdleHandler(new Idler());
        ...
    }

    public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        ...
        try {
            ...
            r.activity.performResume(r.startsNotResumed, reason);
            r.state = null;
            r.persistentState = null;
            //设置当前状态为ON_RESUME
            r.setState(ON_RESUME);
        } 
        ...
        return r;
    }
```

> frameworks/base/core/java/android/app/Activity.java

```java
    final void performResume(boolean followedByPause, String reason) {
        performRestart(true /* start */, reason);
        ...
        // mResumed is set by the instrumentation
        mInstrumentation.callActivityOnResume(this);
        ...
    }
```

> frameworks/base/core/java/android/app/Instrumentation.java

```java
    public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        ...
    }
```

可以发现其实执行onCreate，onStart,onResume的流程大致上是类似的，通过多次调用最终将会调用Activity的onResume方法，至此Activity真正启动完毕！

# 总结

理解根Activity的启动流程至关重要，在此次分析根Activity的启动流程中对Activity的生命周期也有了更深的了解，由于在分析过程中是基于Android9.0的，与之前的版本有些改动，有时候会陷入源码的沼泽中，越挣扎陷的越深。所以我觉得我们应该借助源码来把握整体流程，但又不能纠结于读懂源码中的任何代码，这是不太现实的。

两篇文章结合起来，我们可以得知Activity的启动的整体流程：

1. Launcher进程请求AMS
2. AMS发送创建应用进程请求
3. Zygote进程接受请求并孵化应用进程
4. 应用进程启动ActivityThread
5. 应用进程绑定到AMS
6. AMS发送启动Activity的请求
7. ActivityThread的Handler处理启动Activity的请求

> 参考博客：
>
> - [（Android 9.0）Activity启动流程源码分析](https://blog.csdn.net/lj19851227/article/details/82562115)
> - [Android8.0 根Activity启动过程（后篇）](http://liuwangshu.cn/framework/component/7-activity-start-2.html)

