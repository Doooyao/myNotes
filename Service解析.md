### Service

这次简单分析一下一个Service的生命周期~
之所以先从Service开始是因为Service相对Activity会少一些,不涉及各种栈和视图
话不多说,一般我们启动一个Service有两种方式,先来看看startService
#### startService
##### startService

这个方法是context中的,真正对他的实现是ContextImpl,我们看一下代码

```java
 @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();//如果是系统进程的话,警告
        return startServiceCommon(service, false, mUser);//这里的三个参数,第一个就是intent,第二个表示是不是前台服务,第三个表示当前用户
    }
    private ComponentName startServiceCommon(Intent service, boolean requireForeground,
            UserHandle user) {
        try {
            validateServiceIntent(service);//验证合法性
            service.prepareToLeaveProcess(this);//准备离开进程
            ComponentName cn = ActivityManager.getService().startService( //这里就可以看到,实际上还是调用了ams中的方法
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), requireForeground,
                            getOpPackageName(), user.getIdentifier());
           //```省略判断错误的代码```
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    //上面的代码中有一个验证合法性的,我们来看一下
    private void validateServiceIntent(Intent service) {
        if (service.getComponent() == null && service.getPackage() == null) {//这里可以看到,如果intent没有指明Component和Package
            //及没有显性声明,在LOLLIPOP以上是不行的,会报错
            if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                IllegalArgumentException ex = new IllegalArgumentException(
                        "Service Intent must be explicit: " + service);
                throw ex;
            } else {
                Log.w(TAG, "Implicit intents with startService are not safe: " + service
                        + " " + Debug.getCallers(2, 3));
            }
        }
    }
```
2. AMS 下面我们看看Ams中被调用的方法,尽量把整个流程贴上~但是会省略一些代码~
```java
public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, boolean requireForeground, String callingPackage, int userId)
            throws TransactionTooLargeException {
        enforceNotIsolatedCaller("startService");//方法1 作安全性检查，判断当前用户是否允许启动 这个方法我们下面会看一下
        if (service != null && service.hasFileDescriptors() == true) {//不能有FileDescriptors
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        //省略一些判断和打印
        synchronized(this) {
            final int callingPid = Binder.getCallingPid();//拿到调用者的pid uid
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();//拿到调用者的id,设置当前进程的id进去
            ComponentName res;
            try {
                res = mServices.startServiceLocked(caller, service,
                        resolvedType, callingPid, callingUid,
                        requireForeground, callingPackage, userId);//这里调用了ActivityService的方法来启动Service
            } finally {
                Binder.restoreCallingIdentity(origId);//还原调用者的id
            }
            return res;
        }
    }
```
方法1:
```java
void enforceNotIsolatedCaller(String caller) {
        if (UserHandle.isIsolated(Binder.getCallingUid())) {//作安全性检查，判断请求和当前进程是不是同在一个用户下
            throw new SecurityException("Isolated process not allowed to call " + caller);
        }
    }
```
##### ActivityService 

//这个类就比较重要了,Service相关很多处理判断都是在这个类中完成的 我们看一下开启Service的代码

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        final boolean callerFg; //是否是前台调用的
        if (caller != null) {
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);//获取到调用者进程的记录信息
            if (callerApp == null) {//没有信息,报错
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + callingPid
                        + ") when starting service " + service);
            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;//是否是前台调用的
        } else {
            callerFg = true;//默认前台调用?这里不是很清楚
        }

        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false);//生成一个记录service的东西 这个方法等下再看
        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }

        ServiceRecord r = res.record;

        if  (!r.startRequested && !fgRequired) {

            <!--检测当前app是否允许后台启动-->
            final int allowed = mAm.getAppStartModeLocked(r.appInfo.uid, r.packageName,
                    r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
                    <!--如果不允许  Background start not allowed-->
            if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                ...
                <!--返回 ? 告诉客户端现在处于后台启动状态，禁止你-->
                return new ComponentName("?", "app is in background uid " + uidRec);
            }
        }
        //```省略了一些判断和打印```

        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);//上面省略了很多,我们直接看这个方法
        return cmp;
    }
```
下面我们先看一下上面调用的几个方法
1. AMS.getRecordForAppLocked
这个方法返回了一个进程记录ProcessRecord,实际在AMS中通过Lru保管了当前各个进程状态
2. callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND 我们看一下这个
setSchedGroup是AMS管理进程的一个参考,有三个值
static final int SCHED_GROUP_BACKGROUND = 0;
static final int SCHED_GROUP_DEFAULT = 1;
static final int SCHED_GROUP_TOP_APP = 2;
只有setSchedGroup==ProcessList.SCHED_GROUP_BACKGROUND的进程才被AMS看做后台进程 暂时先知道怎么多就行
下面我们再看看这个方法
3. retrieveServiceLocked
```java
//我们先看看他的返回类型
private final class ServiceLookupResult {
        final ServiceRecord record;
        final String permission;
        //看起来很简单,封装了ServiceRecord和一个权限
        //而这个ServiceRecord 就是一个Service的记录,继承Binder
        ServiceLookupResult(ServiceRecord _record, String _permission) {
            record = _record;
            permission = _permission;
        }
    }
//下面看看retrieveServiceLocked这个方法,比较多,我们挑重要的看
 private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, String callingPackage, int callingPid, int callingUid, int userId,
            boolean createIfNeeded, boolean callingFromFg, boolean isBindExternal) {
        ServiceRecord r = null;
        userId = mAm.mUserController.handleIncomingUser(callingPid, callingUid, userId, false,
                ActivityManagerService.ALLOW_NON_FULL_IN_PROFILE, "service", null);
        //上面这个方法也是用来判断用户权限的,这里上一下ALLOW_NON_FULL_IN_PROFILE的注释
        // We may or may not allow this depending on whether the two users are
        // in the same profile.现在先不管,往下继续看
        ServiceMap smap = getServiceMapLocked(userId);//根据userId获取ServiceMap 这个方法就是从一个SparseArray中查找,没有就new一个存进去
        //这个servicemap就是一个userid的所有service的信息,是一个Handler
        final ComponentName comp = service.getComponent();//获取组件名称
        if (comp != null) {
            r = smap.mServicesByName.get(comp);//从这个handler中查找有没有这个Service的记录
        }
        if (r == null && !isBindExternal) {//如果没有并且不允许运行在进程外面?这个第二个参数现在还不是很清楚
            Intent.FilterComparison filter = new Intent.FilterComparison(service);//这个Intent.FilterComparison是我第一次见到,等下会分析
            r = smap.mServicesByIntent.get(filter);//我们再通过这个intent查一下有没有service记录
        }
        if (r != null && (r.serviceInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0
                && !callingPackage.equals(r.packageName)) {
            // If an external service is running within its own package, other packages
            // should not bind to that instance.//这里的注释很明白了,如果有,但是是其他包在调用的,我们不能用啊
            r = null;
        }
        if (r == null) {//如果没有缓存可用的
            try {
                ResolveInfo rInfo = mAm.getPackageManagerInternalLocked().resolveService(service,
                        resolvedType, ActivityManagerService.STOCK_PM_FLAGS
                                | PackageManager.MATCH_DEBUG_TRIAGED_MISSING,
                        userId, callingUid);//我们解析这个service的intent
                ServiceInfo sInfo =
                    rInfo != null ? rInfo.serviceInfo : null;//获取要启动的serviceInfo
                if (sInfo == null) {//找不到Service
                    return null;
                }
                ComponentName name = new ComponentName(
                        sInfo.applicationInfo.packageName, sInfo.name);//根据找到的信息新建一个组件名
                if ((sInfo.flags & ServiceInfo.FLAG_EXTERNAL_SERVICE) != 0) {//如果允许外部调用
                    if (isBindExternal) {//是外部调用
                        if (!sInfo.exported) {//manifast没写这个exported,报错
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not exported");
                        }
                        if ((sInfo.flags & ServiceInfo.FLAG_ISOLATED_PROCESS) == 0) {//没有独立进程的识别,报错
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                                    " is not an isolatedProcess");
                        }
                        // Run the service under the calling package's application.
                        ApplicationInfo aInfo = AppGlobals.getPackageManager().getApplicationInfo(
                                callingPackage, ActivityManagerService.STOCK_PM_FLAGS, userId);//拿到应用信息
                        if (aInfo == null) {
                            throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " +
                                    "could not resolve client package " + callingPackage);
                        }
                        sInfo = new ServiceInfo(sInfo);//重新生成serviceinfo的副本
                        sInfo.applicationInfo = new ApplicationInfo(sInfo.applicationInfo);//设置相应的应用信息
                        sInfo.applicationInfo.packageName = aInfo.packageName;
                        sInfo.applicationInfo.uid = aInfo.uid;
                        name = new ComponentName(aInfo.packageName, name.getClassName());
                        service.setComponent(name);//设置给intent
                    } else {
                        throw new SecurityException("BIND_EXTERNAL_SERVICE required for " +
                                name);
                    }
                } else if (isBindExternal) {//报错
                    throw new SecurityException("BIND_EXTERNAL_SERVICE failed, " + name +
                            " is not an externalService");
                }
                if (userId > 0) {//这里，如果userid>0 
                    if (mAm.isSingleton(sInfo.processName, sInfo.applicationInfo,
                            sInfo.name, sInfo.flags)
                            && mAm.isValidSingletonCall(callingUid, sInfo.applicationInfo.uid)) {
                        userId = 0;
                        smap = getServiceMapLocked(0);//重新取map
                    }
                    sInfo = new ServiceInfo(sInfo);//还是生成serviceinfo
                    sInfo.applicationInfo = mAm.getAppInfoForUser(sInfo.applicationInfo, userId);
                }
                r = smap.mServicesByName.get(name);//再找一次
                if (DEBUG_SERVICE && r != null) Slog.v(TAG_SERVICE,
                        "Retrieved via pm by intent: " + r);
                if (r == null && createIfNeeded) {//如果之前还是没有而且要创建的话
                    final Intent.FilterComparison filter
                            = new Intent.FilterComparison(service.cloneFilter());//=把filter拿出来
                    final ServiceRestarter res = new ServiceRestarter();//重启者
                    final BatteryStatsImpl.Uid.Pkg.Serv ss;
                    final BatteryStatsImpl stats = mAm.mBatteryStatsService.getActiveStatistics();
                    synchronized (stats) {
                        ss = stats.getServiceStatsLocked(
                                sInfo.applicationInfo.uid, sInfo.packageName,
                                sInfo.name);
                    }
                    r = new ServiceRecord(mAm, ss, name, filter, sInfo, callingFromFg, res);//生成一个新的servicerecord
                    res.setService(r);//设置要重启的service
                    smap.mServicesByName.put(name, r);//根绝name和intent分别存放进去
                    smap.mServicesByIntent.put(filter, r);

                    // Make sure this component isn't in the pending list.
                    for (int i=mPendingServices.size()-1; i>=0; i--) {
                        final ServiceRecord pr = mPendingServices.get(i);
                        if (pr.serviceInfo.applicationInfo.uid == sInfo.applicationInfo.uid
                                && pr.name.equals(name)) {
                            mPendingServices.remove(i);//如果在等待启动的列表里，移除
                        }
                    }
                }
            } catch (RemoteException ex) {
            }
        }
        if (r != null) {//如果现在有record了
            if (mAm.checkComponentPermission(r.permission,//检查需要的权限
                    callingPid, callingUid, r.appInfo.uid, r.exported) != PERMISSION_GRANTED) {
                if (!r.exported) {
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                }
                return new ServiceLookupResult(null, r.permission);
            } else if (r.permission != null && callingPackage != null) {
                final int opCode = AppOpsManager.permissionToOpCode(r.permission);
                if (opCode != AppOpsManager.OP_NONE && mAm.mAppOpsService.noteOperation(
                        opCode, callingUid, callingPackage) != AppOpsManager.MODE_ALLOWED) {
                    return null;
                }
            }

            if (!mAm.mIntentFirewall.checkService(r.name, service, callingUid, callingPid,
                    resolvedType, r.appInfo)) {
                return null;
            }
            return new ServiceLookupResult(r, null);
        }
        return null;//没找到
    }
```
上面的方法分析完了，最后进入了startServiceInnerLocked这个方法，我们再看一下

```java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
       
        r.callStart = false;//开始创建
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startRunningLocked();//记录下开始运行了
        }
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);//开启service//从这个方法继续
        if (error != null) {
            return new ComponentName("!!", error);
        }

        //```省略了```

        return r.name;
    }
```
bringUpServiceLocked
```java
 private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        if (r.app != null && r.app.thread != null) {//这种情况表示service已经运行了
            sendServiceArgsLocked(r, execInFg, false);//发送参数,直接返回
            return null;
        }
        if (!whileRestarting && mRestartingServices.contains(r)) {//正在等待重启,啥也不干
            return null;
        }
        // restarting state.
        if (mRestartingServices.remove(r)) {//因为现在正在启动,就移除restart的任务和状态了
            clearRestartingIfNeededLocked(r);
        }

        // 确保此服务不再被视为延迟，我们现在就开始使用。
        if (r.delayed) {
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);//把这个service从延时启动的service中移除
            r.delayed = false;
        }

        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);//如果拥有这个service的userid已经停止了,就停止这个service的启动
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }
//是否是独立进程
        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;//进城名
        String hostingType = "service";
        ProcessRecord app;

        if (!isolated) {//不是独立进程
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);//拿出要启动的进程记录
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                    realStartServiceLocked(r, app, execInFg);//启动Service
                    return null;
                }
            }
        } else {
            app = r.isolatedProc;//等一个独立进程防止每次startservice都创建新的进程
            if (WebViewZygote.isMultiprocessEnabled()
                    && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                hostingType = "webview_service";
            }
        }
        
        if (app == null && !permissionsReviewRequired) {//没有可用进程而且不用等待权限
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false)) == null) {//尝试开启一个进程并使用此进程运行service
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);//请求进程如果返回null,停止service的启动
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;//把请求来的进程设置给record
            }
        }

        if (r.fgRequired) {//前台请求的
            mAm.tempWhitelistUidLocked(r.appInfo.uid,
                    SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
        }

        if (!mPendingServices.contains(r)) {//即将启动的service+1
            mPendingServices.add(r);
        }

        if (r.delayedStop) {//已经请求停止了,那么就停止service~
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }
```
下面我们先看看非独立进程下,直接调用了realStartServiceLocked 还是一样,省略了一些log打印
```java
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");//方法1
        mAm.updateLruProcessLocked(app, false, null);//进程管理相关,暂时先不看
        updateServiceForegroundLocked(r.app, /* oomAdj= */ false);//更新服务前台
        mAm.updateOomAdjLocked();//进程管理相关,暂时不看

        boolean created = false;
        try {
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);//这里可以看到通过ApplicationThread发送了创建Service的消息
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            mAm.appDiedLocked(app);//异常
            throw e;
        } finally {
            if (!created) {//创建失败
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);//是否destroying
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);//尝试重启
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }

        requestServiceBindingsLocked(r, execInFg);//请求服务绑定

        updateServiceClientActivitiesLocked(app, null, true);

        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null, 0));
        }

        sendServiceArgsLocked(r, execInFg, true);//发送参数

        if (r.delayed) {
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                stopServiceLocked(r);
            }
        }
    }
```
下面我们依次查看一下上面的几个方法
bumpServiceExecutingLocked

```java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        long now = SystemClock.uptimeMillis();
        if (r.executeNesting == 0) {
            r.executeFg = fg;
            ServiceState stracker = r.getTracker();
            if (stracker != null) {
                stracker.setExecuting(true, mAm.mProcessStats.getMemFactorLocked(), now);
            }
            if (r.app != null) {
                r.app.executingServices.add(r);//添加进正在启动的队列
                r.app.execServicesFg |= fg;
                if (r.app.executingServices.size() == 1) {
                    scheduleServiceTimeoutLocked(r.app);
                }
            }
        } else if (r.app != null && fg && !r.app.execServicesFg) {
            r.app.execServicesFg = true;
            scheduleServiceTimeoutLocked(r.app);
        }
        r.executeFg |= fg;
        r.executeNesting++;
        r.executingStart = now;
    }
    //上面这个方法中,除了一些追踪记录和状态改变,还有一个比较重要的方法 scheduleServiceTimeoutLocked 我们看一下
    void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);//创建了一个msg
        msg.obj = proc;
        mAm.mHandler.sendMessageDelayed(msg,//通过AMS的handler延时发送了这个msg 延时事件按照是否前台服务分别是20s和200s
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
    //下面我们看看消息接受方 这个handler位于Ams中
    case SERVICE_TIMEOUT_MSG: {
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
    //又回到了咱们的ActivityService里面
    void serviceTimeout(ProcessRecord proc) {//接收到这个消息,就说明ANR了
        //具体实现暂时省略了
    }
```
下面是ApplicationThread中创建service的方法scheduleCreateService
```java
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            sendMessage(H.CREATE_SERVICE, s);//可以看到这里发送了一个createservice的msg
        }
        //收到消息后 ActivityThread处理
case CREATE_SERVICE:
            handleCreateService((CreateServiceData)msg.obj);
            break;
private void handleCreateService(CreateServiceData data) {//创建Service 隐藏部分打印和异常报错
        unscheduleGcIdler();
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();//通过反射获取Service实例
        } catch (Exception e) {}
        try {
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);//创建所需的context并和service绑定

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            service.onCreate();//这里调用了oncreate方法 由此可知这个方法是运行在主线程的
            mServices.put(data.token, service);//存起来
            try {
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);//创建成功
            } catch (RemoteException e) {}
        } catch (Exception e) {}
    }
下面我们看看service.attach方法
    public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context);
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion
                < Build.VERSION_CODES.ECLAIR;
    }
    //实际上就是设置绑定了一些需要的属性和context
    //再看看ActivityManagerService.serviceDoneExecuting
    public void serviceDoneExecuting(IBinder token, int type, int startId, int res) {
        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                Slog.e(TAG, "serviceDoneExecuting: Invalid service token=" + token);
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.serviceDoneExecutingLocked((ServiceRecord)token, type, startId, res);
        }
    }
    void serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res) {//上面传下来这里的type是SERVICE_DONE_EXECUTING_ANON
        boolean inDestroying = mDestroyingServices.contains(r);
        if (r != null) {
            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) { //省略
            } else if (type == ActivityThread.SERVICE_DONE_EXECUTING_STOP) {//省略
            }
            final long origId = Binder.clearCallingIdentity();
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);//可以看到,直接跳到了这一步
            Binder.restoreCallingIdentity(origId);
        } else {//省略log
        }
    }
    private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        r.executeNesting--;
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);//这里可以看到,移除了anr通知!
                } else if (r.executeFg) {
                    // Need to re-evaluate whether the app still needs to be in the foreground.
                    for (int i=r.app.executingServices.size()-1; i>=0; i--) {
                        if (r.app.executingServices.valueAt(i).executeFg) {
                            r.app.execServicesFg = true;
                            break;
                        }
                    }
                }
                if (inDestroying) {
                    mDestroyingServices.remove(r);
                    r.bindings.clear();
                }
                mAm.updateOomAdjLocked(r.app, true);
            }
            r.executeFg = false;
            if (finishing) {
                if (r.app != null && !r.app.persistent) {
                    r.app.services.remove(r);
                    if (r.whitelistManager) {
                        updateWhitelistManagerLocked(r.app);
                    }
                }
                r.app = null;
            }
        }
    }
```
下面我们再看看之前提到的另一个发送参数的方法sendServiceArgsLocked

```java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }
        ArrayList<ServiceStartArgs> args = new ArrayList<>();

        while (r.pendingStarts.size() > 0) {
            ServiceRecord.StartItem si = r.pendingStarts.remove(0);//取出要传递的参数
            if (si.intent == null && N > 1) {
                continue;
            }
            si.deliveredTime = SystemClock.uptimeMillis();
            r.deliveredStarts.add(si);
            si.deliveryCount++;
            if (si.neededGrants != null) {
                mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                        si.getUriPermissionsLocked());//检查权限
            }
            mAm.grantEphemeralAccessLocked(r.userId, si.intent,
                    r.appInfo.uid, UserHandle.getAppId(si.callingId));
            bumpServiceExecutingLocked(r, execInFg, "start");//start方法
            if (!oomAdjusted) {
                oomAdjusted = true;
                mAm.updateOomAdjLocked(r.app, true);
            }
            if (r.fgRequired && !r.fgWaiting) {
                if (!r.isForeground) {
                    scheduleServiceForegroundTransitionTimeoutLocked(r);
                } else {
                    r.fgRequired = false;
                }
            }
            int flags = 0;
            if (si.deliveryCount > 1) {
                flags |= Service.START_FLAG_RETRY;
            }
            if (si.doneExecutingCount > 0) {
                flags |= Service.START_FLAG_REDELIVERY;
            }
            args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));//把intent添加进去
        }

        ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
        slice.setInlineCountLimit(4);
        Exception caughtException = null;
        try {
            r.app.thread.scheduleServiceArgs(r, slice);//还是通过ActivityThread中的ApplicationThread发送消息通知
        } catch (Exception e) {//这里我省略了一些异常的捕获和处理
            caughtException = e;
        }

        if (caughtException != null) {
            final boolean inDestroying = mDestroyingServices.contains(r);
            for (int i = 0; i < args.size(); i++) {
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);//出现异常,调用done
            }
            if (caughtException instanceof TransactionTooLargeException) {
                throw (TransactionTooLargeException)caughtException;
            }
        }
    }
    //再看看ActivityThread和ApplicationThread中
    //一样的套路
    public final void scheduleServiceArgs(IBinder token, ParceledListSlice args) {
            List<ServiceStartArgs> list = args.getList();
            for (int i = 0; i < list.size(); i++) {
                ServiceStartArgs ssa = list.get(i);
                ServiceArgsData s = new ServiceArgsData();
                s.token = token;
                s.taskRemoved = ssa.taskRemoved;
                s.startId = ssa.startId;
                s.flags = ssa.flags;
                s.args = ssa.args;
                sendMessage(H.SERVICE_ARGS, s);
            }
        }
    //ActivityThread处理msg    
    case SERVICE_ARGS:
             handleServiceArgs((ServiceArgsData)msg.obj);
             break;
    private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);//通过这个token拿到相应的service
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();//准备进入进程
                }
                int res;
                if (!data.taskRemoved) {
                    res = s.onStartCommand(data.args, data.flags, data.startId);//调用service的onStartCommand
                } else {
                    s.onTaskRemoved(data.args);//调用service的onTaskRemoved
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }
                QueuedWork.waitToFinish();
                try {
                    ActivityManager.getService().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);//还是调用serviceDoneExecuting,不过这次的type不一样了~
                } catch (RemoteException e) {}
                ensureJitEnabled();
            } catch (Exception e) {}
        }
    }
    //我们之前看过这个方法,不过这种类型的判断我们省略了,下面我们补上看看
    if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                r.callStart = true;
                switch (res) {//这里我们就知道了,onStartCommand的返回值都有什么区别呢 可以看注释分析一下
                    case Service.START_STICKY_COMPATIBILITY:
                    case Service.START_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, true);//找到并移除这个参数
                        // Don't stop if killed.
                        r.stopIfKilled = false;
                        break;
                    }
                    case Service.START_NOT_STICKY: {
                        // We are done with the associated start arguments.
                        r.findDeliveredStart(startId, true);//找到并移除
                        if (r.getLastStartId() == startId) {
                            // There is no more work, and this service doesn't want to hang around if killed.//如果是最后一个
                            r.stopIfKilled = true;
                        }
                        break;
                    }
                    case Service.START_REDELIVER_INTENT: {
                        // We'll keep this item until they explicitly
                        // call stop for it, but keep track of the fact
                        // that it was delivered.
                        ServiceRecord.StartItem si = r.findDeliveredStart(startId, false);//找到参数但是不移除
                        if (si != null) {
                            si.deliveryCount = 0;
                            si.doneExecutingCount++;
                            // Don't stop if killed.  这个解释有些奇葩 
                       //<!--不需要立即重启 START_REDELIVER_INTENT的时候，依靠的是deliveredStarts触发重启-->
                            // Don't stop if killed.
                            r.stopIfKilled = true;//true
                        }
                        break;
                    }
                    case Service.START_TASK_REMOVED_COMPLETE: {
                        // Special processing for onTaskRemoved().  Don't
                        // impact normal onStartCommand() processing.
                        r.findDeliveredStart(startId, true);//找到并移除
                        break;
                    }
                    default:
                        throw new IllegalArgumentException(
                                "Unknown service start result: " + res);
                }
                if (res == Service.START_STICKY_COMPATIBILITY) {
                    r.callStart = false;//
                }
    //下面我们从网上找来了上面几个参数的效果,对比看看~
START_STICKY：如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。
随后系统会尝试重新创建service，由于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。
如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null。

START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统不会自动重启该服务。

START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，
服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。

START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。
```
上面其实还有两个方法我给偷偷省略了,下面我们来看看
1. requestServiceBindingsLocked 这个方法实际上就是通过ActivityThread调用onBind ,我们一会儿再看
2. updateServiceClientActivitiesLocked 这个也和bind相关 我们等等再看
除此之外,一个startService的顺序我们就彻底分析完了,下面我们再看看启动Service的另一方法

#### bindService
应该猜的出来,这个方法一样在ContextImpl中 我们直接上最后调用的方法
```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
        IServiceConnection sd;
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        }
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess(this);
            int res = ActivityManager.getService().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());//可以看到,这里还是调用的AMS的方法 我们快速追踪一下
            return res != 0;
        } 
    }
```
```java
public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {
        enforceNotIsolatedCaller("bindService");
        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, callingPackage, userId);//最后调用ActivityService中的方法
        }
    }
```
下面这个方法有点长,我们一点点分析一下 照例我们省略一些异常报错和log代码
```java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {
       
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        ActivityRecord activity = null;
        if (token != null) {
            activity = ActivityRecord.isInStackLocked(token);
            if (activity == null) {//这里可以看到,必须是要绑定一个activity的
                return 0;
            }
        }

        int clientLabel = 0;
        PendingIntent clientIntent = null;
        final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;//是否是系统调用

        if (isCallerSystem) {
            // Hacky kind of thing -- allow system stuff to tell us
            // what they are, so we can report this elsewhere for
            // others to know why certain services are running.
            service.setDefusable(true);
            clientIntent = service.getParcelableExtra(Intent.EXTRA_CLIENT_INTENT);
            if (clientIntent != null) {
                clientLabel = service.getIntExtra(Intent.EXTRA_CLIENT_LABEL, 0);
                if (clientLabel != 0) {
                    // There are no useful extras in the intent, trash them.
                    // System code calling with this stuff just needs to know
                    // this will happen.
                    service = service.cloneFilter();
                }
            }
        }

        if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
            mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                    "BIND_TREAT_LIKE_ACTIVITY");
        }

        if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {//不允许非系统调用此flag
            throw new SecurityException(
                    "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                    + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
        }

        final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;//调用者是否在前台
        final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;//
        //解析Service，并保存在ServiceRecord
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                    Binder.getCallingUid(), userId, true, callerFg, isBindExternal); 
        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        ServiceRecord s = res.record;
        boolean permissionsReviewRequired = false;
        
        if (mAm.mPermissionReviewRequired) {//需要审核权限
            if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(//此用户需要审核权限
                    s.packageName, s.userId)) {

                permissionsReviewRequired = true;

                // Show a permission review UI only for binding from a foreground app
                if (!callerFg) {//如果不是前台请求的绑定,直接返回
                    return 0;
                }

                final ServiceRecord serviceRecord = s;
                final Intent serviceIntent = service;

                RemoteCallback callback = new RemoteCallback(//一个远程回调
                        new RemoteCallback.OnResultListener() {
                    @Override
                    public void onResult(Bundle result) {
                        synchronized(mAm) {
                            final long identity = Binder.clearCallingIdentity();
                            try {
                                if (!mPendingServices.contains(serviceRecord)) {
                                    return;
                                }
                                // If there is still a pending record, then the service
                                // binding request is still valid, so hook them up. We
                                // proceed only if the caller cleared the review requirement
                                // otherwise we unbind because the user didn't approve.
                                if (!mAm.getPackageManagerInternalLocked()
                                        .isPermissionsReviewRequired(
                                                serviceRecord.packageName,
                                                serviceRecord.userId)) {
                                    try {
                                        bringUpServiceLocked(serviceRecord,
                                                serviceIntent.getFlags(),
                                                callerFg, false, false);
                                    } catch (RemoteException e) {
                                        /* ignore - local call */
                                    }
                                } else {
                                    unbindServiceLocked(connection);
                                }
                            } finally {
                                Binder.restoreCallingIdentity(identity);
                            }
                        }
                    }
                });//这个回调里面我们暂时不看~

                final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                        | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
                intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
                intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);
                mAm.mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));//使用Ams的进程开启权限检测
                    }
                });
            }
        }

        final long origId = Binder.clearCallingIdentity();

        try {
            if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {//如果正在重启,先不用重启了,因为接下来即将启动
            }

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {//如果需要自动创建这个
                s.lastActivity = SystemClock.uptimeMillis();
                if (!s.hasAutoCreateConnections()) {//如果这是第一次bind,还没有自动创建
                    // This is the first binding, let the tracker know.
                    ServiceState stracker = s.getTracker();
                    if (stracker != null) {
                        stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                                s.lastActivity);
                    }
                }
            }

            mAm.startAssociationLocked(callerApp.uid, callerApp.processName, callerApp.curProcState,
                    s.appInfo.uid, s.name, s.processName);//
            // Once the apps have become associated, if one of them is caller is ephemeral
            // the target app should now be able to see the calling app
            mAm.grantEphemeralAccessLocked(callerApp.userId, service,
                    s.appInfo.uid, UserHandle.getAppId(callerApp.uid));

            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);//
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);//生成一个链接记录

            IBinder binder = connection.asBinder();//把回调转换成binder
            ArrayList<ConnectionRecord> clist = s.connections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                s.connections.put(binder, clist);//把这个链接的记录存进去
            }
            clist.add(c);
            b.connections.add(c);//
            if (activity != null) {
                if (activity.connections == null) {
                    activity.connections = new HashSet<ConnectionRecord>();
                }
                activity.connections.add(c);
            }
            b.client.connections.add(c);
            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.hasAboveClient = true;
            }
            if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                s.whitelistManager = true;
            }
            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c, true);
            }
            clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<ConnectionRecord>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c);

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {//这里可以看到,我们通过bringUpServiceLocked创建了Activity
                    return 0;
                }
            }

            if (s.app != null) {
                if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    s.app.treatLikeActivity = true;
                }
                if (s.whitelistManager) {
                    s.app.whitelistManager = true;
                }
                // This could have made the service more important.
                mAm.updateLruProcessLocked(s.app, s.app.hasClientActivities
                        || s.app.treatLikeActivity, b.client);
                mAm.updateOomAdjLocked(s.app, true);
            }
            if (s.app != null && b.intent.received) {//这里说明service已经存在了 直接进行回调
                // Service is already running, so we can immediately
                // publish the connection.
                try {
                    c.conn.connected(s.name, b.intent.binder, false);//直接在这里回调了conn的connected
                } catch (Exception e) {}

                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);//请求binding
                }
            } else if (!b.intent.requested) {
                requestServiceBindingLocked(s, b.intent, callerFg, false);//请求binding
            }

            getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        return 1;
    }
```
下面我们先看看如果bind之前,Service已经启动了的话
首先要走的是c.conn.connected
这个c.conn是个啥呢 追踪一下可以知道,是我们传进来的IServiceConnection,也就是一个IBinder
他是怎么实现的呢 我们找到最初传入这个参数的地方
在ContextImpl的bindServiceCommon中
```
sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);//一般写法情况这里都是传的主线程的handler
```
点进去看一下 方法在LoadedApk中
```java
public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);//一个map缓存
            if (map != null) {
                sd = map.get(c);
            }
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler);
            }
            return sd.getIServiceConnection();//最后调用了这个方法
        }
    }
```
再看看ServiceDispatcher
```java
static final class ServiceDispatcher {//服务分发者 //我们省略一些代码,调整一下方法顺序 隐藏了一些get方法
        private final ServiceDispatcher.InnerConnection mIServiceConnection;
        private final ServiceConnection mConnection;
        private final Context mContext;
        private final Handler mActivityThread;
        private final ServiceConnectionLeaked mLocation;
        private final int mFlags;
        private RuntimeException mUnbindLocation;
        private boolean mForgotten;
        //首先是构造方法,属性复制,初始化InnerConnection和ServiceConnectionLeaked 没什么好讲的
        ServiceDispatcher(ServiceConnection conn,
                Context context, Handler activityThread, int flags) {
            mIServiceConnection = new InnerConnection(this);
            mConnection = conn;
            mContext = context;
            mActivityThread = activityThread;
            mLocation = new ServiceConnectionLeaked(null);
            mLocation.fillInStackTrace();
            mFlags = flags;
        }
        //下面看看这个InnerConnection
        private static class InnerConnection extends IServiceConnection.Stub {//可以看到是一个binder通信的服务端
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;//持有分发者的弱引用

            InnerConnection(LoadedApk.ServiceDispatcher sd) {//
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service, boolean dead)//这就是我们最开始上面看到的connected方法了
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();//取出引用
                if (sd != null) {//如果还没被回收,调用connected
                    sd.connected(name, service, dead);//我们追踪一下这个方法
                }
            }
        }
        
        public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityThread != null) {//这里可以看到,如果制定了handler,则交给handler的线程处理
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {//否则直接在这里处理
                doConnected(name, service, dead);
            }
        }
        
        private final class RunConnection implements Runnable {//看看这个run
            RunConnection(ComponentName name, IBinder service, int command, boolean dead) {
                mName = name;
                mService = service;
                mCommand = command;
                mDead = dead;
            }

            public void run() {
                if (mCommand == 0) {//看一看到,我们上面传的命令就是0,所以走这个doConnected
                    doConnected(mName, mService, mDead);
                } else if (mCommand == 1) {//如果传1,走这个
                    doDeath(mName, mService);
                }
            }

            final ComponentName mName;
            final IBinder mService;
            final int mCommand;
            final boolean mDead;
        }
        //下面看一下链接的方法,一般情况下此方法在主线程运行~
        public void doConnected(ComponentName name, IBinder service, boolean dead) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                if (mForgotten) {//这个参数不知道,暂时不看
                    // We unbound before receiving the connection; ignore
                    // any connection received.
                    return;
                }
                old = mActiveConnections.get(name);//查找这个service的链接信息,如果是第一次,那么就是空的
                if (old != null && old.binder == service) {//activity相同,返回的binder相同,那就不用再传了
                    // Huh, already have this one.  Oh well!
                    return;
                }

                if (service != null) {//穿进来的不是null
                    // A new service is being connected... set it all up.//链接
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {//传个null进来代表断开连接
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);//断开旧的
                }
            }

            // If there was an old service, it is now disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);//调用旧的断开的方法
            }
            if (dead) {
                mConnection.onBindingDied(name);//binder是否死了
            }
            // If there is a new service, it is now connected.
            if (service != null) {
                mConnection.onServiceConnected(name, service);//connnected可以看到在这里调用了回调onserviceConnected
            }
        }

        private static class ConnectionInfo {//存了binder和死亡代理
            IBinder binder;
            IBinder.DeathRecipient deathMonitor;
        }
        
        private final ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo> mActiveConnections
            = new ArrayMap<ComponentName, ServiceDispatcher.ConnectionInfo>();//对当前的活动链接进行缓存

        //```其他的代码我们先省略掉~```
    }
```
下面我们再看看这个方法
requestServiceBindingLocked 
```java
private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {//请求服务绑定
        if ((!i.requested || rebind) && i.apps.size() > 0) {//还没请求过或者是重新绑定
            try {
                bumpServiceExecutingLocked(r, execInFg, "bind");//
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);//进程记录切换状态
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);//调用ApplicationThread的bind
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            }
            //```省略异常代码```
        }
        return true;
    }
```
我们看看这个scheduleBindService
```java
public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();//一样的套路
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
            sendMessage(H.BIND_SERVICE, s);
        }
case BIND_SERVICE:
            handleBindService((BindServiceData)msg.obj);
            break;
private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
                        IBinder binder = s.onBind(data.intent);//这里看到看到,我们调用了service的onBind方法,获取了返回的IBinder对象
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);//通过AMS.publishService 调用到ActivityService.publishServiceLocked
                    } else {
                        s.onRebind(data.intent);
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                    ensureJitEnabled();
                } catch (RemoteException ex) {
                    throw ex.rethrowFromSystemServer();
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to bind to service " + s
                            + " with " + data.intent + ": " + e.toString(), e);
                }
            }
        }
    }            
```
publishService这个方法之前也有见过,下面就来看看
```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service;
                    b.requested = true;
                    b.received = true;
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                continue;
                            }
                            try {
                                c.conn.connected(r.name, service, false);//这里调用链接
                            } 
                        }
                    }
                }
                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```



