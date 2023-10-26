# 应用程序与 AMS 的通讯实现

## 1、前言

咱们在许多和`Framework`解析相关的文章中，都会看到`ActivityManagerService`这个类，可是在上层的应用开发中，却不多直接会使用到他。bash

那么咱们为何要学习它的呢，一个最直接的好处就是它是咱们理解应用程序启动过程的基础，只有把和`ActivityManagerService`以及和它相关的类的关系都理解透了，咱们才能理清应用程序启动的过程。app

今天这篇文章，咱们就来对`ActivityManagerService`与应用进程之间的通讯方式作一个简单的总结，这里咱们根据进程之间的通讯方向，分为两个部分来讨论：框架

- 从应用程序进程到管理者进程ide

  **(a) IActivityManager** **(b) ActivityManagerNative** **(c) ActivityManagerProxy** **(d) ActivityManagerService**函数

- 从管理者进程到应用程序进程学习

  **(a) IApplicationThread** **(b) ApplicationThreadNative** **(c) ApplicationThreadProxy** **(d) ApplicationThread**ui

## 2、从应用程序进程到管理者进程

在这一方向上的通讯，应用程序进程做为客户端，而管理者进程则做为服务端。举一个最简单的例子，当咱们启动一个`Activity`，就须要通知全局的管理者，让它去负责启动，这个全局的管理者就运行在另一个进程，这里就涉及到了从应用程序进程到管理者进程的通讯。this

这一个方向上通讯过程所涉及到的类包括： spa

### 2.1 应用程序进程向管理者进程发送消息

当咱们想要启动一个`Activity`，会首先调用到`Activity`的：代理

```
@Override
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        //...
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        //...
    }
```

以后调用到`Instrumentation`的：

```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
            //....
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //...
    }
```

这里咱们看到了上面`UML`图中`ActivityManagerNative`，它的`getDefault()`方法返回的是一个`IActivityManager`的实现类：

```
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            //1.获得一个代理对象
            IBinder b = ServiceManager.getService("activity");
            //2.将这个代理对象再通过一层封装
            IActivityManager am = asInterface(b);
            return am;
        }
    };

    static public IActivityManager getDefault() {
        return gDefault.get();
    }
```

这里，对于`gDefault`变量有两点说明：

- 这是一个`static`类型的变量，所以在程序中的任何地方调用`getDefault()`方法访问的是内存中的同一个对象
- 这里采用了`Singleton`模式，也就是懒汉模式的单例，只有第一次调用`get()`方法时，才会经过`create()`方法来建立一个对象

`create()`作了两件事：

- 经过`ServieManager`获得`IBinder`，这个`IBinder`是管理进程在应用程序进程的代理对象，经过`IBinder`的`transact`方法，咱们就能够向管理进程发送消息：

```
public boolean transact(int code, Parcel data, Parcel reply, int flags)
```

- 将`IBinder`传入`asInterface(IBinder b)`构建一个`IActivityManager`的实现类，能够看到，这里返回的是一个`ActivityManagerProxy`对象：

```
static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        //这一步先忽略....
        IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ActivityManagerProxy(obj);
    }
```

下面，咱们在来看一下这个`ActivityManagerProxy`，它实现了`IActivityManager`接口，咱们能够看到它所实现的`IActivityManager`接口方法都是经过构造这个对象时所传入的`IBinder.transact(xxxx)`来调用的，这些方法之间的区别就在于消息的类型以及参数。

```
class ActivityManagerProxy implements IActivityManager {

    public ActivityManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public IBinder asBinder() {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        //这个也很重要，咱们以后分析..
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        //发送消息到管理者进程...
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
 }
```

通过上面的分析，咱们用一句话总结：

> 应用程序进程经过`ActivityManagerProxy`内部的`IBinder.transact(...)`向管理者进程发送消息，这个`IBinder`是管理者进程在应用程序进程的一个代理对象，它是经过`ServieManager`得到的。

### 2.2 管理者进程处理消息

下面，咱们看一下管理者进程对于消息的处理，在管理者进程中，最终是经过`ActivityManagerService`对各个应用程序进行管理的。

它继承了`ActivityManagerNative`类，并重写了`Binder`类的`onTransact(...)`方法，咱们前面经过`ActivityManagerProxy`的`IBinder`对象发送的消息最终会调用到管理者进程中的这个函数当中，`ActivityManagerNative`对该方法进行了重写：

```
@Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
        case START_ACTIVITY_TRANSACTION:
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            String callingPackage = data.readString();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            ProfilerInfo profilerInfo = data.readInt() != 0
                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            //这里在管理者进程进行处理操做....
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
      //...
  }
```

在`onTransact(xxx)`方法中，会根据收到的消息类型，调用`IActivityManager`接口中所定义的不一样接口，而`ActivityManagerNative`是没有实现这些接口的，真正的处理在`ActivityManagerService`中，`ActivityManagerService`开始进行一系列复杂的操做，这里以后咱们介绍应用程序启动过程的时候再详细分析。

```
@Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```

一样的，咱们也用一句话总结：

> 管理者进程经过`onTransact(xxxx)`处理应用程序发送过来的消息

## 3、从管理者进程到应用程序进程

接着，咱们考虑另外一个方向上的通讯方式，从管理者进程到应用程序进程，这一方向上的通讯过程涉及到下面的类：

 `ApplicationThread`的整个框架和上面很相似，只不过在这一个方向上，管理者进程做为客户端，而应用程序进行则做为服务端。



### 3.1 管理者进程向应用程序进程发送消息

前面咱们分析的时候，应用程序进程向管理者进程发送消息的时候，是经过`IBinder`这个管理者进程在应用程序进程中的代理对象来实现的，而这个`IBinder`则是经过`ServiceManager`获取的：

```
IBinder b = ServiceManager.getService("activity");
```

同理，若是管理者进程但愿向应用程序进程发送消息，那么它也必须设法获得一个应用程序进程在它这边的代理对象。

咱们回忆一下，在第二节的分析中，应用者进程向管理者进程发送消息的同时，经过`writeStringBinder`，放入了下面这个对象：

```
public int startActivity(IApplicationThread caller, ...) {
    //...
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    //...
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
}
```

这个`caller`是在最开始调用`startActivityForResult`时传入的：

```
ActivityThread mMainThread;

    @Override
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        //...
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        //...
    }
```

经过查看`ActivityThread`的代码，咱们能够看到它实际上是一个定义在`ApplicationThread`中的`ApplicationThread`对象，它的`asBinder`实现是在`ApplicationThreadNative`当中：

```
public IBinder asBinder() {
       return this;
   }
```

在管理者进程接收消息的时候，就能够经过`readStrongBinder`得到这个`ApplicationThread`对象在管理者进程的代理对象`IBinder`：

```
@Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
      switch (code) {
        case START_ACTIVITY_TRANSACTION:
            //取出传入的ApplicationThread对象，以后调用asInterface方法..
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            //...
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            return true;
     }
  }
```

接着，它再经过`asInterface(IBinder xx)`方法把传入的代理对象经过`ApplicationThreadProxy`进行了一层封装：

```
static public IApplicationThread asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IApplicationThread in = (IApplicationThread) obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ApplicationThreadProxy(obj);
    }
```

以后，管理者进程就能够经过这个代理对象的`transact(xxxx)`方法向应用程序进程发送消息了：

```
class ApplicationThreadProxy implements IApplicationThread {

    private final IBinder mRemote;

    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }

    public final IBinder asBinder() {
        return mRemote;
    }

    public final void schedulePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) throws RemoteException {
        //....
        mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null, IBinder.FLAG_ONEWAY);
        //...
    }
```

这一个过程能够总结为：

> 管理者进程经过`ApplicationThreadProxy`内部的`IBinder`向应用程序进程发送消息，这个`IBinder`是应用程序进程在管理者进程的代理对象，它是在管理者进程接收应用程序进程发送过来的消息中得到的。

### 3.2 用户进程接收消息

在用户程序进程中，`ApplicationThread`的`onTransact(....)`就能够收到管理者进程发送的消息，以后再调用`ApplicationThread`所实现的`IApplicationThread`的接口方法进行消息的处理：

```
@Override
  public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case SCHEDULE_PAUSE_ACTIVITY_TRANSACTION: 
            data.enforceInterface(IApplicationThread.descriptor);
            IBinder b = data.readStrongBinder();
            boolean finished = data.readInt() != 0;
            boolean userLeaving = data.readInt() != 0;
            int configChanges = data.readInt();
            boolean dontReport = data.readInt() != 0;
            schedulePauseActivity(b, finished, userLeaving, configChanges, dontReport);
            return true;
  }
```

## 4、小结

以上就是应用程序进程和管理者进程之间的通讯方式，究其根本，都是经过获取对方进程的代理对象的`transact(xxxx)`方法发送消息，而对方进程则在`onTransact(xxxx)`方法中进行消息的处理，从而实现了进程之间的通讯

#  应用进程与 WMS 的通讯实现

## 1、前言

在 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现这篇文章中，咱们分析了应用进程和`AMS`之间的通讯实现，咱们今天讨论一下应用进程和`WindowManagerService`之间的通讯实现。java

在以前的分析中，咱们分两个部分来介绍了应用进程与`AMS`之间的通讯：bash

- 应用进程发送消息到`AMS`进程
- `AMS`发送消息到应用进程

如今，咱们也按照同样的讨论，分为这两个方向来介绍应用进程与`WMS`之间的通讯实现，整个通讯的过程会涉及到下面的这些类，其中加粗的线就是整个通讯实现的调用路径

## 2、WindowManagerImpl & WindowManagerGlobal

在`AMS`的讨论中，咱们以在应用进程中启动`Activity`为例子进行了介绍，今天，咱们选取另外一个你们很常见的例子：`Activity`启动以后，是如何将界面添加到屏幕上的。框架

### 2.1 WindowManagerImpl

`DecorView`的相关知识，它就对应于咱们须要添加到屏幕上的`View`的根节点，而这一添加的过程就须要涉及到和`WMS`之间的通讯，它是在`ActivityThread`的下面这个方法中实现的：ide

```
<!-- ActivityThread.java -->

final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        r = performResumeActivity(token, clearHide, reason);
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient && !a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
            } else if (!willBeVisible) {
                if (localLOGV) Slog.v(
                    TAG, "Launch " + r + " mStartedActivity set");
                r.hideForNow = true;
            }
        }
    }
```

上面的代码中，关键的是下面这几个步骤：函数

```
//(1) 经过 Activity 得到 Window 的实现类 PhoneWindow
r.window = r.activity.getWindow();

//(2) 经过 PhoneWindow 得到 DecorView
View decor = r.window.getDecorView();

//(3) 经过 Activity 得到 ViewManager 的实现类 WindowManagerImpl
ViewManager wm = a.getWindowManager();
WindowManager.LayoutParams l = r.window.getAttributes();

//(4) 经过 WindowManagerImpl 添加 DecorView
wm.addView(decor, l);
```

**(1) 经过 Activity 得到 Window 的实现类 PhoneWindow**oop

这里首先调用了`Activity`的`getWindow()`方法：ui

```
<!-- Activity.java -->

public Window getWindow() {
    return mWindow;
}
```

而这个`mWindow`是在`Activity.attach(xxxx)`中被赋值的，它实际上是`Window`的实现类`PhoneWindow`，`PhoneWindow`的构造函数中传入了`Activity`以及`parentWindow`：this

```
<!-- Activity.java -->

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        //....
        mWindow = new PhoneWindow(this, window);
}
```

第一步的分析就结束了，你们要记得一个结论：spa

> 经过`Activity`的`getWindow()`返回的是`PhoneWindow`对象，若是之后须要查看`mWindow`调用的函数，那么应当首先去`PhoneWindow.java`中查看是否有对应的实现，若是没有，那么再去`Window.java`中寻找。



**(2) 经过 PhoneWindow 得到 DecorView **



在第二步中，经过第一步返回的`PhoneWindow`得到`DecorView`，这个`mDecor`就是咱们所介绍的`DecorView`：

```
<!-- PhoneWindow.java -->

@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```

**(3) 经过 Activity 得到 ViewManager 的实现类 WindowManagerImpl**

下面，咱们看第三步，这里经过`Activity`的`getWindowManager()`返回了一个`ViewManager`的实现类：

```
<!-- Activity.java -->

public WindowManager getWindowManager() {
    return mWindowManager;
}
```

和`mWindow`相似，它也是在`Activity`的`attach`方法中赋值的：

```
<!-- Activity.java -->

    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
        //...
        mWindow = new PhoneWindow(this, window);
        //...
        mWindow.setWindowManager( 
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, 
                mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;
    }
```

它经过`Window`的`getWindowManager()`返回，咱们看一下`Window`的这个方法：

```
<!-- Window.java -->

public WindowManager getWindowManager() {
    return mWindowManager;
}
```

`PhoneWindow`和`Activity`相似，也有一个`mWindowManager`变量，咱们再去看一下它被赋值的地方：

```
<!-- Activity.java -->

public void setWindowManager(WindowManager wm, IBinder appToken, String appName, boolean hardwareAccelerated) {
        mAppToken = appToken;
        mAppName = appName;
        mHardwareAccelerated = hardwareAccelerated || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
        if (wm == null) {
            wm = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
        }
        mWindowManager = ((WindowManagerImpl) wm).createLocalWindowManager(this);
}
```

在`createLocalWindowManager`返回的是一个`WindowManagerImpl`对象：

```
<!-- WindowManagerImpl.java -->

public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
    return new WindowManagerImpl(mContext, parentWindow);
}
```

这样第三步的结论就是：

> 经过`Activity`的`getWindowManager()`方法返回的是它内部的`mWindowManager`对象，而这个对象是经过`Window`中的`mWindowManager`获得的，它实际上是`ViewManager`接口的实现类`WindowManagerImpl`。

**(4) 经过 WindowManagerImpl 添加 DecorView**

在第四步中，咱们经过`ViewManager`的`addView(View, WindowManager.LayoutParams)`方法添加`DecorView`，通过前面的分析，咱们知道它实际上是一个`WindowManagerImpl`对象，所以，咱们去看一下它所实现的`addView`方法：

```
public final class WindowManagerImpl implements WindowManager {

    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
}
```

能够看到`WindowManagerImpl`什么都没有作，它只是一个代理类，真正去作工做的是`mGlobal`，而且这个`mGlobal`使用了单例模式，也就是说，同一个进程中的全部`Activity`，调用的是同一个`WindowManagerGlobal`对象。

那么，咱们下面分析的重点就集中在了`WindowManagerGlobal`上了。

### 2.2 WindowManagerGlobal

咱们来看`WindowManagerGlobal`的`addView`方法，这里的第一个参数就是前面传递过来的`mDecor`：

```
<!-- WindowManagerGlobal -->

    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        //...
        ViewRootImpl root;
        View panelParentView = null;
        //....
        synchronized (mLock) {
            //...
            root = new ViewRootImpl(view.getContext(), display);
            mViews.add(view);
            mRoots.add(root);
        }
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            throw e;
        }
    }
```

在`addView(xxx)`方法中，会生成一个`ViewRootImpl`对象，并调用它的`setView(xxx)`方法把它和`DecorView`和它关联起来，与`WMS`通讯的逻辑都是由`ViewRootImpl`负责的，`WindowManagerGlobal`则负责用来管理应用进程当中的全部`ViewRootImpl`

在介绍 `ViewRootImpl`以前，咱们还要先讲一下在 `WindowManagerGlobal`中比较重要的两个静态变量：



```
<!-- WindowManagerGlobal.java -->

private static IWindowManager sWindowManagerService;
private static IWindowSession sWindowSession;
```

**(1) sWindowManagerService 为管理者进程在应用进程中的代理对象**

```
<!-- WindowManagerGlobal.java -->

sWindowManagerService = IWindowManager.Stub.asInterface(ServiceManager.getService("window"));
```

**(2) sWindowSession 为应用进程和管理者进程之间的会话**

```
<!-- WindowManagerGlobal.java -->

InputMethodManager imm = InputMethodManager.getInstance();
IWindowManager windowManager = getWindowManagerService();
sWindowSession = windowManager.openSession(
    new IWindowSessionCallback.Stub() {
        @Override
        public void onAnimatorScaleChanged(float scale) {
            ValueAnimator.setDurationScale(scale);
        }
    }, imm.getClient(), imm.getInputContext());
```

这个会话的方向为从应用进程到管理者进程，经过这个会话，应用进程就能够向管理者进程发送消息，而发送消息的逻辑则是经过`ViewRootImpl`来实现的，下面咱们就来看一下这个最重要的类是如何实现的。

## 3、ViewRootImpl

`ViewRootImpl`实现了`ViewParent`接口

在这个类中有三个最关键的变量：



```
<!-- ViewRootImpl.java -->

View mView;
IWindowSession mWindowSession;
IWindow W;
```

- `mView(View)`：这就是咱们在`WindowManagerGlobal`中经过`setView()`传递进来的，对应于`Activity`中的`DecorView`。

```
<!-- ViewRootImpl.java -->

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                //....
            }
        }
}
```

- `mWindowSession(IWindowSession)`：表明了从**应用进程到管理者进程**的会话，它其实就是`WindowManagerGlobal`中的`sWindowSession`，对于同一个进程，会复用同一个会话。

```
<!-- ViewRootImpl.java -->

public ViewRootImpl(Context context, Display display) {
    mWindowSession = WindowManagerGlobal.getWindowSession();
    //...
}
```

- `mWindow(W)`：表明了从**管理者进程到应用进程**的会话，是在`ViewRootImpl`中定义的一个内部类。

```
<!-- ViewRootImpl.java -->

    public ViewRootImpl(Context context, Display display) {
        mWindow = new W(this);
    }

    static class W extends IWindow.Stub {

        private final WeakReference<ViewRootImpl> mViewAncestor;
        private final IWindowSession mWindowSession;

        W(ViewRootImpl viewAncestor) {
            mViewAncestor = new WeakReference<ViewRootImpl>(viewAncestor);
            mWindowSession = viewAncestor.mWindowSession;
        }
    }
```

### 3.1 从应用进程到管理者进程

`IWindowSession`是应用进程到管理者进程的会话，它定义了管理者进程所支持的调用接口，经过`IWindowSession`内部的管理者进程的远程代理对象，咱们就能够实现从应用进程向管理者进程发送消息。

而在管理者进程中，经过`WindowManagerService`来处理来自各个应用进程的消息，在`WMS`中有一个`Session`列表，全部从应用进程到管理进程的会话都保存在该列表中。

```
<!-- WindowManagerService.java -->

final ArraySet<Session> mSessions = new ArraySet<>();
```

`Session`则实现了`IWindowSession.Stub`接口

当它收到消息以后，就会回调 `IWindowSession`的接口方法



咱们举一个例子，在`ViewRootImpl`的`setView(xxx)`方法中，调用了`IWindowSession`的下面这个接口方法：

```
<!-- ViewRootImpl.java -->

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        //....
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(),
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mInputChannel);
    }
```

最终这一跨进程的调用会回调到该应用进程在管理者进程中对应的`Session`对象的回调方法中：

```
<!-- Session.java -->

    @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```

### 3.2 从管理者进程到应用进程

若是咱们但愿实现从管理者进程发送消息到应用进程，那么也须要一个应用进程在管理者进程的代理对象。

在调用`addToDisplay`时，咱们传入的第一个参数是`mWindow`，前面咱们介绍过，它实现了`IWindow.Stub`接口

这样管理者进程在 `Session`的 `addToDisplay`方法被回调时，就能够得到一个远程代理对象，它就能够经过 `IWindow`中定义的接口方法，实现从管理者进程到应用进程的通讯。



在`Session`的`addToDisplay()`方法中，会调用`WMS`的`addWindow`方法，而在`addWindow`方法中，它会建立一个`WindowState`对象，一个进程中的每一个`ViewRootImpl`会对应于一个`IWindow`会话，它们被保存在`WMS`的下面这个`HashMap`中：

```
<!-- WindowManagerService.java -->

final HashMap<IBinder, WindowState> mWindowMap = new HashMap<>();
```

其中`key`值就表示应用进程在管理者进程中的远程代理对象，例如咱们在`WMS`中调用了下面这个方法：

```
<!-- WindowState.java -->

mClient.windowFocusChanged(focused, inTouchMode);
```

那么应用进程中`IWindow.Stub`的实现的`ViewRootImpl.W`类的对应方法就会被回调，在该回调方法中又会调用`ViewRootImpl`的方法：

```
<!-- ViewRootImpl.java -->

        @Override
        public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
            final ViewRootImpl viewAncestor = mViewAncestor.get();
            if (viewAncestor != null) {
                viewAncestor.windowFocusChanged(hasFocus, inTouchMode);
            }
        }
```

而在`ViewRootImpl`的`windowFocusChanged`方法中，会经过它内部的一个`ViewRootHandler`发送消息，`ViewRootHandler`的`Looper`是和应用进程中的主线程所绑定的，所以它就能够在`handleMessage`进行后续逻辑处理。

```
<!-- ViewRootImpl.java -->

    final ViewRootHandler mHandler = new ViewRootHandler();

    public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
        Message msg = Message.obtain();
        msg.what = MSG_WINDOW_FOCUS_CHANGED;
        msg.arg1 = hasFocus ? 1 : 0;
        msg.arg2 = inTouchMode ? 1 : 0;
        mHandler.sendMessage(msg);
    }
```

## 4、小结

作一个简单的总结，应用进程与`WMS`之间通讯是经过`WindowManagerGlobal`中`ViewRootImpl`来管理的，`ViewRootImpl`中的`IWindowSession`对应于从应用进程到`WMS`的通讯，而`IWindow`对应于从管理者进程到应用进程的通讯。

# 应用进程之间的通讯实现

## 1、前言

在 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现和 Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通讯实现 这两篇文章中，咱们介绍了应用进程与`AMS`以及`WMS`之间的通讯实现，可是逻辑仍是比较绕的，为了方便你们更好地理解，咱们介绍一下你们见得比较多的应用进程间通讯的实现。java

## 2、例子

提及应用进程之间的通讯，相信你们都不陌生，应用进程之间通讯最经常使用的方式就是`AIDL`，下面，咱们先演示一个`AIDL`的简单例子，接下来，咱们再分析它的内部实现。android

### 2.1 服务端

#### 2.1.1 编写 AIDL 文件

第一件事，就是服务端须要声明本身能够为客户端提供什么功能，而这一声明则须要经过一个`.aidl`文件来实现，咱们先简要介绍介绍一下`AIDL`文件：bash

- `AIDL`文件的后缀为`*.aidl`
- 对于`AIDL`默认支持的数据类型，是不须要导入包的，这些默认的数据类型包括：
- 基本数据类型：`byte/short/int/long/float/double/boolean/char`
- `String/CharSequence`
- `List<T>`：其中`T`必须是`AIDL`支持的类型，或者是其它`AIDL`生成的接口，或者是实现了`Parcelable`接口的对象，`List`支持泛型
- `Map`：它的要求和`List`相似，可是不支持泛型
- `Tag`标签：对于接口方法中的形参，咱们须要用`in/out/inout`三种关键词去修饰：
- `in`：表示服务端会收到客户端的完整对象，可是在服务端对这个对象的修改不会同步给客户端
- `out`：表示服务端会收到客户端的空对象，它对这个对象的修改会同步到客户端
- `inout`：表示服务端会收到客户端的完整对象，它对这个对象的修改会同步到客户端

说了这么多，咱们最多见的需求无非就是两个：让复杂对象实现`Parcelable`接口实现传输以及定义接口方法。app

**(1) Parcelable 的实现**

咱们能够很方便地经过`AS`插件来让某个对象实现`Parcelable`接口，并自动补全要实现的方法： 函数

安装完以后重启，咱们本来的对象为：

```
public class InObject {

    private int inData;

    public int getInData() {
        return inData;
    }

    public void setInData(int inData) {
        this.inData = inData;
    }
    
}
```

在文件的空白处，点击右键`Generate -> Parcelable`

它就会帮咱们实现 `Parcelable`中的接口：



```
public class InObject implements Parcelable {

    private int inData;

    public int getInData() {
        return inData;
    }

    public void setInData(int inData) {
        this.inData = inData;
    }

    public InObject() {}

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.inData);
    }

    protected InObject(Parcel in) {
        this.inData = in.readInt();
    }

    public static final Parcelable.Creator<InObject> CREATOR = new Parcelable.Creator<InObject>() {
        @Override
        public InObject createFromParcel(Parcel source) {
            return new InObject(source);
        }

        @Override
        public InObject[] newArray(int size) {
            return new InObject[size];
        }
    };
}
```

**(2) 编写 AIDL 文件**this

点击`File -> New -> AIDL -> AIDL File`以后，会多出一个名叫`aidl`的文件夹，以后咱们全部须要用到的`aidl`文件都被存放在这里

初始时候， `AS`为咱们生成的 `AIDL`文件为：



```
// AIDLInterface.aidl
package com.demo.lizejun.binderdemoclient;

// Declare any non-default types here with import statements

interface AIDLInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

当咱们编译以后，就会在下面这个路径中生成一个`Java`文件：

```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /home/lizejun/Repository/RepoGithub/BinderDemo/app/src/main/aidl/com/demo/lizejun/binderdemoclient/AIDLInterface.aidl
 */
package com.demo.lizejun.binderdemoclient;
// Declare any non-default types here with import statements

public interface AIDLInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.demo.lizejun.binderdemoclient.AIDLInterface {
        private static final java.lang.String DESCRIPTOR = "com.demo.lizejun.binderdemoclient.AIDLInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.demo.lizejun.binderdemoclient.AIDLInterface interface,
         * generating a proxy if needed.
         */
        public static com.demo.lizejun.binderdemoclient.AIDLInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.demo.lizejun.binderdemoclient.AIDLInterface))) {
                return ((com.demo.lizejun.binderdemoclient.AIDLInterface) iin);
            }
            return new com.demo.lizejun.binderdemoclient.AIDLInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0 != data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.demo.lizejun.binderdemoclient.AIDLInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean) ? (1) : (0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
}
```

有没有感受在 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现 和 Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通讯实现中也见到过相似的东西 `asInterface/asBinder/transact/onTransact/Stub/Proxy...`，这个咱们以后再来解释，下面咱们介绍服务端的第二步操作。

#### 2.1.2 编写 Service

既然服务端已经定义好了接口，那么接下来就服务端就须要实现这些接口：

```
public class AIDLService extends Service {

    private final AIDLInterface.Stub mBinder = new AIDLInterface.Stub() {
        @Override
        public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
            Log.d("basicTypes", "basicTypes");
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

在这个`Service`中，咱们只须要实现`AIDLInterface.Stub`接口中的`basicTypes`接口就能够了，它就是咱们在`AIDL`文件中定义的接口，再把这个对象经过`onBinde`方法返回。

#### 2.1.3 声明 Service

最后一步，在`AndroidManifest.xml`文件中声明这个`Service`：

```
<service
            android:name=".server.AIDLService"
            android:enabled="true"
            android:exported="true">
        </service>
```

### 2.2 客户端

#### 2.2.1 编写 AIDL 文件

和服务端相似，客户端也须要和服务端同样，生成一个相同的`AIDL`文件，这里就很少说了：

```
// AIDLInterface.aidl
package com.demo.lizejun.binderdemoclient;

// Declare any non-default types here with import statements

interface AIDLInterface {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

和服务端相似，咱们也会获得一个由`aidl`生成的`Java`接口文件。

#### 2.2.2 绑定服务，并调用

**(1) 绑定服务**

```
private void bind() {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName("com.demo.lizejun.binderdemo", "com.demo.lizejun.binderdemo.server.AIDLService"));
        boolean result = bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
        Log.d("bind", "result=" + result);
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinder = AIDLInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mBinder = null;
        }
    };
```

**(2) 调用接口**

```
public void sayHello(View view) {
        try {
            if (mBinder != null) {
                mBinder.basicTypes(0, 0, false, 0, 0, "aaa");
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
```

## 3、实现原理

下面，咱们就一块儿来分析一下经过`AIDL`实现的进程间通讯的原理。

**(1) 客户端得到服务端的远程代理对象 IBinder **

咱们从客户端提及，当咱们在客户端调用了`bindService`方法以后，就会启动服务端实现的`AIDLService`，而在该`AIDLService`的`onBind()`方法中返回了`AIDLInterface.Stub`的实现类，绑定成功以后，客户端就经过`onServiceConnected`的回调，获得了它在客户端进程的远程代理对象`IBinder`。

**(2) AIDLInterface.Stub.asInterface(IBinder)**

在拿到这个`IBinder`对象以后，咱们经过`AIDLInterface.Stub.asInterface(IBinder)`方法，对这个`IBinder`进行了一层包装，转换成为`AIDLInterface`接口类，那么这个`AIDLInterface`是怎么来的呢，它就是经过咱们在客户端定义的`aidl`文件在编译时生成的，能够看到，最终`asInterface`方法会返回给咱们一个`AIDLInterface`的实现类`Proxy`，`IBinder`则被保存为它内部的一个成员变量`mRemote`。

也就是说，咱们在客户端中保存的 `mBinder`其实是一个由 `AIDL`文件所生成的 `Proxy`对象。



**(3) 调用 AIDLInterface 的接口方法**

`Proxy`实现了`AIDLInterface`中定义的接口方法，当咱们调用它的接口方法时

其实是经过它内部的 `mRemote`对象的 `transact`方法发送消息

**(4) 服务端接收消息**

那么客户端发送的这一消息去哪里了呢，回想一下在服务端中经过`onBind()`放回的`IBinder`，它实际上是由`AIDL`文件生成的`AIDLInterface.java`中的内部类`Stub`的实现，而在`Stub`中有一个回调函数`onTransact`，当咱们经过客户端发送消息以后，那么服务端的`Stub`对象就会经过`onTransact`收到这一发送的消息

**(5) 调用子类的处理逻辑**

在`onTransact`方法中，又会去调用`AIDLInterface`定义的接口

而咱们在服务端`AIDLService`中实现了该接口，所以，最终就打印出了咱们在上面所看到的文字

## 4、进一步讨论

如今，咱们经过这一进程间的通讯过程来复习一下前面两篇文章中讨论的应用进程与`AMS`和`WMS`之间的通讯实现。

在 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现中，`AMS`所在进程在收到消息以后，进行了下面的处理逻辑

这里面的处理逻辑就至关于咱们在客户端中 `onServiceConnected`中作的工做同样， `IApplicationThread`其实是一个 `ApplicaionThreadProxy`对象，和咱们上面 `AIDLInterface`其实是一个 `AIDLInterface.Stub.Proxy`的原理是同样的。当咱们调用了它的接口方法时，就是经过内部的 `mRemote`发送了消息。



在`AMS`通讯中，接收消息的是应用进程中的`ApplicationThread`，而咱们上面例子中接收消息的是服务端进程的`AIDLInterface.Stub`的实现类`mBinder`，一样是在`onTransact()`方法中接收消息，再由子类去实现处理的逻辑。

Framework 源码解析知识梳理(2) - 应用进程与 WMS 的通讯实现 中的原理就更好解释了，由于它就是经过`AIDL`来实现的，咱们从客户端向`WMS`所在进程创建会话时，是经过`IWindowSession`来实现的，会话过程当中`IWindowSession`就对应于`AIDLInterface`，而`WMS`中的`IWindowSession.Stub`的实现类`Session`，就对应于上面咱们在`AIDLService`中定义的`AIDLInterface.Stub`的实现类`mBinder`。

## 5、小结

其实`AIDL`并无什么神秘的东西，它的本质就是`Binder`通讯，咱们定义`aidl`文件的目的，主要有两个：

- 为了让应用进程间通讯的实现者不用再去编写`transact/onTransact`里面的代码，由于这些东西和业务逻辑是无关的，只不过是简单的发送消息、接收消息。
- 让服务端、客户端只关心接口的定义。

若是咱们明白了`AIDL`的原理，那么咱们彻底能够不用定义`AIDL`文件，本身去参考由`AIDL`文件所生成的`Java`文件的逻辑，进行消息的发送和处理。

# 从源码角度谈谈 Handler 的应用

## 1、前言

其实，这篇文章原本应该叫作"`Handler`源码解析"的，可是想一想写`Handler`源码的文章太多了，仍是缓一缓，先写些不同的，今天，咱们从源码的角度来总结一下应用`Handler`的一些场景的原理。面试

## 2、Handler 的应用

### 2.1 线程间的通讯

在平时的开发当中，`Handler`最多见的用法就是用于线程之间的通讯，特别是当咱们在子线程中去处理耗时的任务，当任务完成以后，咱们但愿将结果发送到主线程中进行处理，那么就会使用到`Handler`

当咱们在子线程中经过 `Handler`放入在主线程中的循环队列时，那么主线程就会收到消息，以后，咱们就能够在主线程中进行后续的处理，例以下面这样：

```
public class MainActivity extends AppCompatActivity {

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            Log.d("Handler", "HandleMessageThreadId=" + Thread.currentThread().getId());
        }
    };

    private void startThread() {
        new Thread() {
            @Override
            public void run() {
                super.run();
                Log.d("Handler", "ThreadId=" + Thread.currentThread().getId());
                mHandler.sendEmptyMessage(0);
            }
        }.start();
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.d("Handler", "MainThreadId=" + Thread.currentThread().getId());
        startThread();
    }
    
}
```

打印出上面的线程`Id`，能够看到最后`handleMessage`收到消息的时候是在主线程当中

### 2.2 实现延时操做

除了线程间的通讯以外，咱们还能够经过`Handler`提供的`sendXXXDelay`方法，实现延时操做，延时操做有两种目的：app

- 一种是但愿让不那么紧急的任务延后执行，例如在应用启动过程当中，咱们在`onCreate`方法中的任务不是那个紧急，那么能够经过`Handler`发送一个延时消息出去，让它不占用主线程去渲染布局的资源，从而提升应用的启动速度。
- 另外一种就是防抖动操做，咱们收到一个命令，并非立刻执行它，而是经过`Handler`的`sendxxxDelay`方法延时，若是在这段事件内又有一个相同的命令来到了，那么就把以前的消息移除，再放入一个新的延时消息。 例以下面的例子，当咱们点击一个按钮以后，咱们不马上执行任务，而是一段时间以后仍然没有收到第二次点击事件采起执行任务：

```
private Handler mDelayHandler = new Handler() {

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            Log.d("Handler", "handleMessage");
        }

    };

    private void performClickDelay() {
        Log.d("Handler", "performClickDelay");
        mDelayHandler.removeMessages(1);
        mDelayHandler.sendEmptyMessageDelayed(1, 500);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTv = (TextView) findViewById(R.id.tv);
        mTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                performClickDelay();
            }
        });
    }
```

咱们连续屡次点击按钮以后，只收到了一次消息

### 2.3 使用`HandlerThread`在异步线程执行耗时操做

因为`Looper`实际上是线程的一个私有变量（`ThreadLocal`），主线程能够有`Looper`，子线程一样也能够有`Looper`，只不过从子线程中的`Looper`中的`MessageQueue`中取出消息以后，是在子线程当中处理的，那么咱们就能够经过它来执行异步操作

`HandlerThread`就是基于这一思想来实现的，关于 `HandlerThread`的内部实现，它的本质就是当异步线程启动以后，会初始化这个线程私有的 `Looper`，所以，当咱们经过 `HandlerThread`中的 `getLooper()`方法得到这个 `Looper`以后，在经过这个 `Looper`来建立一个 `Handler`，那么这个 `Handler`的 `handleMessage`回调时所在的线程就是这个异步线程。



下面是一个简单的事例：

```
private void useHandlerThread() {
        final HandlerThread handlerThread = new HandlerThread("handlerThread");
        System.out.println("MainThreadId=" + Thread.currentThread().getId() + ",handlerThreadId=" + handlerThread.getId());
        handlerThread.start();
        MyHandler handler = new MyHandler(handlerThread.getLooper());
        handler.sendEmptyMessage(0);
    }

    private class MyHandler extends Handler {

        MyHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            System.out.println("handleMessageThreadId=" + Thread.currentThread().getId());
        }
    }
```

咱们打印出`handleMessage`的执行时的线程`ID`，能够看到它就是`HandlerThread`的线程`ID`，所以咱们就能够经过发送消息的方式，在异步线程执行一些耗时的操作

### 2.4 使用`AsyncQueryHandler`对`ContentProvider`进行操做

若是你的本地数据使用`ContentProvider`封装的，那么对这些数据的增删改查操做，为了避免影响主线程，应该在子线程中进行操做，而`AsyncQueryHandler`就是系统为咱们提供的很方便的工具类，它经过`Handler + HandlerThread`的方式，实现了异步的增删改查。函数

它是在`HandlerThread`的基础之上扩展出来的，其基本思想以下图所示，这一框架就保证了发起命令和接收回调是在同一个线程，而任务的执行则是在另外一个线程

固然`AsyncQueryHandler`限制了只能操做经过`ContentProvider`封装的数据，咱们能够参考它的思想，进行扩展，实现对于数据库的增删改查

### 2.5 使用`Handler`机制检测应用中的卡顿问题

对于每一个应用程序来讲，它的入口函数为`ActivityThread`中的`main()`方法：

```
public static void main(String[] args) {
        //...
        Looper.prepareMainLooper();
        //...
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

而在`main()`函数的最后，则构建一个在主线程中的循环队列，以后应用程序收到的事件以后，还主线程就会被唤醒，进行事件的处理，假如这一处理的事件过长，那么就会发生`ANR`，所以，咱们就能够经过计算主线程中对于单次消息的处理时间，从而间隔地判断是否存在卡顿问题。 那么要怎么知道主线程中单次消息的处理时间呢，咱们查看`Looper`的源码：

```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted. final long newIdent = Binder.clearCallingIdentity(); if (ident != newIdent) { Log.wtf(TAG, "Thread identity changed from 0x" + Long.toHexString(ident) + " to 0x" + Long.toHexString(newIdent) + " while dispatching to " + msg.target.getClass().getName() + " " + msg.callback + " what=" + msg.what); } msg.recycleUnchecked(); } }
```

对于消息的处理对应这句：

```
msg.target.dispatchMessage(msg);
```

而在这句话的先后，则调用了`Printer`类打印，那么咱们就能够经过这两次打印的之间的时长来得到单次消息的处理时间

```
public class MainLoopMonitor {

    private static final String MSG_START = ">>>>> Dispatching";
    private static final String MSG_END = "<<<<< Finished";
    private static final int TIME = 1000;
    private Handler mExecuteHandler;
    private Runnable mExecuteRunnable;

    private static class Holder {
        private static final MainLoopMonitor INSTANCE = new MainLoopMonitor();
    }

    public static MainLoopMonitor getInstance() {
        return Holder.INSTANCE;
    }

    private MainLoopMonitor() {
        HandlerThread monitorThread = new HandlerThread("LooperMonitor");
        monitorThread.start();
        mExecuteHandler = new Handler(monitorThread.getLooper());
        mExecuteRunnable = new ExecutorRunnable();
    }

    public void startMonitor() {
        Looper.getMainLooper().setMessageLogging(new Printer() {
            @Override
            public void println(String x) {
                if (x.startsWith(MSG_START)) {
                    mExecuteHandler.postDelayed(mExecuteRunnable, TIME);
                } else if (x.startsWith(MSG_END)) {
                    mExecuteHandler.removeCallbacks(mExecuteRunnable);
                }
            }
        });
    }
    
    private class ExecutorRunnable implements Runnable {

        @Override
        public void run() {
            StringBuilder sb = new StringBuilder();
            StackTraceElement[] stackTrace = Looper.getMainLooper().getThread().getStackTrace();
            for (StackTraceElement s : stackTrace) {
                sb.append(s).append("\n");
            }
            System.out.println("MainLooperMonitor:" + sb.toString());
        }
    }
}
```

假如咱们像下面这样，在主线程中进行了耗时的操作：

```
public void anrButton(View view) {
        for (int i = 0; i < (1 << 30); i++) {
            int b = 6;
        }
}
```

## 3、Handler 使用注意事项

在介绍完`Handler`的这些用法，咱们再来总结一下使用`Handler`是比较容易犯的一些错误。

### 3.1 内存泄漏

一个比较常见的错误就是将定义`Handler`的子类时将它做为`Activity`的内部类，而因为内部类会默认持有外部类的引用，所以，若是这个内部类的实例在`Activity`试图被回收的时候，没有被销毁掉，那么就会致使`Activity`没法被回收，从而引发内存泄漏。

而`Handler`实例没法被销毁掉最多见的状况就是，咱们经过它发送了一个延时消息出去，此时这个消息会被放入到该线程所对应的`Looper`中的`MessageQueue`当中，而该消息为了能在获得执行以后，回调到对应的`Handler`，所以它会持有这个`Handler`的实例。

### 3.2 在子线程中实例化 new Handler()

在子线程中`new Handler()`有下面两个须要注意的点：

**(1) 在 new Handler() 以前调用 Looper.prepare() 方法**

另一个错误就是咱们在子线程当中，直接调用`new Handler()`，那么这时候会抛出异常，例以下面这样：

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                mHandler = new MyHandler();
            }

        }.start();
    }
```

`Handler`的构造函数中，会去检查当前线程的 `Looper`是否已经被初始化：



```
public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

若是没有，那么就会抛出上面代码当中的异常，正确的作法，是须要保证咱们在`new Handler`以前，要保证当前线程的`Looper`对象已经被初始化，也就是`Looper`当中的下面这个方法被调用：

```
/** Initialize the current thread as a looper.
      * This gives you a chance to create handlers that then reference
      * this looper, before actually starting the loop. Be sure to call
      * {@link #loop()} after calling this method, and end it by calling
      * {@link #quit()}.
      */
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

**(2) 若是但愿经过该 Looper 接收消息那么要调用 Looper.loop() 方法，而且须要放在最后一句调用**

假如咱们只调用了`prepare()`方法，仅仅只是初始化了一个线程的私有的变量，此时是没法经过这个`Looper`构建的`Handler`来接收消息的，例以下面这样：

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                Looper.prepare();
                mHandler = new MyHandler();
            }
            
        }.start();
    }

    public void anrButton(View view) {
        mHandler.sendEmptyMessage(0);
    }

    private static class MyHandler extends Handler {

        @Override
        public void handleMessage(Message msg) {
            Log.e("TAG", "handleMessageId=" + Thread.currentThread().getId());
        }
    }
```

此时咱们应当调用`loop`方法让这个循环队列开始工做，而且该调用必定要位于最后一句，由于一旦调用了`loop`方法，那么它就会进入等待 -> 收到消息 -> 唤醒 -> 处理消息 -> 等待的循环过程，而这一过程只有等到`loop`方法返回的时候才会结束，所以，咱们初始化`Handler`的语句要放在`loop`方法以前，相似于下面这样：

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        new Thread() {

            @Override
            public void run() {
                Looper.prepare();
                mHandler = new MyHandler();
                //最后一句再调用
                Looper.loop();
            }

        }.start();
    }
```

## 4、小结

`Handler`的机制彷佛已经成为面试必问的题目，若是你们能在回答完内部的实现原理，再根据这些实现原理引伸出上面的几个应用和注意事项，能够加分很多哦~

# startService 源码分析

## 1、前言

最近在看关于插件化的知识，遇到了如何实现`Service`插件化的问题，所以，先学习一下`Service`内部的实现原理，这里面会涉及到应用进程和`ActivityManagerService`的通讯，建议你们先阅读一下以前的这篇文章 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现，整个`Service`的启动过程离不开和`AMS`的通讯，从整个宏观上来看，它的模型以下，是否是和咱们以前分析的通讯过程如出一辙

## 2、源码分析

当咱们在`Activity/Service/Application`中，经过下面这个方法启动`Service`时：

```
public ComponentName startService(Intent service)
```

与以前分析过的不少源码相似，最终会调用到`ContextImpl`的同名方法当中，而该方法又会调用内部的`startServiceCommon(Intent, UserHandle)`：

在该方法中，主要作了两件事：



- 校验`Intent`的合法性，对于`Android 5.0`之后，咱们要求`Intent`中须要制定`Package`和`Component`中至少一个，不然会抛出异常。
- 经过`Binder`通讯调用到`ActivityManagerService`的对应方法，关于调用的详细过程能够参考 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现中对于应用进程到`ActivityManagerService`的通讯部分分析。

`ActivityManagerService`中的实现为： 数据结构

这里面，经过 `mServices`变量来去实现 `Service`的启动，而 `mServices`的类型为 `ActiveServices`，它是 `ActivityManagerService`中负责处理 `Service`相关业务的类，不管咱们是经过 `start`仍是 `bind`方式启动的 `Service`，都是经过它来实现的，它也被称为“应用服务”的管理类。



从经过`ActiveServices`调用`startServiceLocked`，到`Service`真正被启动的过程，下面，咱们就来一一分析每个阶段究竟作了什么：

咱们首先来看入口函数`startServiceLocked`的内部实现：

```
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        final boolean callerFg;
        //caller是调用者进程在AMS的远程代理对象，类型为ApplicationThreadProxy。
        if (caller != null) {
            //得到调用者的进程信息。
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + Binder.getCallingPid()
                        + ") when starting service " + service);
            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }

        //经过传递的信息，解析出要启动的Service，封装在ServiceLookupResult中。
        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false);
        if (res == null) {
            return null;
        }
        if (res.record == null) {
            return new ComponentName("!", res.permission != null
                    ? res.permission : "private to package");
        }
        //对于每一个被启动的Service，它们在AMS端的数据结构为ServiceRecord。
        ServiceRecord r = res.record;

        if (!mAm.mUserController.exists(r.userId)) {
            Slog.w(TAG, "Trying to start service with non-existent user! " + r.userId);
            return null;
        }
        //判断是否容许启动该服务。
        if (!r.startRequested) {
            final long token = Binder.clearCallingIdentity();
            try {
                final int allowed = mAm.checkAllowBackgroundLocked(
                        r.appInfo.uid, r.packageName, callingPid, true);
                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                    Slog.w(TAG, "Background start not allowed: service "
                            + service + " to " + r.name.flattenToShortString()
                            + " from pid=" + callingPid + " uid=" + callingUid
                            + " pkg=" + callingPackage);
                    return null;
                }
            } finally {
                Binder.restoreCallingIdentity(token);
            }
        }
        //是否须要配置相应的权限。
        NeededUriGrants neededGrants = mAm.checkGrantUriPermissionFromIntentLocked(
                callingUid, r.packageName, service, service.getFlags(), null, r.userId);

        if (Build.PERMISSIONS_REVIEW_REQUIRED) {
            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage,
                    callingUid, service, callerFg, userId)) {
                return null;
            }
        }
        //若是该Service正在等待被从新启动，那么移除它。
        if (unscheduleServiceRestartLocked(r, callingUid, false)) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
        }
        //给ServiceRecord添加必要的信息。
        r.lastActivity = SystemClock.uptimeMillis();
        r.startRequested = true;
        r.delayedStop = false;
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants));
        //其它的一些逻辑，与第一次启动无关，就先不分析了。
        return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
    }
```

简单地来讲，就是根据`Intent`查找须要启动的`Service`，封装成`ServiceRecord`对象，初始化其中的关键变量，在这一过程当中，加入了一些必要的逻辑判断，最终调用了`startServiceInnerLocked`方法

```
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
        //...
        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        //...
        return r.name;
    }
```

这里面的逻辑咱们先忽略掉一部分，只须要知道在正常状况下它会去调用`bringUpServiceLocked`来启动`Service`

```
private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting) throws TransactionTooLargeException {
 
        //若是该Service已经启动。
        if (r.app != null && r.app.thread != null) {
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }
        //若是正在等待被从新启动，那么什么也不作。
        if (!whileRestarting && r.restartDelay > 0) {
            return null;
        }

        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent);

        //清除等待被从新启动的状态。
        if (mRestartingServices.remove(r)) {
            r.resetRestartCounter();
            clearRestartingIfNeededLocked(r);
        }

        //由于咱们立刻就要启动该Service，所以去掉它的延时属性。
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        //若是该Service所属的用户没有启动，那么调用 bringDownServiceLocked 方法。
        if (mAm.mStartedUsers.get(r.userId) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        ProcessRecord app;
        //若是不是运行在独立的进程。
        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            //若是该进程已经启动，那么调用realStartServiceLocked方法。
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                    //在Service所属进程已经启动的状况下调用的方法。
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortName, e);
                }
            }
        } else {
            app = r.isolatedProc;
        }

        //若是该Service所对应的进程没有启动，那么首先启动该进程。
        if (app == null) {
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    "service", r.name, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }
        //将该ServiceRecord加入到等待的集合当中，等到新的进程启动以后，再去启动它。
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
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

`bringUpServiceLocked`的逻辑就比较复杂了，它会根据目标`Service`及其所属进程的状态，走向不一样的分支：

- 进程已经存在，而且目标`Service`已经启动：`sendServiceArgsLocked`
- 进程已经存在，可是目标`Service`没有启动：`realStartServiceLocked`
- 进程不存在：`startProcessLocked`，而且将`ServiceRecord`加入到`mPendingServices`中，等待进程启动以后再去启动该`Service`。

下面，咱们就来一块儿分析一下这三种状况

### 2.1 进程已经存在，而且目标 Service 已经启动

首先来当进程已经存在，且目标`Service`已经启动时所调用的`sendServiceArgsLocked`方法：

```
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        final int N = r.pendingStarts.size();
        if (N == 0) {
            return;
        }

        while (r.pendingStarts.size() > 0) {
            Exception caughtException = null;
            ServiceRecord.StartItem si;
            try {
                si = r.pendingStarts.remove(0);
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Sending arguments to: "
                        + r + " " + r.intent + " args=" + si.intent);
                if (si.intent == null && N > 1) {
                    // If somehow we got a dummy null intent in the middle,
                    // then skip it.  DO NOT skip a null intent when it is
                    // the only one in the list -- this is to support the
                    // onStartCommand(null) case.
                    continue;
                }
                si.deliveredTime = SystemClock.uptimeMillis();
                r.deliveredStarts.add(si);
                si.deliveryCount++;
                if (si.neededGrants != null) {
                    mAm.grantUriPermissionUncheckedFromIntentLocked(si.neededGrants,
                            si.getUriPermissionsLocked());
                }
                bumpServiceExecutingLocked(r, execInFg, "start");
                if (!oomAdjusted) {
                    oomAdjusted = true;
                    mAm.updateOomAdjLocked(r.app);
                }
                int flags = 0;
                if (si.deliveryCount > 1) {
                    flags |= Service.START_FLAG_RETRY;
                }
                if (si.doneExecutingCount > 0) {
                    flags |= Service.START_FLAG_REDELIVERY;
                }
                r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
            } catch (TransactionTooLargeException e) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large: intent="
                        + si.intent);
                caughtException = e;
            } catch (RemoteException e) {
                // Remote process gone...  we'll let the normal cleanup take care of this. if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r); caughtException = e; } catch (Exception e) { Slog.w(TAG, "Unexpected exception", e); caughtException = e; } if (caughtException != null) { // Keep nesting count correct final boolean inDestroying = mDestroyingServices.contains(r); serviceDoneExecutingLocked(r, inDestroying, inDestroying); if (caughtException instanceof TransactionTooLargeException) { throw (TransactionTooLargeException)caughtException; } break; } } }
```

这里面最关键的就是调用了`r.app.thread`的`scheduleServiceArgs`方法，它其实就是`Service`所属进程的`ApplicationThread`对象在`AMS`端的一个远程代理对象，也就是`ApplicationThreadProxy`，经过`binder`通讯，它最终会回调到`Service`所属进程的`ApplicationThread`的`scheduleServiceArgs`中：

```
private class ApplicationThread extends ApplicationThreadNative {

        public final void scheduleServiceArgs(IBinder token, boolean taskRemoved, int startId,
            int flags ,Intent args) {
            ServiceArgsData s = new ServiceArgsData();
            s.token = token;
            s.taskRemoved = taskRemoved;
            s.startId = startId;
            s.flags = flags;
            s.args = args;
            sendMessage(H.SERVICE_ARGS, s);
        }
   }
```

`sendMessage`函数会经过`ActivityThread`内部的`mH`对象发送消息到主线程当中，`mH`其实是一个`Handler`的子类，在它的`handleMessage`回调中，处理`H.SERVICE_ARGS`这条消息

在 `handleServiceArgs`方法，就会从 `mServices`去查找对应的 `Service`，调用咱们熟悉的 `onStartCommand`方法，至于 `Service`对象是如何被加入到 `mServices`中的，咱们在 `2.2`节中进行分析

```
private void handleServiceArgs(ServiceArgsData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
                if (data.args != null) {
                    data.args.setExtrasClassLoader(s.getClassLoader());
                    data.args.prepareToEnterProcess();
                }
                int res;
                if (!data.taskRemoved) {
                    //调用 onStartCommand 方法。
                    res = s.onStartCommand(data.args, data.flags, data.startId);
                } else {
                    s.onTaskRemoved(data.args);
                    res = Service.START_TASK_REMOVED_COMPLETE;
                }

                QueuedWork.waitToFinish();

                try {
                    //通知AMS启动完成。
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                } catch (RemoteException e) {
                    // nothing to do.
                }
                ensureJitEnabled();
            } catch (Exception e) {
                if (!mInstrumentation.onException(s, e)) {
                    throw new RuntimeException(
                            "Unable to start service " + s
                            + " with " + data.args + ": " + e.toString(), e);
                }
            }
        }
    }
```

这里面，咱们会取得`onStartCommand`的返回值，再经过`Binder`将该返回值传回到`AMS`端，在`AMS`的`mServices`的`serviceDoneExecutingLocked`中，会根据该返回值来修改`ServiceRecord`的属性，这也就是咱们常说的`onStartCommand`方法的返回值，会影响到该`Services`以后的一些行为的缘由，关于这些返回值之间的差异，能够查看`Service.java`的注释，也能够查看网上的一些教程

### 2.2 进程已经存在，可是 Service 没有启动

下面，咱们来看`Service`没有启动的状况，这里会调用`realStartServiceLocked`方法：

```
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        if (DEBUG_MU)
            Slog.v(TAG_MU, "realStartServiceLocked, ServiceRecord.uid = " + r.appInfo.uid
                    + ", ProcessRecord.uid = " + app.uid);
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        bumpServiceExecutingLocked(r, execInFg, "create");
        //更新Service所在进程的oom_adj值。
        mAm.updateLruProcessLocked(app, false, null);
        mAm.updateOomAdjLocked();

        boolean created = false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod = r.shortName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            //通知应用端建立Service对象。
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
                if (newService) {
                    app.services.remove(r);
                    r.app = null;
                }
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }
        //这里面会遍历 ServiceRecord.bindings 列表，由于咱们这里是用startService方式启动的，所以该列表为空，什么也不会调用。
        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null));
        }
        //这个在第一种状况中分析过了，会调用 onStartCommand 方法。
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMap(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop! r.delayedStop = false; if (r.startRequested) { if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Applying delayed stop (from start): " + r); stopServiceLocked(r); } } } 
```

这段逻辑，有三个地方须要注意：

- `app.thread.scheduleCreateService`：经过这里会在应用进程建立`Service`对象
- `requestServiceBindingsLocked`：若是是经过`bindService`方式启动的，那么会去调用它的`onBind`方法。
- `sendServiceArgsLocked`：正如`2.1`节中所分析，这里最终会触发应用进程的`Service`的`onStartCommand`方法被调用。

第二点因为咱们今天分析的是`startService`的方式，因此不用在乎，而第三点咱们以前已经分析过了。因此咱们主要关注`app.thread.scheduleCreateService`这一句，和`2.1`中的过程相似，它会回调到`H`中的下面这个消息处理分支当中：

具体的 `handleCreateService`的处理逻辑以下所示：

```
private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            //根据Service的名字，动态的加载该Service类。
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            //1.建立ContextImpl对象。
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            //2.将 Service 做为它的成员变量 mOuterContext 。
            context.setOuterContext(service);
            //3.建立Application对象，若是已经建立那么直接返回，不然先调用Application的onCreate方法。
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            //4.调用attach方法，将ContextImpl做为它的成员变量mBase。
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            //5.调用Service的onCreate方法。
            service.onCreate();
            //6.将该Service对象缓存起来。
            mServices.put(data.token, service);
            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                // nothing to do.
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
```

以上的逻辑分为如下几个部分：

- 根据`Service`的名字，经过`ClassLoader`加载该类，并生成一个`Service`实例。
- 建立一个新的`ContextImpl`对象，并将第一步中建立的`Service`实例做为它的`mOuterContext`变量。
- 建立`Application`对象，这里会先判断当前进程所对应的`Application`对象是否已经建立，若是已经建立，那么就直接返回，不然会先建立它，并依次调用它的`attachBaseContext(Context context)`和`onCreate()`方法。
- 调用`Service`的`attach`方法，初始化成员变量

- 调用`Service`的`onCreate()`方法。
- 将该`Service`经过`token`做为`key`，缓存在`mServices`列表当中，以后`Service`生命周期的回调，都依赖于该列表

### 2.3 进程不存在的状况

在这种状况下，`Service`天然也不会存在，咱们会走到`mAm.startProcessLocked`的逻辑，这里会去启动`Service`所在的进程，它到底是怎么启动的咱们先不去细看，只须要知道它是这个目的就能够了，此外，还须要注意的是它将该`ServiceRecord`加入到了`mPendingServices`中。

对于`Service`所在的进程，它的入口函数为`ActivityThread`的`main()`方法，在`main`方法中，会建立一个`ApplicationThread`对象，并调用它的`attach`方法

而在 `attach`方法中，会调用到 `AMS`的 `attachApplication`方法

在 `attachApplication`方法中，又会去调用内部的 `attachApplicationLocked`

这里面的逻辑比较长，咱们只须要关注下面这句，咱们又看到了熟悉的 `mServices`对象

在 `ActiveServices`中的该方法中，就会去遍历前面谈到的 `mPendingServices`列表，再依次调用 `realStartServiceLocked`方法，至于这个方法作了什么，你们能够回到前面的 `2.2`节去看，这里就再也不重复分析了

## 3、小结

以上就是`startService`的整个流程，`bindService`也是相似一个调用过程，其过程并不复杂，本质上仍是 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现所谈到的通讯过程，咱们所须要学习的是`AMS`端和`Service`所在的应用进程对于`Service`是如何管理的。

系统当中的全部`Service`都是经过`AMS`的`mServices`变量，也就是`ActiveServices`类来进行管理的，而且每个应用进程中的`Service`都会在`AMS`端会对应一个`ServiceRecord`对象，`ServiceRecord`中维护了应用进程中的`Service`对象所须要的状态信息。

而且，不管咱们调用多少次`startService`方法，在应用进程侧都会只存在一个`Service`的实例，它被存储到`ActivityThread`的`ArrayMap`类型的`mServices`变量当中

# ContentProvider 源码解析

## 1、前言

在 Framework 源码解析知识梳理(5) - startService 源码分析中，咱们分析了`Service`启动的内部实现原理，今天，咱们趁热打铁，看一下`Android`中的四大组件中另外一个组件`ContentProvider`

## 2、源码解析

### 2.1 ContentResolver 获取过程

在使用`ContentProvider`来进行数据的增删改查时，第一步就是要经过`getContentResolver()`，得到一个`ContentResolver`对象，该方法实际上调用了基类中的`mBase`变量，也就是`ContextImpl`中的`getContentResolver()`方法，并返回它其中的`mContentResolver`变量

```
//ContextImpl.java

    @Override
    public ContentResolver getContentResolver() {
        return mContentResolver;
    }
```

而该`mContentResolver`是在`ContextImpl`的构造函数中初始化的，分析的`getResources()`方法返回一个`Resources`对象的过程相似。数据结构

```
private ContextImpl(ContextImpl container, ActivityThread mainThread,
            LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted,
            Display display, Configuration overrideConfiguration, int createDisplayWithId) {
        //...
        mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    }
```

这上面的`ApplicationContentResolver`是`ContentResolver`的子类

### 2.2 简单的查询过程

如今，咱们以`ContentResolver`所提供的`query`方法为例，对`ContentProvider`的调用过程进行一次简单的走读：

```
public final @Nullable Cursor query(final @NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder, @Nullable CancellationSignal cancellationSignal) {
        Preconditions.checkNotNull(uri, "uri");
        //1.获取ContentProvider接口。
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        Cursor qCursor = null;
        try {
            long startTime = SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                //2.建立取消信号量。
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            try {
                //3.调用IContentProvider的query方法。
                qCursor = unstableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                //若是发生了异常，那么销毁unstableProvider对象，从新获取一个stableProvider对象。
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                //若是stableProvider对象仍是为空，那么直接返回空。
                if (stableProvider == null) {
                    return null;
                }
                //调用stableProvider进行查询。
                qCursor = stableProvider.query(mPackageName, uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }

            // Force query execution.  Might fail and throw a runtime exception here.
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);

            //用CursorWrapperInner把qCursor包裹起来。
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            qCursor = null;
            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (qCursor != null) {
                qCursor.close();
            }
            if (cancellationSignal != null) {
                cancellationSignal.setRemote(null);
            }
            if (unstableProvider != null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```

咱们对上面的流程进行一个简单的梳理：

- 经过`acquireUnstableProvider`获取一个`unstableProvider`实例，按字面上的翻译它是一个不稳定的`ContentProvider`。
- 经过第一步中获取的`unstableProvider`实例进行查询，若是查询成功，那么获得`qCursor`对象；若是`ContentProvider`所对应的进程已经死亡，那么将会释放`unstableProvider`对象，再经过调用`acquireProvider`方法从新获得一个`stableProvider`，它和`unstableProvider`相同，都是实现了`IContentProvider`接口，以后在经过它来查询获得`qCursor`。
- 把第二步中得到的`qCursor`用`CursorWrapperInner`包裹起来，这里须要注意的是第二个参数，若是是经过`unstableProvider`查询获得的`qCursor`，那么将须要调用`acquireProvider`，并将返回值传入。

那么，咱们接下来就要分析经过`acquireUnstableProvider`、`acquireProvider`获取`IContentProvider`的过程。源码分析

### 2.3 IContentProvider 获取过程

首先，经过`acquireUnstableProvider`方法根据`Uri`中的`authority`字段，调用`acquireUnstableProvider(Context c, String auth)`方法

该方法是由咱们前面看到的 `ApplicationContentResolver`所实现的

能够看到，这里调用了 `mMainThread`的 `acquireProvider`方法，它其实是一个 `ActivityThread`实例，其实现为：

```
public final IContentProvider acquireProvider(
            Context c, String auth, int userId, boolean stable) {
        //首先从缓存中获取，若是获取到就直接返回。
        final IContentProvider provider = acquireExistingProvider(c, auth, userId, stable);
        if (provider != null) {
            return provider;
        }

        IActivityManager.ContentProviderHolder holder = null;
        try {
            //若是缓存当中没有，那么首先经过AMS进行获取。
            holder = ActivityManagerNative.getDefault().getContentProvider(
                    getApplicationThread(), auth, userId, stable);
        } catch (RemoteException ex) {
        }
        if (holder == null) {
            Slog.e(TAG, "Failed to find provider info for " + auth);
            return null;
        }

        //根据返回的holder信息进行安装。
        holder = installProvider(c, holder, holder.info,
                true /*noisy*/, holder.noReleaseNeeded, stable);
        return holder.provider;
    }
```

这里，首先会去缓存中查找`IContentProvider`，若是没有找到，那么在调用`AMS`的方法去查找，获取一个`ContentProviderHolder`对象

#### 2.3.1 调用者进程不存在缓存的状况

在这种状况下面，会执行两步操做：

- 第一步：经过`ActivityManagerService`获取`ContentProviderHolder`
- 第二步：经过返回的`ContentProviderHolder`中的信息进行安装

##### 第一步，经过 ActivityManagerService 获取 ContentProviderHolder

这里咱们先假设没有缓存的状况，经过 Framework 源码解析知识梳理(1) - 应用进程与 AMS 的通讯实现中学到的知识，咱们知道它最终会调用到`ActivityManagerService`的下面这个方法

接下来最终会调用到 `getContentProviderImpl`方法返回一个 `ContentProviderHolder`对象，这个方法比较长，就不贴代码了，直接说结论，这里会分为如下几种状况：

**(a) ContentProvider 所在进程已经启动，而且已经该 ContentProvider 已经被安装**

这种状况下，直接返回该`ContentProviderHolder`便可

**(b) ContentProvider 所在进程已经启动，可是该 ContentProvider 没有被安装**



此时，就须要经过`ApplicationThread`对象，再和`ContentProvider`所在的进程进行交互，以返回一个`ContentProviderHolder`实例：

通过 `Binder`通讯，那么最终会调用到 `ContentProvider`所在进程的下面这个方法：

这里面调用有调用了内部的 `installContentProviders`方法

这里的操做分为两步：



- 安装：根据传过来的`List<ProviderInfo>`对象，经过`installProvider`方法进行安装，并将结果存放在`List<ContentProviderHolder>`列表中。
- 发布：将安装的结果，再经过一次消息传递，返回给`ActivityManagerService`。

**(b-1) 安装过程**

在这一步当中，传入的第二个参数`holder`为`null`，所以会根据`Provider`的名字，动态地加载该类，并调用它的`attachInfo`方法

咱们上面的有两个 `Provider`：



- `localProvider`，类型为`ContentProvider`
- `provider`，类型为`Transport`

`provider`是经过`localProvider`的`getIContentProvider`方法得到的，它是`ContentProvider`的一个内部类，它的做用就是做为`ContentProvider`在远程调用者中的一个代理对象，也就是说，`ContentProvider`的使用者是经过获取`ContentProvider`所在进程的一个代理类`Transport`，再经过这个`Transport`对象调用到`ContentProvider`进行查询的：

接下来，还会去调用 `localProvider`的 `attachInfo`方法，这里面会初始化权限相关的信息，最终会执行 `ContentProvider`的 `onCreate()`方法：

假设上面咱们得到的 `localProvider`不为空，那么会执行下面的逻辑

这里面，咱们会生成一个 `ProviderClientRecord`对象，其内部包含了下面几个变量：

- `mNames`：`ContentProvider`对象的`authority`
- `mProvider`：远程代理对象
- `mLocalProvider`：本地对象
- `mHolder`：返回给`AMS`的数据结构，`AMS`再会把它返回给`ContentProvider`的调用者，`mHolder`的类型为`IActivityManager.ContentProviderHolder`，其内部包含的数据结构

**(b-2) 发布过程**



发布过程，其实就是调用了`ActivityManagerService`的`publishContentProviders`方法，将在`ContentProvider`拥有者所建立的`List<ContentProviderHolder>`保存起来

**(c) ContentProvider 所在进程没有启动**

在这种状况下，就须要先经过`startProcessLocked`启动`ContentProvider`所在进程，等待进程启动完毕以后，再进行安装。

##### 第二步，利用返回的 ContentProviderHolder 中的信息，进行安装

在第一步中，经过`ActivityManagerService`，咱们最终得到了`ContentProviderHolder`对象，接下来就是调用`installProvider`方法，这里和咱们以前在第一步中的`(b-1)`中所看到的`installProvider`实际上是同一个方法，区别在于，以前咱们分析的`installProvider`传入的`holder`参数为空

在 `installProviderAuthoritiesLocked`方法中，会将它缓存在 `mProviderMap`当中

#### 2.3.2 调用者进程存在缓存的状况

当调用者进程存在缓存时，会调用`acquireExistingProvider`方法，这里面就会经过咱们前面所看到的`mProviderMap`进行查找

## 小结

这篇文章总算是完成了，源码看的真的头晕，其实最终看下来，发现整个调用过程，和咱们上面分析过的 Framework 源码解析知识梳理(5) - startService 源码分析很相似，究其根本，就是调用者进程、全部者进程和`ActivityManagerService`进程的三方调用

