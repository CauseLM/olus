## Jetpack 全家桶

### 【背上Jetpack之Lifecycle】万物基于 Lifecycle 默默无闻大用处

![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d987d578b78?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 前言

> Android 中有一个比较重要的概念：「生命周期」。刚毕业去面试，总会被问到「四大组件的生命周期」这类的问题。17年的 IO 大会上，Google 推出了 Lifecycle-Aware Components（生命周期感知组件），帮助开发者组织更好，更轻量，易于维护的代码

本文介绍 `Lifecycle` 的职责以及简单分析 lifecycle 如何感知 activity 和 fragment ，帮助您对 `Lifecycle` 有一个感性的认识

#### 万物基于 Lifecycle

#### 手动管理生命周期的痛苦你不懂



![lifecycles](https://user-gold-cdn.xitu.io/2020/3/30/1712bbbbf69f3b1e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 鲁迅曾说过：万物基于 Lifecycle



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d53c2ae3262?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



哦不对



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d41f43aab70?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



Android 中的视图控制器就有这么多生命周期的情况，所以处理好生命周期十分重要，否则会导致内存泄漏甚至是程序崩溃。这里引用 [官方文档](https://developer.android.com/topic/libraries/architecture/lifecycle) 的例子

```
class MyLocationListener {
    public MyLocationListener(Context context, Callback callback) {
        // ...
    }

    void start() {
        // 连接系统的定位服务
    }

    void stop() {
        // 与系统的定位服务断开连接
    }
}

class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    @Override
    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, (location) -> {
            // 更新 UI
        });
    }

    @Override
    public void onStart() {
        super.onStart();
        myLocationListener.start();
        //管理其他需要响应 activity 生命周期的组件
    }

    @Override
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
        //管理其他需要响应 activity 生命周期的组件
    }
}
   
```

此示例看起来不错，在实际的应用程序中，您仍然会响应生命周期的当前状态而进行过多的调用来管理 UI 和其他组件。 管理多个组件会在生命周期方法中放置大量代码，例如 onStart() 和 onStop()，这使它们难以维护

而且，不能保证组件在 activity 或 fragment 停止之前就已启动。 如果我们需要执行长时间运行的操作（例如onStart() 中的某些配置检查），则可能会导致争用情况，其中onStop() 方法在 onStart() 之前完成，从而使组件的生存期超过了所需的生存期。

```
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // 更新 UI
        });
    }

    @Override
    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            // 如果在 activity 停止后调用此回调怎么办？
            if (result) {
                myLocationListener.start();
            }
        });
    }

    @Override
    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
   
```

如果有所有的组件，都能感知外部的生命周期，能在相应的时机释放资源，并且在错过生命周期时能及时叫停异步的任务就好了，



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d41f6da8083?imageslim)



我们不妨先思考一下，如果实现这样的想法，应该如何做

#### 按照惯例的思考

首先我们先来整理一下我们的需求

- 内部组件能够感知外部的生命周期
- 能够统一地管理，做到一处修改，处处生效
- 能够及时叫停错过的任务

针对需求1，可以用观察者模式，内部组件能够在外部生命周期变化时做出相应

针对需求2，可以将依赖组件的代码移出生命周期方法内，然后移入组件本身，这样只需修改组件内部逻辑即可

针对需求3，可以在合适的时机移除观察者

#### 观察者模式

关于观察者模式，我第一次比较详细的了解是在 [扔物线](https://juejin.im/user/2524134386185736) 的 [给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)。

> 观察者模式面向的需求是：A 对象（观察者）对 B 对象（被观察者）的某种变化高度敏感，需要在 B 变化的一瞬间做出反应。举个例子，新闻里喜闻乐见的警察抓小偷，警察需要在小偷伸手作案的时候实施抓捕。在这个例子里，警察是观察者，小偷是被观察者，警察需要时刻盯着小偷的一举一动，才能保证不会漏过任何瞬间。程序的观察者模式和这种真正的『观察』略有不同，观察者不需要时刻盯着被观察者（例如 A 不需要每过 2ms 就检查一次 B 的状态），而是采用 **注册(Register)** 或者称为 **订阅(Subscribe)** 的方式，告诉被观察者：我需要你的某某状态，你要在它变化的时候通知我。 Android 开发中一个比较典型的例子是点击监听器 `OnClickListener` 。对设置 `OnClickListener` 来说， `View` 是被观察者， `OnClickListener` 是观察者，二者通过 `setOnClickListener()` 方法达成订阅关系。订阅之后用户点击按钮的瞬间，Android Framework 就会将点击事件发送给已经注册的 `OnClickListener` 。采取这样被动的观察方式，既省去了反复检索状态的资源消耗，也能够得到最高的反馈速度。当然，这也得益于我们可以随意定制自己程序中的观察者和被观察者，而警察叔叔明显无法要求小偷『你在作案的时候务必通知我』。

OnClickListener 的模式大致如下图：



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d41f70250a0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 上述描述及图片均来自 [给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)

**因此在生命周期组件的生命周期发生变化时告诉观察者，内部组件即可感知外部的生命周期**

#### 引入 Lifecycle 后

```
public class MyObserver implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void connectListener() {
        ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void disconnectListener() {
        ...
    }
}

myLifecycleOwner.getLifecycle().addObserver(new MyObserver());
   
```

#### 源码结构



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d41f87b43d3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



这是 [Lifecycle](https://developer.android.com/reference/androidx/lifecycle/Lifecycle)  的结构，抽象类，其内部有两个枚举，分别代表着「事件」和「状态」，此外还有三个方法，添加/移除观察者，获取当前状态

> **注意，这里 State 中的枚举顺序是有意义的，后文详细介绍**

其实现类为  [LifecycleRegistry](https://developer.android.com/reference/androidx/lifecycle/LifecycleRegistry) ，可以处理多个观察者



![LifecycleRegistry](https://user-gold-cdn.xitu.io/2020/3/31/17130d41f994a036?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



其内部持有当前的状态 mState ，LifecycleOwner 以及观察者的自定义列表，同时重写了父类的添加/删除观察者的方法



![LifecycleOwner](https://user-gold-cdn.xitu.io/2020/3/31/17130d422416eb17?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



[LifecycleOwner](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner) ，具有 Android 的生命周期，定制组件可以使用这些事件来处理生命周期更改，而无需在 Activity 或 Fragment 中实现任何代码

[LifecycleObserver](https://developer.android.com/reference/androidx/lifecycle/LifecycleObserver) ，将一个类标记为 `LifecycleObserver`。 它没有任何方法，而是依赖于 OnLifecycleEvent 注解的方法

[LifecycleEventObserver](https://developer.android.com/reference/androidx/lifecycle/LifecycleEventObserver) ，可以接收任何生命周期更改并将其分派给接收方。

**如果一个类实现此接口并同时使用 OnLifecycleEvent，则注解将被忽略**

[DefaultLifecycleObserver](https://developer.android.com/reference/androidx/lifecycle/DefaultLifecycleObserver) ，用于监听 [LifecycleOwner](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner) 状态更改的回调接口。

如果一个类同时实现了此接口和 [LifecycleEventObserver](https://developer.android.com/reference/androidx/lifecycle/LifecycleEventObserver)，则将首先调用`DefaultLifecycleObserver` 的方法，然后再调用LifecycleEventObserver.onStateChanged（LifecycleOwner，Lifecycle.Event）

> 注意：使用 [DefaultLifecycleObserver](https://developer.android.com/reference/androidx/lifecycle/DefaultLifecycleObserver) 需引入
>
> implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

#### 简单的源码分析

#### activity 生命周期处理

首先我们还是来看 **androidx.activity.ComponentActivity** ，这个类我们这个系列的文章里提到多次，第一次提及是在 [【背上Jetpack】绝不丢失的状态 androidx SaveState ViewModel-SaveState 分析](https://juejin.im/post/6844904097351467015#heading-2) ，感兴趣的小伙伴可以看看。



![ComponentActivity](https://user-gold-cdn.xitu.io/2020/3/31/17130d4225c4298f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



其实现的接口大多数我们都已经探讨过了，今天我们来看看 LifecycleOwner

> ActivityResultCaller 为 activity 1.2.0-alpha02 推出的，旨在统一 onActivityResult ，这里暂时不讨论它



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d4225724425?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



既然实现了 `LifecycleOwner` 接口，必定重写 getLifecycle() 方法

```
// androidx.activity.ComponentActivity.java
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
   
```

其返回的 `Lifecycle` 为 实现类 `LifecycleRegistry` 的实例

而 activity 操作生命周期是通过 `ReportFragment` 处理的

```
// androidx.activity.ComponentActivity.java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
    //...
}

// ReportFragment
public static void injectIfNeededIn(Activity activity) {
    if (Build.VERSION.SDK_INT >= 29) {
        // api 29 及以上 直接注册正确的生命周期回调
        activity.registerActivityLifecycleCallbacks(
                new LifecycleCallbacks());
    }
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        manager.executePendingTransactions();
    }
}
   
```



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130d422c832f40?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```
// ReportFragment.java
static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }
    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}

private void dispatch(@NonNull Lifecycle.Event event) {
    if (Build.VERSION.SDK_INT < 29) {
        dispatch(getActivity(), event);
    }
}

@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatch(Lifecycle.Event.ON_CREATE);
}
@Override
public void onStart() {
    super.onStart();
    dispatch(Lifecycle.Event.ON_START);
}
@Override
public void onResume() {
    super.onResume();
    dispatch(Lifecycle.Event.ON_RESUME);
}
@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}
@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}
@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
}
   
// LifecycleCallbacks
static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
    @Override
    public void onActivityPostCreated(@NonNull Activity activity,
            @Nullable Bundle savedInstanceState) {
        dispatch(activity, Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onActivityPostStarted(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_START);
    }

    @Override
    public void onActivityPostResumed(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_RESUME);
    }
    @Override
    public void onActivityPrePaused(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onActivityPreStopped(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onActivityPreDestroyed(@NonNull Activity activity) {
        dispatch(activity, Lifecycle.Event.ON_DESTROY);
    }
	//...
}
   
```

在 activity 的 onCreate 方法中，调用了 `ReportFragment` 中的静态方法 `injectIfNeededIn()` 。而其内部，**如果 api 29 及以上的设备上直接注册正确的生命周期回调，低版本通过启动 ReportFragment ，借助 fragment 各个生命周期来处理生命周期回调**

#### fragment 生命周期处理

在 fragment 内部，每个生命周期节点调用 `handleLifecycleEvent` 方法

```
// Fragment.java
public Fragment() {
    initLifecycle();
}

private void initLifecycle() {
    mLifecycleRegistry = new LifecycleRegistry(this);
}

@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}

void performCreate(Bundle savedInstanceState) {
    onCreate(savedInstanceState);
	mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);    
}

void performStart() {
    onStart();
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
}

void performResume() {
    onResume();
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);    
}

void performPause() {
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
    onPause();
}

void performStop() {
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_STOP);
    onStop();
}

void performDestroy() {
    mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    onDestroy();
}
   
```

#### Lifecycle State 大小比较

`Lifecycle.State`  中有一个 `isAtLeast` 方法，用于判断当前状态是否不小于传入的状态

```
// Lifecycle.State
public boolean isAtLeast(@NonNull State state) {
    return compareTo(state) >= 0;
}
   
```

**枚举的 compareTo 方法其实是比较的枚举声明的顺序**

而 State 的顺序为 DESTROYED -> INITIALIZED -> CREATED -> STARTED -> RESUMED

> 如果传入的 state 为 STARTED，则当前状态为 STARTED 或 RESUMED 时返回 true ，否则返回 false
>
> LiveData 篇会用到这个知识点

### 【背上Jetpack之ViewModel】即使您不使用MVVM也要了解ViewModel ——ViewModel 的职能边界

![目录](https://user-gold-cdn.xitu.io/2020/3/23/171066ab61224844?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 前言

> Android 开发时，我们使用 activity 和 fragment 作为视图控制器， 可能还会使用有一些类可以存储和提供 UI 数据（例如MVP中的 `Presenter` ）

但是 当配置更改时（如旋转屏幕），activity 会重建，但对于 UI 数据的持有者呢？

- 开发者需要重新保存相关的信息并传递给重建的 activity ，否则开发者必须再次获取数据（通过网络请求或本地数据库）
- 由于 UI 数据的持有者的生命周期可能比 activity 长，因此开发者还需要避免出现内存泄漏的问题

如何解决上述问题？ViewModel

**本文重点介绍 ViewModel 的职责（what）以及重点功能的实现原理（how），即使您不使用 `Jetpack MVVM` 架构，也要了解一下 ViewModel**

ViewModel 的原理部分要求您了解 activity 的启动流程，这部分内容网上文章很多，本文不再赘述

#### ViewModel 的职责

我先上个 [视频](https://www.bilibili.com/video/av97794796/) ，这个小姐姐表述的比文字更形象



[![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab53361337?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://www.bilibili.com/video/av97794796/)



`ViewModel` 主要用于存储 UI 数据以及生命周期感知的数据



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab567fd05e?imageslim)



> 图片来自 [Android Architecture Components: ViewModel](https://android.jlelse.eu/android-architecture-components-viewmodel-e74faddf5b94)



![ViewModel生命周期](https://user-gold-cdn.xitu.io/2020/3/23/171066ab577f86bb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> `ViewModel` 的生命周期 ，图片来自 [官方文档](https://developer.android.com/topic/libraries/architecture/viewmodel#lifecycle)

#### 作为数据持有者

`ViewModel` 能够实时进行配置更改。 这意味着即使在手机旋转后销毁并重新创建 activity 之后，您仍然拥有相同的 `ViewModel` 和相同的数据。 因此：

- 您无需担心 UI 数据持有者的生命周期。 `ViewModel` 将由工厂自动创建，您无需自行创建和销毁
- 数据将始终更新，旋转手机后，您将获得与以前相同的数据。 因此，您无需手动将数据传递给新的 activity 实例或再次调用网络或数据库来获取数据。

#### Fragment 间共享数据

一个 activity 中的两个或更多 fragment 需要相互通信是很常见的。例如您有一个片段，用户在其中从列表中选择一个 item，另一个片段显示了所选 item 的内容。 传统做法两个 fragment 都需要定义一些接口，并且宿主 activity 必须将两者绑定在一起。 此外，两个 fragment 都必须处理另一个 fragment 尚未创建或不可见的情况。

可以通过使用 `ViewModel` 对象解决此问题。 这些 fragment 可以使用 activity 范围内共享一个 `ViewModel` 来处理此通信，如以下示例代码所示：

```
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}


public class MasterFragment extends Fragment {
    private SharedViewModel model;

    public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        model = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {

    public void onViewCreated(@NonNull View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        SharedViewModel model = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
        model.getSelected().observe(getViewLifecycleOwner(), { item ->
           // Update the UI.
        });
    }
}
   
```

> 由于 两个 fragment 使用的都是 activity 范围的 `ViewModel` （`ViewModelProvider` 构造器传入的 activity ），因此它们获得了相同的 ViewModel 实例，自然其持有的数据也是相同的，这也 **保证了数据的一致性**

这种方法具有以下优点：

- 宿主 activity 无需执行任何操作，也无需了解此通信。
- 除 `SharedViewModel` 外，fragment 不需要彼此了解。 如果其中一个 fragment 消失了，则另一个继续照常工作。
- 每个 fragment 都有其自己的生命周期，并且不受另一个 fragment 的生命周期影响。 如果一个 fragment 替换了另一个 fragment，则 UI 可以继续正常工作而不会出现任何问题。

#### 代替 Loader

`CursorLoader` 这样的 Loader 类经常用于使应用程序 UI 中的数据与数据库保持同步。您可以使用 `ViewModel` 和其他一些类来替换 Loader。 使用 `ViewModel` 可将视图控制器与数据加载操作分开，这意味着您在类之间的强引用较少。

在使用 Loader 的一种常见方法中，应用程序可能会使用 `CursorLoader` 来观察数据库的内容。 当数据库中的值更改时，加载程序会自动触发数据的重新加载并更新 UI



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab58534ee7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 图片来自 [官方文档](https://developer.android.com/topic/libraries/architecture/viewmodel#loaders)

`ViewModel` 与 `Room` 和 `LiveData` 一起使用以替换 Loader。 `ViewModel` 确保数据在设备配置更改后仍然存在。 当数据库发生更改时，`Room` 会通知 `LiveData` ，然后 `LiveData` 会使用修改后的数据更新 UI



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab66d2c4e3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 图片来自 [官方文档](https://developer.android.com/topic/libraries/architecture/viewmodel#loaders)

#### 总结

- **ViewModel 可作为 UI 数据的持有者，在 activity/fragment 重建时 ViewModel 中的数据不受影响，同时可以避免内存泄漏**
- **可以通过 ViewModel 来进行 activity 和 fragment ，fragment 和 fragment 之间的通信，无需关心通信的对方是否存在，使用 application 范围的 ViewModel 可以进行全局通信**
- **可以代替 Loader**

#### ViewModel 源码分析

分析源码时我们可以不计较细枝末节，只分析主要的逻辑即可。因此我们来思考几个问题，并从源码中寻找答案

- 如何做到 activity 重建后 `ViewModel` 仍然存在？
- 如何做到 fragment 重建后 `ViewModel` 仍然存在？
- 如何控制作用域？（即保证相同作用域获取的 `ViewModel` 实例相同）
- 如何避免内存泄漏？

维持我们一贯的风格，我们先来大胆地猜一猜

对于问题1 ：activity 有着 `saveInstanceState` 机制，因此可能通过该机制来处理（**事实证明不是**）

对于问题2：可能 fragment 通过 宿主 activity 或 父 fragment 的帮助来确保 `ViewModel` 实例在重建后仍然存在

对于问题3：实现一个类似单例的效果，相同作用域获取的对象是相同的

对于问题4：避免 `ViewModel` 持有 view 或 context 的引用

首先我们要先了解一下 `ViewModel` 的结构

- `ViewModel`：抽象类，主要有 clear 方法，它是 final 级，不可修改，clear 方法中包含 onClear 钩子，开发者可重写 onClear 方法来自定义数据的清空
- `ViewModelStore`：内部维护一个 HashMap 以管理 `ViewModel`
- `ViewModelStoreOwner`：接口，`ViewModelStore` 的作用域，实现类为 `ComponentActivity` 和 `Fragment`，此外还有 `FragmentActivity.HostCallbacks`
- `ViewModelProvider`：用于创建 `ViewModel`，其构造方法有两个参数，第一个参数传入 `ViewModelStoreOwner` ，确定了 `ViewModelStore` 的作用域，第二个参数为 `ViewModelProvider.Factory`，用于初始化 `ViewModel` 对象，默认为 `getDefaultViewModelProviderFactory()` 方法获取的 factory

简单来说 **ViewModelStoreOwner 持有 ViewModelStore 持有 ViewModel**



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab8548ed9d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### 1. 如何做到 activity 重建后 ViewModel 仍然存在？

在 [【背上Jetpack】绝不丢失的状态 androidx SaveState ViewModel-SaveState 分析](https://juejin.im/post/6844904097351467015) 中我们提到了 androidx.core.app.ComponentActivity 的引入并探讨了其作为中间层的作用



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab88e4f7b3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



我们已经讲过 `SavedStateRegistryOwner` 和 `OnBackPressedDispatcherOwner` 这两种角色，而今天我们来聊一下

```
ViewModelStoreOwner` 和 `HasDefaultViewModelProviderFactory` 。其中前者代表着 `ViewModelStore` 的作用域，后者来标记 `ViewModelStoreOwner` 拥有默认的 `ViewModelProvider.Factory
```

那么 `ViewModel` 的逻辑肯定就在该类了

`ComponentActivity` 实现了 `ViewModelStoreOwner`  接口，意味着需要重写 `getViewModelStore()` 方法，该方法为 `ComponentActivity`  的 `mViewModelStore` 变量赋值。**activity 重建后 ViewModel 仍然存在，只要保证 activity 重建后 mViewModelStore 变量值不变即可**

顺着这个思路，我们来看一下 `getViewModelStore()` 的实现

```
public ViewModelStore getViewModelStore() {
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            //核心，在该位置重置 mViewModelStore
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
   
```

即 `mViewModelStore` 的值由 `getLastNonConfigurationInstance()` 返回的 `NonConfigurationInstances` 对象中的 `viewModelStore` 赋值，如果此时还为空才去 new ViewModelStore 对象。因此我们只需找到

`getLastNonConfigurationInstance` 中的 `NonConfigurationInstances` 在哪里保存的即可

```
getLastNonConfigurationInstance` 为平台 activity 中的方法，返回 `mLastNonConfigurationInstances.activity
public Object getLastNonConfigurationInstance() {
    return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
}
   
```

那么我们看一下 `mLastNonConfigurationInstances` 的赋值位置

```
//省略其他参数
final void attach(NonConfigurationInstances lastNonConfigurationInstances){
	mLastNonConfigurationInstances = lastNonConfigurationInstances;
    //...
}
   
```

了解过 activity 的启动流程的小伙伴肯定知道，这个 attach 方法是 `ActivityThread` 中的 `performLaunchActivity` 调用的

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Activity activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    //省略其他参数
    activity.attach(r.lastNonConfigurationInstances);
    r.lastNonConfigurationInstances = null;
    //...
}
   
```

深入追踪源码我们整理一下调用流程



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066ab91099f6d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



由于 `ActivityThread` 中的 `ActivityClientRecord` 不受 activity 重建的影响，所以 activity 重建时 `mLastNonConfigurationInstances` 能够得到上一次的值，使得 `ViewModelStore` 值不变 ，问题1就解决了

#### 2. 如何做到 fragment 重建后 ViewModel 仍然存在？

对于问题2，有了上面的思路我们可以认定 fragment 重建后其内部的 `getViewModelStore()` 方法返回的对象是相同的。

```
// Fragment.java
public ViewModelStore getViewModelStore() {
    return mFragmentManager.getViewModelStore(this);
}
   
```

可以看到 `getViewModelStore()` 内部调用的是 `mFragmentManager`（普通fragment 对应 activity 中的 `FragmentManager`，子 fragment 则对应父 fragment 的 `childFragmentManager`）的 `getViewModelStore()` 方法

```
// FragmentManager.java
private FragmentManagerViewModel mNonConfig;

ViewModelStore getViewModelStore(@NonNull Fragment f) {
    return mNonConfig.getViewModelStore(f);
}
   
```

**而 FragmentManager 中的 getViewModelStore 使用的是 mNonConfig ，mNonConfig 竟然是个 ViewModel！**

```
// FragmentManagerViewModel.java
private final HashMap<String, FragmentManagerViewModel> mChildNonConfigs = new HashMap<>();
private final HashMap<String, ViewModelStore> mViewModelStores = new HashMap<>();
   
```

`FragmentManagerViewModel` 管理着内部的 `ViewModelStore` 和 child 的 `FragmentManagerViewModel` 。因此保证 mNonConfig   值不变即能确保 fragment 中的 `getViewModelStore()`  不变。那么看看 mNonConfig  赋值的位置

```
// FragmentManager.java
void attachController(@NonNull FragmentHostCallback<?> host, @NonNull FragmentContainer container, @Nullable final Fragment parent) {
    //...
    if (parent != null) {
        // 嵌套 fragment 的情况，有父 fragment
        mNonConfig = parent.mFragmentManager.getChildNonConfig(parent);
    } else if (host instanceof ViewModelStoreOwner) {
        // host 是 FragmentActivity.HostCallbacks
        ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
        mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
    } else {
        mNonConfig = new FragmentManagerViewModel(false);
    }
}


// FragmentManagerViewModel.java
static FragmentManagerViewModel getInstance(ViewModelStore viewModelStore) {
    ViewModelProvider viewModelProvider = new ViewModelProvider(viewModelStore,
            FACTORY);
    return viewModelProvider.get(FragmentManagerViewModel.class);
}
   
```

我们先看 fragment 的直接宿主是 activity （即没有嵌套）的情况，mNonConfig 由`FragmentManagerViewModel.getInstance(viewModelStore)` 赋值，而 getInstance 中使用的是 `ViewModelProvider` 获取 `ViewModel` ，根据我们上面的分析，**只要保证作用域（viewModelStore）相同，即可获取相同的 `ViewModel` 实例**，因此我们需要看一下 host 的 getViewModelStore 方法。经过一番寻找，host 是 `FragmentActivity.HostCallbacks`

```
// FragmentActivity.java 内部类
class HostCallbacks extends FragmentHostCallback<FragmentActivity> implements ViewModelStoreOwner, OnBackPressedDispatcherOwner {
    public ViewModelStore getViewModelStore() {
        // 宿主 activity 的 getViewModelStore
    	return FragmentActivity.this.getViewModelStore();
	}
}
   
```

host 的 getViewModelStore 方法返回的是宿主 activity 的 `getViewModelStore()` ，而 activity 重建后其内部的 `mViewModelStore` 是不变的，因此即使 activity 重建，其内部的 FragmentManager 对象变化，但 FragmentManager 内部的  FragmentManagerViewModel 的实例（`mNonConfig`）不变，mNonConfig.getViewModelStore 不变，fragment 的 `getViewModelStore()` 亦不变，fragment 重建后其内部的 `ViewModel` 仍然存在

对于嵌套 fragment ，mNonConfig 通过 parent.mFragmentManager.getChildNonConfig(parent) 获取

```
// FragmentManager.java
private FragmentManagerViewModel getChildNonConfig(@NonNull Fragment f) {
    return mNonConfig.getChildNonConfig(f);
}
   
```

上文提到 `FragmentManagerViewModel` 管理着 mChildNonConfigs Map，因此子 fragment 重置后其内部的 mNonConfig 对象也是相同的

至此问题 2 就解决了

#### 3. 如何控制作用域？

对于问题3，我们知道 `ViewModelStoreOwner` 代表着作用域，其内部唯一的方法返回 `ViewModelStore` 对象，也即不同的作用域对应不同的 `ViewModelStore` ，而 `ViewModelStore`  内部维护着 `ViewModel` 的 HashMap ，因此只要保证相同作用域的 `ViewModelStore` 对象相同就能保证相同作用域获取到相同的 `ViewModel` 对象，而问题1我们已经解释了重建时如何保证 `ViewModelStore`  对象不变。

因此问题3也解决了。

#### 4. 如何避免内存泄漏？

对于问题4，由于 `ViewModel` 的设计，使得 activity/fragment 依赖它，而 `ViewModel` 不依赖 activity/fragment。因此只要不让 `ViewModel` 持有 context 或 view 的引用，就不会造成内存泄漏

#### 总结

简单的总结一下：

- **activity 重建后 mViewModelStore 通过 ActivityThread 的一系列方法能够保持不变，从而当 activity 重建时 ViewModel 中的数据不受影响**
- **通过宿主 activity 范围内共享的 FragmentManagerViewModel 来存储 fragment 的 ViewModelStore 和子 fragment 的 FragmentManagerViewModel ，而 activity 重建后 FragmentManagerViewModel  中的数据不受影响，因此 fragment 内部的 ViewModel 的数据也不受影响**
- **通过同一 ViewModelStoreOwner 获取的 ViewModelStore 相同，从而保证同一作用域通过 ViewModelProvider 获取的 ViewModel 对象是相同的**
- **通过单向依赖（视图控制器持有 ViewModel ）来解决内存泄漏的问题**

#### ViewModel 和 onSaveInstanceState

`ViewModel` 和 `onSaveInstanceState` 的功能有些类似，但它们也有很多差异



![img](https://user-gold-cdn.xitu.io/2020/3/23/171066c1bd79cc48?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



从存储位置上来说，`ViewModel` 是在内存中，因此其读写速度更快，但当进程被系统杀死后，`ViewModel` 中的数据也不存在了。从数据存储的类型上来看，`ViewModel` 适合存储相对较重的数据，例如网络请求到的 list 数据，而 `onSaveInstanceState` 适合存储轻量可序列化的数据

那么我们该如何使用呢？可以使用 `viewmodel-savedstate` 库，详情参考 [【背上Jetpack】绝不丢失的状态 androidx SaveState ViewModel-SaveState 分析](https://juejin.im/post/6844904097351467015#heading-10)

## 【背上Jetpack之DataBinding】数据驱动魔法师 何时迎来翻身日？

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0920c6ef89f41fcb99640754667082e~tplv-k3u1fbpfcp-zoom-1.image)

### 前言

> [LiveData 篇](https://juejin.im/post/5e834bb5f265da480d61668d) 我们提到 Android 开发的主要工作内容是将数据转换为 UI ，同时我们也介绍了数据驱动 UI 的思想，使用 ViewModel + LiveData，可以安全地在订阅者的生命周期内分发正确的数据。但是 activity 和 fragment 充斥着大量的模板代码，铺天盖地的 findViewById，以及各种 set （根据数据设置 UI）。如果能够消灭掉这些模板代码就好了

![他来了他来了](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94ac786efa4140aab84394b70d346830~tplv-k3u1fbpfcp-zoom-1.image)

他来了他来了，他欢快地走来了

![DataBinding](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cdf2aa5c89a4fad96cfbc3dac6b4e73~tplv-k3u1fbpfcp-zoom-1.image)

然而，很多开发者对 DataBinding 存在偏见，「DataBinding 不是个好东西，在声明式编程中书写 UI 逻辑，既不可调试，也不便于察觉和追踪，万一出现问题就麻烦了。」

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/307d5cb318f444d9859030cdcea812aa~tplv-k3u1fbpfcp-zoom-1.image)

本文主要介绍 DataBinding 的解决的问题以及其背后的逻辑，带您对 DataBinding 有一个感性的认识。本文末尾会对各个 findViewById 的替代方案进行对比

DataBinding 的相关资源

- [官方文档](https://developer.android.com/topic/libraries/data-binding)
- [codelab](https://codelabs.developers.google.com/codelabs/android-databinding/#0)
- [官方示例](https://github.com/android/databinding-samples)

### 数据驱动魔法师

DataBinding 允许使用声明性格式而不是通过编程方式将布局中的 UI 组件与数据源绑定

```java
// before
TextView textView = findViewById(R.id.sample_text);
textView.setText(viewModel.getUserName());
    
<TextView
    android:text="@{viewmodel.userName}" />
    
```

通过在布局文件中绑定组件，您可以删除 activity 中的许多设置 UI 调用，从而使它们更易于维护。 这也可以提高应用程序的性能，并有助于防止内存泄漏和空指针异常

> 如果仅替换 findViewById 而不需要数据的绑定，可以使用 ViewBinding，它使用起来更简单，性能也更好。
>
> 使用方法参见 [[译\]深入研究ViewBinding 在 include, merge, adapter, fragment, activity 中使用](https://juejin.im/post/5e4806f3e51d4526c550a2ef)

### DataBinding 基础

详细内容参见 [官方文档](https://developer.android.com/topic/libraries/data-binding) ，这里只简单介绍 DataBinding

### DataBinding 引入

app build.gradle 中加入

```groovy
android {
    ...
    dataBinding {
        enabled = true
    }
}

// Android Studio 4.0
android {
    ...
    buildFeatures {
        dataBinding = true
    }
}

    
```

> 若在 library module 使用 DataBinding，须在 app module 和 library module 同时声明，即使 app module 没有直接使用 DataBinding

使用 DataBinding 无需开发者手动引入库，android build gradle plugin 内部已经引入了

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7d90b2fc80b41fdbc4805aecc98a5df~tplv-k3u1fbpfcp-zoom-1.image)

DataBinding 中使用了注解，因此在构建速度上比 ViewBinding 差些（不过功能这么强大要啥自行车）

#### 布局

DataBinding 布局文件略有不同，它们以 layout 的根标记开始，后跟一个 data 元素和一个 view 根元素。 view 元素是您的根将位于非绑定布局文件中的元素。 以下代码显示了一个示例布局文件：

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto">
    <data>
        <variable
            name="viewmodel"
            type="com.myapp.data.ViewModel" />
    </data>
    <ConstraintLayout... /> <!-- UI layout's root element -->
</layout>
    
```

#### 生成绑定类

DataBinding 会为每个在布局声明 layout 标签的 xml 布局文件生成一个绑定类。 默认情况下，类的名称基于布局文件的名称。 上面的布局文件名是 activity_main.xml，因此相应的生成类是 ActivityMainBinding。 此类包含从布局属性（例如，viewmodel 变量）到布局视图的所有绑定，并且知道如何为绑定表达式分配值

#### 配置绑定

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // before
    // setContentView(R.layout.activity_main)
    
    // after
    val binding : ActivityMainBinding =
    DataBindingUtil.setContentView(this, R.layout.activity_main)
}
    
```

#### 使用DataBinding 解决的问题及实现原理

不知道你是否有这些烦恼：activity 和 fragment 中有着大量的模板代码，即使使用 ButterKnife 等工具写起代码来也很繁琐。而且 View id 与 View 的类型不匹配时，只有在运行期才能发现；旋转屏幕后如果新的布局中不存在之前 id 的 view ，可能还导致空指针异常；项目中使用各类 bus 通知 UI 刷新，但是有时 UI 的显示并不符合预期，而排查起来特别困难，因为数据源很多...

不要慌，DataBinding 可以解决以下问题

- 替换 findViewById ，减少模板代码
- 解决类型安全问题
- 解决空安全问题
- 保证了数据的一致性

#### 魔法的背后

com.android.tools.build:gradle 插件中封装了 DataBinding 的魔法

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64a66170a0c0462aaa5c358a1730b42c~tplv-k3u1fbpfcp-zoom-1.image)

查看 com.android.tools.build:gradle:3.6.2 的源码，找到 DataBinding 配置项的类 DataBindingOptions

```java
// DataBindingOptions.java
@Override
public boolean isEnabled() {
    // DataBinding 是否开启，对应上面在 build.gradle 中的配置
    return enabled;
}
    
```

它的调用者很多，在 TaskManager 中的 createDataBindingTasksIfNecessary

```java
// TaskManager 
protected void createDataBindingTasksIfNecessary(@NonNull VariantScope scope) {
    // 是否开启 DataBinding
    boolean dataBindingEnabled = extension.getDataBinding().isEnabled();
    boolean viewBindingEnabled = extension.getViewBinding().isEnabled();
    if (!dataBindingEnabled && !viewBindingEnabled) {
        // DataBinding 和 ViewBinding 均未开启则直接 return
        return;
    }
    createDataBindingMergeBaseClassesTask(scope);
    createDataBindingMergeArtifactsTask(scope);
    //...
    // 构建 DataBinding 相应绑定类
    taskFactory.register(new DataBindingGenBaseClassesTask.CreationAction(scope));
}
// CreationAction
override fun handleProvider(taskProvider: TaskProvider<out DataBindingGenBaseClassesTask>) {
    variantScope.artifacts.producesDir(
        // DATA_BINDING_BASE_CLASS_SOURCE_OUT
        InternalArtifactType.DATA_BINDING_BASE_CLASS_SOURCE_OUT,
        BuildArtifactsHolder.OperationType.INITIAL,
        taskProvider,
        DataBindingGenBaseClassesTask::sourceOutFolder
	)
}
    
```

可以看到生成 DataBinding 绑定类的 task 为 DataBindingGenBaseClassesTask，而InternalArtifactType.DATA_BINDING_BASE_CLASS_SOURCE_OUT 则对应着 build 目录生成的 DataBinding 类的

data_binding_base_class_source_out 目录

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12253a291d9c4cac9fcfb51158cfe7ed~tplv-k3u1fbpfcp-zoom-1.image)

这里可以简单看一下，感兴趣的小伙伴可以自己查看源码

#### DataBinding 如何解决上述问题的

我们可以查看 DataBinding 生成的绑定类

```java
public final class FragmentSingleChildBinding implements ViewBinding {
  // NonNull 注解标记
  // 如果存在不同配置的不同布局文件（如横竖屏）且该控件不是存在于所有布局，该处使用 Nullable注解标记
  @NonNull
  public final MaterialButton button;
    
  // 省略...
    
  @NonNull
  public static FragmentSingleChildBinding bind(@NonNull View rootView) {
    String missingId;
    missingId: {
      //其内部也是使用 findViewById
      MaterialButton button = rootView.findViewById(R.id.button);
      if (button == null) {
        missingId = "button";
        break missingId;
      }
      return new FragmentSingleChildBinding((MaterialButton) rootView, button);
    }
    throw new NullPointerException("Missing required view with ID: ".concat(missingId));
  }
}
    
```

- Binding 类内部的也是使用 findViewById ，因此 DataBinding 可以代替 findViewById ，并且减少模板代码
- View 控件变量类型是固定的，因此不会出现类型安全问题
- View 控件变量由空/非空注解修饰，（如果为 Nullable java 中会有 lint 警告，而 kotlin 直接调用时无法通过编译的）因此 不会出现空安全问题
- 通过声明式的配置，UI 完全来自唯一可信的数据源配置，保证了数据的一致性

> 注意：以上分析同样适用于 ViewBinding

### 感受魔法的魅力

这里简单展示一下 DataBinding 的「魔法」

#### 基本操作

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
      
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView
           android:id="@+id/firstName"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           />
       <TextView
           android:id="@+id/lastName"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           />
   </LinearLayout>
</layout>
    
```



```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

		// Before Data Binding
		//setContentView(R.layout.activity_main);
        
		//TextView firstName = (TextView) findViewById(R.id.firstName);
		//TextView lastName = (TextView) findViewById(R.id.lastName);

		//firstName.setText("xxx");
		//lastName.setText("xxx");


		// After Data Binding
        ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);

        binding.firstName.setText("xxx");
        binding.lastName.setText("xxx");
    }
}
```



上面展示了 DataBinding 的基础操作（单纯的替换 findViewById），如果仅使用 DataBinding 这部分功能，可以考虑使用 ViewBinding

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/deb478bef26c4724904b73d667e2256f~tplv-k3u1fbpfcp-zoom-1.image)

#### 绑定数据

在之前的布局的基础上绑定数据

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView 
           android:id="@+id/firstName"      
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"/>
       <TextView 
           android:id="@+id/lastName"      
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"/>
   </LinearLayout>
</layout>
    
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ActivityMainBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_main);
		binding.user = new User("xxx","xxx");
    }
}
    
```

这种方式也可以用在 recyclerview adapter 中，adapter 中的代码大大减少

#### Binding Adapter

您可能会好奇配置 `android:text="@{user.firstName}` 后内部发生了什么

DataBinding 中使用 `Binding Adapter` 来处理，它主要处理「属性」和「事件」，前者如  `setText()` ，后者如 `setOnClickListener()`。上面的 `android:text` 实际上调用的是下面的方法

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff6deb8d0c5d406d81c0934a4be0e4d4~tplv-k3u1fbpfcp-zoom-1.image)

DataBinding 中提供了很多 `Binding Adapter`

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/966ddb64425b4e37835df772d474f3e6~tplv-k3u1fbpfcp-zoom-1.image)

如果官方提供的 `Binding Adapter` 不满足您的需求，您还可以自定义 `Binding Adapter`

```java
@BindingAdapter({"imageUrl", "error"})
public static void loadImage(ImageView view, String url, Drawable error) {
  Glide.with(view).load(url).error(error).into(view);
}
    
<ImageView app:imageUrl="@{venue.imageUrl}" app:error="@{@drawable/venueError}" />
    
```

#### DadaBinding + LiveData

要将 LiveData 与 DataBinding 一起使用，需要指定生命周期所有者来定义 LiveData 对象的范围

```java
class ViewModelActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        UserBinding binding = DataBindingUtil.setContentView(this, R.layout.user);

        binding.setLifecycleOwner(this);
    }
}
    
```

#### 双向绑定

使用单向 DataBinding，可以在属性上设置一个值，并设置一个对该属性的更改做出反应的监听器：

```xml
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:checked="@{viewmodel.rememberMe}"
    android:onCheckedChanged="@{viewmodel.rememberMeChanged}"
/>
    
```

使用双向绑定可以简化该过程

```xml
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:checked="@={viewmodel.rememberMe}"
/>
    
```

使用 `@={}` 接收对该属性的数据更改，并同时监听用户更新（注意，这里有 `=` ）

那么究竟什么是双向绑定呢？

所谓的「数据驱动」就是数据驱动视图的变化，而 DataBinding 的单向绑定就是如此。反过来讲，有些时候我们需要视图来驱动数据的变化（例如当我们在 EditText 上输入了文字，我们希望对应的 ViewModel 的 LiveData 的值能够及时响应该变化）

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/771202b967644994a57cfe144687ea55~tplv-k3u1fbpfcp-zoom-1.image)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a7be6740cf764d269d1a15ba6cbb8f07~tplv-k3u1fbpfcp-zoom-1.image)

如图，绿色部分为独立的 fragment ，内部存在两个 TextView，用于显示外部 fragment EditText 输入的文字

如果实现上述功能，传统做法可能是使用 activity 级别的 ViewModel 进行两个 fragment 之间的通信，通过监听 EditText 文字的变化改变 ViewModel 中 LiveData 的值，并在绿色 fragment 中观察 LiveData 并显示到 TextView 中

```kotlin
class NormalViewModel : ViewModel() {
    val firstName = MutableLiveData<String>()
    val lastName = MutableLiveData<String>()
}

class NormalDetailFragment : Fragment(R.layout.fragment_normal_detail) {
    private val mViewModel by activityViewModels<NormalViewModel>()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        mViewModel.firstName.observe(viewLifecycleOwner) {
            tvFirstName.text = it
        }
        mViewModel.lastName.observe(viewLifecycleOwner) {
            tvLastName.text = it
        }
    }
}

class NormalFragment : Fragment(R.layout.fragment_normal) {
    private val mViewModel by activityViewModels<NormalViewModel>()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        etFirstName.addTextChangedListener {
            mViewModel.firstName.value = it.toString()
        }
        etLastName.addTextChangedListener {
            mViewModel.lastName.value = it.toString()
        }
    }
}
    
```

得益于 kotlin ，上面的代码以及很简洁了，如果使用 java 代码片段只会更长。

不过使用 DataBinding，还可以更简洁

```xml
<com.google.android.material.textview.MaterialTextView
    android:id="@+id/tvFirstName"
    android:text="@{vm.firstName}"/>
<com.google.android.material.textview.MaterialTextView
    android:id="@+id/tvLastName"
    android:text="@{vm.lastName}"/>
    
<com.google.android.material.textfield.TextInputEditText
    android:id="@+id/etFirstName"
    android:text="@={vm.firstName}"/>
<com.google.android.material.textfield.TextInputEditText
    android:id="@+id/etLastName"
    android:text="@={vm.lastName}"/>
    
```

只需配置好双向绑定（EditText 驱动 ViewModel 的 LiveData 的值变化，ViewModel 再驱动 TextView 显示数据），并在 fragment 通过固定的模板代码设置好 ViewModel 即可

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a10cdd4ebfc4f258d5339cb795d32a8~tplv-k3u1fbpfcp-zoom-1.image)

这里的魔法还是来自 Binding Adapter

```java
// TextViewBindingAdapter.java
@BindingAdapter(value = {"android:beforeTextChanged", "android:onTextChanged",
        "android:afterTextChanged", "android:textAttrChanged"}, requireAll = false)
public static void setTextWatcher(TextView view, final BeforeTextChanged before,
        final OnTextChanged on, final AfterTextChanged after,
        final InverseBindingListener textAttrChanged) {
    final TextWatcher newValue;
    if (before == null && after == null && on == null && textAttrChanged == null) {
        newValue = null;
    } else {
        newValue = new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {
                if (before != null) {
                    before.beforeTextChanged(s, start, count, after);
                }
            }
            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                if (on != null) {
                    on.onTextChanged(s, start, before, count);
                }
                if (textAttrChanged != null) {
                    //通知发生变化
                    textAttrChanged.onChange();
                }
            }
            @Override
            public void afterTextChanged(Editable s) {
                if (after != null) {
                    after.afterTextChanged(s);
                }
            }
        };
    }
    final TextWatcher oldValue = ListenerUtil.trackListener(view, newValue, R.id.textWatcher);
    if (oldValue != null) {
        view.removeTextChangedListener(oldValue);
    }
    if (newValue != null) {
        view.addTextChangedListener(newValue);
    }
}
    
```

这里使用 InverseBindingListener （调用 `textAttrChanged.onChange()`）来通知 LiveData 数据发生变化

而变化后的值 通过 @InverseBindingAdapter 注解标记的方法处理，这里的 event 与上面的标记匹配（`android:textAttrChanged`）

```java
// TextViewBindingAdapter.java
@InverseBindingAdapter(attribute = "android:text", event = "android:textAttrChanged")
public static String getTextString(TextView view) {
    return view.getText().toString();
}
    
```

view 层变化通知数据变化，数据变化再通知 view 层变化，仿佛是个套娃

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfa61a66ca52476180d0ff8ef8594e3c~tplv-k3u1fbpfcp-zoom-1.image)

因此避免这种死循环十分重要，setText 方法判断了新旧值是否相等来避免死循环

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7b9493ade494b68bc5d1807dab26f2c~tplv-k3u1fbpfcp-zoom-1.image)

### 总结

DataBinding 主要提供两部分功能

- 替换 findViewById ，如果只用这部分功能可以使用 ViewBinding
- 进行 data 和 UI 的绑定，使用「数据驱动」的思想解决了视图的一致性问题

#### 各种 findViewById 替代方案对比

- findViewById
- Butterknife
- Kotlin Synthetics
- Data Binding
- View Binding

##### findViewById

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5efb8141c17143cea8e7650c1fad27fd~tplv-k3u1fbpfcp-zoom-1.image)

findViewById 有两个问题

1. 当不能在 Activity/Fragment/ViewGroup 中定位到指定 id 的 View，会在运行期间崩溃，即非空安全
2. 如果某个 view 为 TextView 类型，而在使用中将其指定为其他类型不会在编译器报错，即非类型安全

在 compileSdk 的 API 级别 26 中，对该方法的定义稍作更改以消除强制类型转换问题

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/417a35b09335435b9511e43e841586bc~tplv-k3u1fbpfcp-zoom-1.image)

现在，开发人员无需在代码中手动转换 view 类型。 如果您引用 id 指向类型 TextView 的 View 并将其指定为 Button，则 Android SDK 会尝试查找具有提供的 id 的 Button，并且它将返回 Null，因为它将无法找到它

但是在 Kotlin 中，您仍然需要提供诸如 findViewById(R.id.txtUsername) 之类的类型。 如果您不检查视图是否具有 null 安全，则可能出现 NullPointerException，但是此方法不会像以前那样抛出ClassCastException

##### Butterknife

[Butterknife](https://github.com/JakeWharton/butterknife) 是 [Jake Wharton](https://medium.com/u/8ddd94878165?source=post_page-----98b8ef5b9249----------------------) 大神写的替代 findViewById 的库，该库使用注解处理并生成 findViewById 代码

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/402c204614f74ad7b6908c300147b3de~tplv-k3u1fbpfcp-zoom-1.image)

它具有与 findViewById 几乎相似的问题。 但是，它在运行时添加了null 安全检查以避免 NullPointerException

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be7b6e6504694e97bb277692a1b8689f~tplv-k3u1fbpfcp-zoom-1.image)

由于 DataBinding 和 ViewBinding 的出现，沃神已经宣布弃用该库

##### Kotlin Synthetics（已弃用）

Kotlin 引入的最大功能之一是 Kotlin 扩展方法。 在它的帮助下，Kotlin Synthetics 诞生了。 Kotlin Synthetics 通过自动生成的 Kotlin 扩展方法，使开发人员可以从 xml 布局直接访问其内部的 view

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb55ee7de840493dae37fde802d05172~tplv-k3u1fbpfcp-zoom-1.image)

Kotlin Synthetics 第一次调用 findViewById 方法，然后默认情况下将 view 实例缓存在 HashMap 中。 可以通过Gradle 设置将此缓存配置更改为 SparseArray 或不缓存

总体而言，Kotlin Synthetics 是一种很好的选择，因为它类型安全，并且通过 Kotlin 的 ？进行空检查。 它不需要开发人员的额外代码。 但这仅适用于 Kotlin 项目

但是，在使用 Kotlin Synthetics 时遇到了一个小问题。 例如，如果将内容视图设置为布局，然后使用仅存在于其他布局中的 id ，则 IDE 可让您自动完成并添加新的 import 语句。 除非您专门检查以确保其 import 语句仅导入正确的 view，否则没有安全的方法来验证这不会导致运行时问题

##### DataBinding

DataBinding 在功能上比其他方法优越得多，因为它不仅为您提供类型安全和空安全的 view 引用，而且还允许您直接在 xml 布局内使用数据驱动视图变化

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ce683368e9d49d3a66872e1ddc35f14~tplv-k3u1fbpfcp-zoom-1.image)

##### ViewBinding

最近在 Android Studio 3.6 中引入的 ViewBinding 是 DataBinding 库的子集。 由于不需要注解处理，因此可以缩短构建时间。详细的使用可以参见 [这篇文章](https://juejin.im/post/5e4806f3e51d4526c550a2ef)

|            | findViewById | Butterknife | Kotlin Synthetics | DataBinding | ViewBinding |
| ---------- | ------------ | ----------- | ----------------- | ----------- | ----------- |
| 一直空安全 | ❌            | 部分        | 部分              | ✔️           | ✔️           |
| 类型安全   | ❌            | ❌           | ✔️                 | ✔️           | ✔️           |
| 样板代码   | 多           | 少          | 少                | 中等        | 少          |
| 构建时间   | ✔️            | ❌           | ✔️                 | ❌           | ✔️           |
| 支持语音   | java/kotlin  | java/kotlin | kotlin            | java/kotlin | java/kotlin |



## 【背上Jetpack之Navigation】想去哪就去哪，Android世界的指南针

### 前言

`androidx Navigation` 组件是 Android 中应用内导航的官方库，当前最新的版本为 `2.3.0-beta01`（2020.05.20）

很多人不喜欢 Navigation 因为其设计不符合开发者的预期，它在管理「平级界面」时来回切换会导致平级的 fragment 重建。网上针对这一问题有一个 [重写 Navigator 的方案](https://github.com/STAR-ZERO/navigation-keep-fragment-sample)，大多数人会简单地认为 Navigation 无法保存 fragment 状态是因为使用了 replace（曾经的我也这样认为）

本文的内容为 Navigation 的职能边界，简单使用，高阶使用技巧（例如同一 activity 部分内部分 fragment 共享 ViewModel，模块化）以及关于 Navigation 所谓的「设计问题」的探讨

对 Navigation 的使用及职能边界已经了解的小伙伴可以直接跳过前两部分

由于文章内容较长，已将模块化部分单独成文，[地址在这](https://juejin.im/post/6844904164166909965)

### 没有 Navigation 的世界

Android 中，activity 和 fragment 是主要的视图控制器，因此界面间的调转也是围绕 activity / fragment 进行的

```
// 跳转 activity
val intent = Intent(this, SecondActivity::class.java)
intent.putExtra("key", "value")
startActivity(intent)


// 跳转 fragment
supportFragmentManager.commit {
    addToBackStack(null)
    val args = Bundle().apply { putString("key", "value") }
    replace<HomeFragment>(R.id.container, args = args)
}
    
```

如果项目比较大，我们可能会将 KEY 抽取为常量，并且在 activity 和 fragment 中填写静态方法以告诉调用者该界面需要什么参数

```
// SecondActivity
companion object {
    @JvmStatic
    fun startSecondActivity(context: Context, param: String) {
        val intent = Intent(context, SecondActivity::class.java)
        intent.putExtra(Constant.KEY, param)
        context.startActivity(intent)
    }
}

// HomeFragment
companion object {
    fun newInstance(param: String) = HomeFragment().apply {
        arguments = Bundle().also {
            it.putString(Constant.KEY, param)
        }
    }
}
    
```

可以看到，得益于 kotlin 的扩展函数，界面间跳转的代码已足够简洁

但是

- 如果在每个界面加上跳转动画呢？
- 当你接手一个较大的项目，如何能快速理清界面间的跳转关系？
- 在单 activity 的项目中，如何控制几个相关的 fragment 有着相同的 ViewModel 作用域，而不是整个 activity 共享？
- 组件间的界面跳转？

### Navigation 简介

![摘自19I/0大会](https://user-gold-cdn.xitu.io/2020/5/21/17237f5ed82ca68e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)摘自19I/0大会

Jetpack 导航组件是一套库，工具和指南，为应用内导航提供了强大的导航框架

#### 它是一套库，封装着应用内导航的 API

![img](https://user-gold-cdn.xitu.io/2020/5/21/17237f5ed9a5fb72?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

引入依赖如下

```
dependencies {
  def nav_version = "2.3.0-beta01"

  // Java language implementation
  implementation "androidx.navigation:navigation-fragment:$nav_version"
  implementation "androidx.navigation:navigation-ui:$nav_version"

  // Kotlin
  implementation "androidx.navigation:navigation-fragment-ktx:$nav_version"
  implementation "androidx.navigation:navigation-ui-ktx:$nav_version"

  // Dynamic Feature Module Support
  implementation "androidx.navigation:navigation-dynamic-features-fragment:$nav_version"

  // Testing Navigation
  androidTestImplementation "androidx.navigation:navigation-testing:$nav_version"
}

    
```

![destination](https://user-gold-cdn.xitu.io/2020/5/21/17237f5edb04c7ba?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)destination

它支持 fragment ，activity，或者是自定义的 destination 间的跳转

![Navigation UI](https://user-gold-cdn.xitu.io/2020/5/21/17237f5edb47adf6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)Navigation UI

Navigation UI 库 支持 Drawer，Toolbar 等 UI 组件

#### 它是一套工具，在 Android Studio 中可以可视化管理界面的导航逻辑

![Android Studio 提供可视化管理的工具](https://user-gold-cdn.xitu.io/2020/5/21/17237f5efff709be?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)Android Studio 提供可视化管理的工具

现在我们对 Navigation 有一个初步的认识，接下来我们看看 Navigation 的职能边界

#### Navigation 能做什么

- 简化界面跳转的配置
- 管理返回栈
- 自动化 fragment transaction
- 类型安全地传递参数
- 管理转场动画
- 简化 deep link
- 集中并且可视化管理导航

#### Navigation 工作逻辑

Navigation 主要有三个部分

- Navigation Graph
- NavHost
- NavController

#### Navigation Graph

`Navigation Graph` 是一种新的 resource type，它是一个集中管理 navigation 的 xml 文件

![Navigation Graph](https://user-gold-cdn.xitu.io/2020/5/21/17237f5edcab0618?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)Navigation Graph

**「Navigation Graph 中的每一个界面叫：Destination」**，它可以使 fragment ，activity，或者自定义的 Destination

Navigation 管理的就是 Destination 间的跳转

点击 Destination，可以在屏幕右侧看见 deep link 等信息的配置

![destination attitude](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f40817110?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)destination attitude

**「Navigation Graph 中的带箭头的线叫：Action」**，它代表着 Destination 间不同的路径

点击 Action，可以在屏幕右侧看到 Action 的详细配置，动画，Destination 间跳转传递的参数，操作返回栈，Launch Options

![action attributes](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f08015a37?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)action attributes

> ❝
>
> 不知道各位小伙伴大学是否学过 **「图论」**，个人感觉 Navigation Graph 就像 **「有向图」**，而其中的 Destination 和 Action 就像图论中的 **「点」** 和 **「边」**
>
> ❞

#### NavHost

`NavHost` 是一个空容器，用于显示 `navigation graph` 中的 destination。 导航组件提供一个默认的 `NavHost` 实现 `NavHostFragment`，它显示 fragment destination

NavHostFragment 是 navigation-fragment 中的类

![NavHostFragment](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f0a0e76b8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)NavHostFragment 

它提供了一个可独立导航的区域，使用时大概是这样

![NavHostFragment 使用](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f0a0b3698?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)NavHostFragment 使用

![img](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f2547be56?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

所有的 fragment Destination 都是通过 NavHostFragment 管理，相当于一个容器

每个 NavHostFragment 都有一个 NavController，用于定义 navigation host 中的导航。 它还包括 [navigation graph](https://developer.android.com/reference/androidx/navigation/NavGraph) 以及 navigation 状态（例如当前位置和返回栈），它们将与 NavHostFragment 本身一起保存和恢复

#### NavController

`NavController` 帮助 `NavHost` 管理导航，其内部持有 `NavGraph` 和 `Navigator`（通过持有 `NavigatorProvider` 间接持有）

其中 [NavGraph](https://developer.android.com/reference/androidx/navigation/NavGraph) 决定了界面间的跳转逻辑，它通常在 Android resource 下创建，同时也支持通过代码动态创建

[Navigator](https://developer.android.com/reference/androidx/navigation/Navigator) 定义了一在应用内导航的机制。它的实现类有 [ActivityNavigator](https://developer.android.com/reference/androidx/navigation/ActivityNavigator), [DialogFragmentNavigator](https://developer.android.com/reference/androidx/navigation/fragment/DialogFragmentNavigator), [FragmentNavigator](https://developer.android.com/reference/androidx/navigation/fragment/FragmentNavigator), [NavGraphNavigator](https://developer.android.com/reference/androidx/navigation/NavGraphNavigator)。当然，开发者也可以自定义 `Navigator`。每种 `Navigator` 都有自己的导航策略，例如 `ActivityNavigator` 使用 `starActivity` 来进行导航

#### 总结

下面引用一张 [KunMinX](https://xiaozhuanlan.com/u/kunminx) 的专栏 [重学 Android Navigation 一文中的配图](https://xiaozhuanlan.com/topic/5860149732)，帮助大家理解这其中的依赖关系

![img](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f2d2a2a2d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> ❝
>
> 来自 [重学安卓：就算不用 Jetpack Navigation，也请务必领略的声明式编程之美！ ](https://xiaozhuanlan.com/topic/5860149732)
>
> ❞

我们在 res/navigatoin 创建的 xml 文件叫 `Navigation Graph` (类似图论中的图)

其内部每个节点叫 `Destination`(类似图论中的点) ，它对应着 activity/fragment/dialog，代表着屏幕上的界面

连接 `Destination` 与 `Destination` 之间的线叫 `Action`(类似图论中的边)，它是从一个界面跳转另个一个界面的抽象，可以配置跳转动画，传递参数，以及返回栈等信息

```
NavHost` 是显示 `Navigation Graph` 的容器，实现类为 `NavHostFragment`，每个 `NavHostFragment` 中都持有一个 `NavController
```

`NavController` 是导航的大管家，封装着 `navigate` `navigateUp` `popBackStack` 等方法

`Navigator` 是对 `Destination` 之间跳转的封装。由于 `Destination` 可以是 activity 或 fragment，因此有了 `ActivityNavigator` `FragmentNavigator` 等实现类，用于实现具体的界面跳转

#### Navigation 的使用技巧

#### Dialog Destination

Navigation 2.1.0 引入，用于实现 navigate 到一个 DialogFragment

使用也很简单，使用 dialog 标签，其他处理的与 fragment 相同

![dialog destination](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f30105cb3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)dialog destination

#### 同一 graph 中共享 ViewModel

我们都知道 fragment 可以使用 activity 级别共享 ViewModel，但是对于单 activity 项目，这就意味着所有的 fragment 都能拿到这个共享的 ViewModel。本着最少知道原则，这不是一个好的设计

![想要在部分fragment中共享ViewModel](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f313ed2fe?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)想要在部分fragment中共享ViewModel

Navigation 2.1.0，官方引入了 navigation graph 内共享的 ViewModel，这使得 ViewModel 的作用域得到了细化，业务之间可以很好地被隔离

使用起来非常简单

```
// kotlin
val viewModel: MyViewModel by navGraphViewModels(R.id.my_graph)
    
// java
NavBackStackEntry backStackEntry = navController.getBackStackEntry(R.id.my_graph);
MyViewModel viewModel = new ViewModelProvider(backStackEntry).get(MyViewModel.class);
    
```

#### Safe Args

![什么是 Safe Args](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f499f8aa8?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)什么是 Safe Args

什么是 Safe Args？

它是一个 Gradle 插件，可以根据 navigation 文件生成代码，帮助开发者安全地在 `Destination` 之间传递数据

那么为什么要设计这样一个插件呢？

我们知道使用 bundle 或者 intent 传递数据时，如果出现类型不匹配或者其他异常，是在 Runtime 中才能发现的，而 `Safe Args` 把校验转移到了编译期

![为什么设计Safe Args](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f5272e9f7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)为什么设计Safe Args

使用 `Safe Args` 需要手动引入插件

```
buildscript {
    repositories {
        google()
    }
    dependencies {
        def nav_version = "2.3.0-alpha06"
        classpath "androidx.navigation:navigation-safe-args-gradle-plugin:$nav_version"
    }
}
    
```

前面我们提到 `Safe Args` 会生成代码

如果要生成 java 代码，则在 app 或其他 module 的 build.gradle 中加入

```
apply plugin: "androidx.navigation.safeargs"
    
```

如果向生产 kotlin 代码，则加入

```
apply plugin: "androidx.navigation.safeargs.kotlin"
    
```

参数是在 action 中配置的，我们只需在加入 argument 标签

```
<action android:id="@+id/startMyFragment"
    app:destination="@+id/myFragment">
    <argument
        android:name="myArg"
        app:argType="integer"
        android:defaultValue="1" />
</action>
    
```

Navigation 支持以下类型

![支持的参数类型](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f5652a8f1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)支持的参数类型

启用 Safe Args 后，生成的代码将为每个操作以及每个发送和接收 destination 创建以下类型安全的类和方法

- 为拥有 action 的发送 destination 的创建一个类，该类的类名为 destination 名 + Directions。例如我们的发送 destination 为 SpecifyAmountFragment，那么将会生成 SpecifyAmountFragmentDirections 类。该类会为 destination 的每个 action 创建一个方法
- 为每个传递参数的 action 创建一个内部类，如果 action 叫 confirmationAction 则会创建 ConfirmationAction 类。如果 action 的参数没有默认值，则要求开发者使用该类设置参数
- 为接收 destination 创建一个类，该类的类名为 destination 名 + Args。例如我们的接收 destination 为 ConfirmationFragment，那么将会生成 ConfirmationFragmentArgs 类。使用该类的 `fromBundle()` 方法可以取出从发送 destination 传来的参数

下面的代码展示如何在发送 destination 传递参数

```
override fun onClick(v: View) {
   val amountTv: EditText = view!!.findViewById(R.id.editTextAmount)
   val amount = amountTv.text.toString().toInt()
   // 在相应的 action 中配置参数
   val action = SpecifyAmountFragmentDirections.confirmationAction(amount)
   v.findNavController().navigate(action)
}
    
```

接下来展示如何在接收 destination 中取出传来的参数，kotlin 可以使用 `by navArgs()` 获取参数

```
// kotlin
val args: ConfirmationFragmentArgs by navArgs()

override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    val tv: TextView = view.findViewById(R.id.textViewAmount)
    val amount = args.amount
    tv.text = amount.toString()
}
    
// java
@Override
public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    TextView tv = view.findViewById(R.id.textViewAmount);
    int amount = ConfirmationFragmentArgs.fromBundle(getArguments()).getAmount();
    tv.setText(amount + "")
}
    
```

#### 嵌套 navigation graph

有一些 destination 通常是组合使用，并且在多个地方重复调用。例如独立的登录流程，后续的忘记密码，修改密码等 destination 可以看做一个整体来使用

这种情况我们使用嵌套 navigation graph，选中一组可以作为整体的 destination (按住 shift 鼠标点选)，然后右击选中 **「Move to Nested Graph」** > **「New Graph」** 。这样就生成了 嵌套 navigation graph。在 嵌套 navigation graph 上双击即可查看内部的 destination

![创建嵌套navigation graph](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f570a965c?imageslim)创建嵌套navigation graph

**「如果想要引用其他 module 中的 graph，可以使用 `include` 标签」**

![include 使用](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f6f71c7f4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)include 使用

![include 使用](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f7709b92b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)include 使用

#### 全局 action

您可以为多个 destination 创建共用的 action，例如您可能想要在不同的 destination 中导航至相同的界面

对于这种情况您可以使用全局 action

选中一个 destination 并右击，选择 **「Add Action > Global」** ，一个箭头会 出现在 destination 的左边

![全局 action](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f798420dc?imageslim)全局 action

使用也很简单，向 navigate 方法传入全局 action 的资源 id 即可

```
viewTransactionButton.setOnClickListener { view ->
    view.findNavController().navigate(R.id.action_global_mainFragment)
}
    
```

#### 条件导航

在开发过程中，我们可能会遇到一个 destination 根据条件跳转不同的 destination 的情况

例如一些 destination 需要用户处于登录状态才能进入，或者在游戏结束后，胜利和失败跳转不同的 destination

下面我们用一个示例来展示 Navigation 如何处理该种场景

该示例中，用户尝试跳转到资料页中，如果该用户处于未登录状态，则需要跳转到登录界面

![graph](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f83ccf4fe?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)graph

我们使用 LoginViewModel 来保存登录状态，从 ProfileFragment 点击按钮跳转到 ProfileDetailFragment，在该界面判断登录的状态，如果是未授权则跳转到 LoginFragment，如果已授权则提示欢迎

![ProfileDetailFragment](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f86cec299?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)ProfileDetailFragment

在登录界面判断登录状态，如果授权成功则回到 ProfileDetailFragment，授权失败则显示登录失败提示

在登录界面点击返回视为未授权，应该直接返回 ProfileFragment 界面

![LoginFragment](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f98ab90fa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)LoginFragment

#### Deep Links

开发过程中我们可能会遇到这类的需求，我们需要让用户打开 app 时直接空降到某个特定页面（例如点开通知栏跳转到特定文章），亦或者我们需要从一个 destination 跳转到一个在其他流程中比较深的位置的 destination ，如下图，从 FriendList 跳转到 Chat 界面

![img](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f9dd0b4bb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

上面的这种需求叫作 `deep link` ，Navigation 支持两种 `deep link` 的跳转。

#### 显式 deep link

在 manifest activity -> intent-filter 标签下 加入 action ，category ，data 等标签，满足条件的 intent 可以被打开

详情见 [官方文档](https://developer.android.com/training/app-links/deep-linking)，这里不再赘述，我们只关注如何通过 Navigation 构建 intent

```
val pendingIntent = NavDeepLinkBuilder(context)
    .setGraph(R.navigation.nav_graph)
    .setDestination(R.id.android)
    .setArguments(args)
    .createPendingIntent()
    
```

如果已经存在 NavController，可以通过 NavController.createDeepLink() 方法创建 deep link

#### 隐式 deep link

在 navigation graph 中支持 deepLink 标签

例如上图从 FriendListFragment 跳转到 ChatFragment，可以在 graph ChatFragment 下加入 deepLink 标签

```
<fragment
    android:id="@+id/chatFragment"
    android:name="com.flywith24.bottomtest.ChatFragment
    android:label="ChatFragment">
    <argument
        android:name="userId"
        app:argType="string" />
    <deepLink
        android:id="@+id/deepLink"
        app:uri="chat://convention/{userId}" />
</fragment>
    
```

`NavController#navigate()` 方法支持传入 URI

![navigate to deep link](https://user-gold-cdn.xitu.io/2020/5/21/17237f5f9b8c952e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)navigate to deep link

因此可直接调用

```
val userId = "1111"
findNavController().navigate("chat://convention/$userId".toUri())
    
```

#### 模块化

参见 [【奇技淫巧】使用 Navigation + Dynamic Feature Module 实现模块化](https://juejin.im/post/6844904164166909965)

#### Navigation 设计探讨

#### fragment replace 你真的了解吗

androidx 下 frament replace 的行为你真的了解吗？

该部分内容我们在 [【背上 Jetpack】绝不丢失的状态 androidx SaveState ViewModel-SaveState 分析](https://juejin.im/post/6844904097351467015#heading-9) 和[【背上 Jetpack 之 Fragment】从源码角度看 Fragment 生命周期 AndroidX Fragment1.2.2 源码分析](https://juejin.im/post/6844904086437904398) 已有分析

因此我们直接说结论

fragment replace 后 **「前一个 fragment 会执行 onDestroyView 而不执行 onDestroy」** ，即 fragment 本身未销毁，其内部 view 被销毁

> ❝
>
> FragmentManager 的 moveToState 方法在触发 fragment 的 onDestroyView 前根据条件会执行 fragmentStateManager.saveViewState() 方法来保存 view 状态（1.2.2，旧版本该处方法方法名略有不同）
>
> 而由于 fragment 本身没有销毁，其成员也不会被销毁
>
> 所以当返回后 view 的状态会被恢复，而成员状态没有改变，所以 replace 后 fragment 能恢复到之前的状态
>
> ❞

那么 Navigation 的所谓 「设计问题」是怎么回事？

#### 被重建的 fragment

我们通过 Android Studio 创建项目，模版选择 Bottom Navigation Activity，可以得到使用 Navigation 实现的 平级 tab 切换模版。我们使用从 HomeFragment 切换至 DashboardFragment，之后切换回 HomeFragment。日志如下

![使用navigation切换fragment](https://user-gold-cdn.xitu.io/2020/5/21/17237f5faa1bd726?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)使用navigation切换fragment

可以看到 HomeFragment 被重建了，**「原实例（e6c266），新实例（c3e49cc）」**

**「fragment 被重建，这就是原因所在！」**

那么为什么出现这种现象？我们翻一下源码

![navigate方法](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fad32bf02?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)navigate方法

![通过反射创建新的fragment实例](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fc0720836?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)通过反射创建新的fragment实例

从源码可以看到，其内部通过反射创建了新的 fragment 实例，这导致 fragment 内部的状态无法恢复

不过如果 navigation 导航的所有 destination 没有平级关系，换句话说在一个返回栈内，这样的设计是没有问题的

但是有些时候我们希望使用 navigation 管理一些平级界面，例如 BottomNavigation

#### 不符合 Material Design 的 BottomNavigation

issuetracker 有这样一个 issue，注意提出的时间

![issue](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fbd33d492?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)issue

主要意思就是现阶段的 bottom tab navigation 不符合 Material Design 的规范

- 标签间要保存滚动位置
- 每个标签应该有自己独立的返回栈

相同类型的 issue 还有这些

![相同类型的issue](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fcb8fccc5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)相同类型的issue

Ian Lake 在该条 issue 下给出了详细的解答

我在这里简单介绍一下 Ian Lake，虽然不知道他的职位，但从他的活跃程度看应该是 fragment 和 navigation 的负责人，在 Google I/O 大会 和 Android Dev Summit 多次进行演讲，例如 fragment 的过去，现在和将来，单 activity 项目

![官方解答](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fd413eeea?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)官方解答

**「一句话解释：单个 FragmentManager 不支持多个返回栈，以现有的 fragment API 无法做到这一支持，开发者不得不自己实现多返回栈（例如我在 [【背上 Jetpack 之 Fragment】从源码的角度看 Fragment 返回栈 附多返回栈 demo](https://juejin.im/post/6844904090921779214) 一文中提供的 demo）」**

但他提供了短期和中期方案

- 短期：提供一个公开示例，展示使用当前 API 进行多返回栈的实现（即，每个底部导航项都有一个单独的 NavHostFragment 和 navigation graph）。[其 demo 在这](https://github.com/android/architecture-components-samples/tree/master/NavigationAdvancedSample) ，核心逻辑在 `NavigationExtensions.kt` 内
- 中期：在 Fragment 上构建正确的 API，以便它们可以正确地支持多个返回栈，包括为所有返回栈上的所有 Fragment 正确保存和恢复已保存的实例状态和非配置实例状态。 这项工作目前处于探索阶段，尽管我希望，但我无法提供时间表或保证这项工作将会成功

以上回复发生在 2019 年 2 月

在 2019 年 10 月 Ian Lake 再次回复了开发者的疑问

![官方补充1](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fd7498d5c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)官方补充1

![官方补充2](https://user-gold-cdn.xitu.io/2020/5/21/17237f5fe3c789b6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)官方补充2

多返回栈支持计划需要三步走

- 为 fragment 提供相应的 API 以保证多返回栈的 fragment 状态能够被保存
- NavController API 提供了通用框架，该框架允许任何 Navigator（括 FragmentNavigator）支持多返回栈
- NavigationUI 中提供的新 API，可让您在使用 BottomNavigationView 或 NavigationView 时 无论是单返回栈（当前模式）还是多返回栈都能控制 setupWithNavController()

之后他回复了 70 楼的开发者，明确了如果使用单个 NavHostFragment / navigation graph / FragmentManager 能够支持多返回栈，那么从 A 或 B 导航到 C 就不会有任何问题

接着时间来到了 2020 年

![2020年的最新回复](https://user-gold-cdn.xitu.io/2020/5/21/17237f637e33684a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)2020年的最新回复

今年 1 月 30 日，Ian Lake 再次做了回复

大概意思就是原定 `Fragment 1.3.0` 和 `Navigation 2.3.0` 提供多返回栈支持的计划跳水了，计划在 `Fragment 1.4.0-alpah01` 和 `Navigation 2.4.0-alpha01` 提供

至此关于 Navigation 所谓的 「设计问题」就探讨结束了



## 【背上Jetpack之LiveData】ViewModel 的左膀右臂 数据驱动真的香

![目录](https://user-gold-cdn.xitu.io/2020/3/31/17130eb48b4574e3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 前言

> 之前我们讨论过 [ViewModel 的职能边界](https://juejin.im/post/6844904100493017095) ，得益于 ViewModel 的生命周期更长，我们可以在 activity 重建后将数据传递给 activity ，也可以避免内存泄漏。但是如果不是每次需要就获取数据，而是当每次有新数据时通知我们，应该怎么办？

本文介绍 `LiveData` ，一个 **生命周期感知的，可观察的，数据持有者**。同时还会简单分析 `LiveData` 的源码实现

### 我们都是 Adapter

在谈 `LiveData` 前我们来思考一个问题

**Android 开发（亦或者说前端开发）的本质工作内容是什么？**

对于应用层 app 开发者，开发者的工作主要工作就是 Adapter

什么是 Adapter ，下图可能比较直观



![Adapter](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb508a1393?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 图片来自 google image

我们的工作本质是 **将数据转换成 UI**

数据可能来自网络，来自本地数据库，来自内存，而 UI 可能是 activity 或 fragment。

### 理想的数据模型

上面我们提到 Android 开发者的核心工作就是将数据转换为 UI 。这个过程比较理想的状态是：当数据发生变化时，UI 跟随变化。我们还可以进一步展开：当 UI 对用户可见时，数据发生变化时 UI 跟随变化；当 UI 对用户不可见时，我们希望数据变化时什么都不做，当 UI 再次对用户可见时根据最新的数据进行 UI 的处理。

而 `LiveData` 就是我们理想中的数据模型



![LiveData](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb508eebda?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 图片来自 [Android Dev Summit '18-Fun with LiveData](https://www.youtube.com/watch?v=2rO4r-JOQtA&list=PLWz5rJ2EKKc_dskHzXdKHB2ZvAlMyFwZe&index=2&t=0s)

LiveData 可以三个关键词概括

- lifecycle-aware
- observable
- data holder

#### observable

Android 中不同的组件有着不同的生命周期，不同的存活时间



![ViewModel](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb50a0423c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



因此我们不会在 `ViewModel` 中持有 `Activity` 的引用，因为这会导致当 `Activity` 重建时内存泄漏，甚至出现空指针的情况



![observable](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb50a44353?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



通常我们会在 `Activity` 中持有 `ViewModel` 的引用，那么如何进行二者间的通信，如何向 `Activity` 发送 `ViewModel` 中的数据？

答案是让 `Activity` 观察 `ViewModel`



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb520bea5d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```
LiveData` 是 `observable
```

#### lifecycle-aware

当观察者观察着某个数据时，该数据必须保留对观察者的引用才能调用它，为了解决这个问题，`LiveData` 被设计成可感知生命周期



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb52097caa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



当 activity / fragment 被销毁后，它会自动的取消订阅

#### data holder

`LiveData` 仅持有 **单个且最新** 的数据



![data holder](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb7f1ec390?imageslim)



上图中，最右侧是在 `ViewModel` 中的 `LiveData`，左侧为观察这个 `LiveData` 的 activity / fragment 。一旦我们为 `LiveData` 设值，该值会传递到 activity。简而言之，`LiveData` 值改变，activity 收到最新的值的变化。但是当观察者不再处于活动状态（STARTED 到 RESUMED ），数据 C 不会被发送到 activity 。当 activity 回到前台，它将收到最新的值，数据 D。**LiveData 仅持有单个且最新的数据**。当 activity 执行销毁流程时，此时的数据 E 也不会产生任何影响

#### Transformations

```
LiveData` 提供 两种 transformation ，`map` 和 `switch map`。开发者也可以创建自定义的  `MediatorLiveData
```

我们都知道 `LiveData` 可以为 View 和 ViewModel 提供通信，但如果有一个第三方组件（例如 repository ）也持有 `LiveData`。那么它应该如何在 `ViewModel` 中订阅？该组件并没有 lifecycle



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb7f78ae88?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



一旦我们的应用愈发复杂，repository 可能会观察数据源



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb812c5b97?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



那么 view 如何获取 repository 中的 `LiveData`？

#### 一对一的静态转换（map）



![one-to-one static transformation](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb843cf0da?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



在上面的示例中，`ViewModel` 仅将数据从 repository 转发到 view，然后将其转换为 UI Model。 每当 repository 中有新数据时，`ViewModel` 只需 `map`

```
class MainViewModel {
  val viewModelResult = Transformations.map(repository.getDataForUser()) { data ->
     convertDataToMainUIModel(data)
  }
}
    
```

第一个参数为 `LiveData` 源（来自 repository ），第二个参数是一个转换函数。

```
// 这里的转换为将 X 转换为 Y
inline fun <X, Y> LiveData<X>.map(crossinline transform: (X) -> Y): LiveData<Y> =
        Transformations.map(this) { transform(it) }
    
```

#### 一对一的动态转换（switchMap）

假如您正在观察一个提供用户的用户管理器，并且需要提供用户的 id 才能开始观察 repository



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb8bdadc5b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



您不能将其写到 `ViewModel` 初始化的过程中，因为此时用户的 id 还不可用

这时 `switchMap` 就派上用场了

```
class MainViewModel {
  val repositoryResult = Transformations.switchMap(userManager.userId) { userId ->
     repository.getDataForUser(userId)
  }
}
    
```

`switchMap` 在内部使用 `MediatorLiveData`，因此了解它非常重要，因为当您要组合多个 `LiveData` 源时需要使用它

```
// 这里的转换为将 X 转换为 LiveData<Y>
inline fun <X, Y> LiveData<X>.switchMap(
    crossinline transform: (X) -> LiveData<Y>
): LiveData<Y> = Transformations.switchMap(this) { transform(it) }
    
```

#### 一对多依赖（MediatorLiveData）

`MediatorLiveData` 允许您将一个或多个数据源添加到单个可观察的 `LiveData` 中

```
val liveData1: LiveData<Int> = ...
val liveData2: LiveData<Int> = ...

val result = MediatorLiveData<Int>()

result.addSource(liveData1) { value ->
    result.setValue(value)
}
result.addSource(liveData2) { value ->
    result.setValue(value)
}
    
```

在上面的例子中，当任何一个数据源变化时，result 会更新。

> 注意：数据并不是合并，MediatorLiveData 只是处理通知

为了实现示例中的转换，我们需要将两个不同的 LiveData 组合为一个



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfb8e8098a1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 图片来自 [LiveData beyond the ViewModel — Reactive patterns using Transformations and MediatorLiveData](https://medium.com/androiddevelopers/livedata-beyond-the-viewmodel-reactive-patterns-using-transformations-and-mediatorlivedata-fda520ba00b7)

使用 `MediatorLiveData` 合并数据的一种方法是添加源并以其他方法设置值：

```
fun blogpostBoilerplateExample(newUser: String): LiveData<UserDataResult> {

    val liveData1 = userOnlineDataSource.getOnlineTime(newUser)
    val liveData2 = userCheckinsDataSource.getCheckins(newUser)

    val result = MediatorLiveData<UserDataResult>()

    result.addSource(liveData1) { value ->
        result.value = combineLatestData(liveData1, liveData2)
    }
    result.addSource(liveData2) { value ->
        result.value = combineLatestData(liveData1, liveData2)
    }
    return result
}
    
```

数据的实际组合是在 `combineLatestData` 方法中完成的

```
private fun combineLatestData(
        onlineTimeResult: LiveData<Long>,
        checkinsResult: LiveData<CheckinsResult>
): UserDataResult {

    val onlineTime = onlineTimeResult.value
    val checkins = checkinsResult.value

    // Don't send a success until we have both results
    if (onlineTime == null || checkins == null) {
        return UserDataLoading()
    }

    // TODO: Check for errors and return UserDataError if any.

    return UserDataSuccess(timeOnline = onlineTime, checkins = checkins)
}
    
```

检查值是否准备好并发出结果（加载中，失败或成功）

### LiveData 的错误用法

#### 错误地使用 var LiveData

```
var lateinit randomNumber: LiveData<Int>

fun onGetNumber() {
   randomNumber = Transformations.map(numberGenerator.getNumber()) {
       it
   }
}
    
```

这里有一个重要的问题需要理解：转换会在调用时（`map` 和 `switchMap`）会创建一个新的 `LiveData`。 在此示例中，randomNumber 公开给 View ，但是每次用户单击按钮时都会对其进行重新赋值。 观察者只会在订阅时收到分配给 var 的 `LiveData` 更新的信息

```
// 只会收到第一次分配的值
viewmodel.randomNumber.observe(this, Observer { number ->
    numberTv.text = resources.getString(R.string.random_text, number)
})
    
```

如果 viewmodel.randomNumber `LiveData` 实例发生更改，这里永远不会回调。而且这里泄漏了之前的 `LiveData` ，这些 `LiveData` 不会再发送更新



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfbab121d0e?imageslim)



一言以蔽之，**不要在 var 中使用 Livedata**

正确示例见  [demo](https://github.com/Flywith24/Flywith24-Jetpack-Demo/tree/master/demo_livedata)

#### LiveData 粘性事件

一般来说我们使用 LiveData 持有 UI 数据和状态，但是如果通过它来发送事件，可能会出现一些问题。这些问题及解决方案 [在这](https://juejin.im/post/6844903623252508685)

#### fragment 中错误地传入 LifecycleOwner

`androidx fragment 1.2.0` 起，添加了新的 Lint 检查，以确保您在从 onCreateView()、onViewCreated() 或 onActivityCreated() 观察 `LiveData` 时使用 getViewLifecycleOwner()



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfbacfb20d4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)





![bug](https://user-gold-cdn.xitu.io/2020/3/31/17130dfbb69c3c1e?imageslim)



如图，我们有一个 fragment ，onCreate 观察 `LiveData`，通过正常的生命周期创建了 View ，接着进入了 resume 状态。此时你使用了 `LiveData`，UI 将开始展示它。之后，用户点击了按钮，由于跳转了另一个 fragment，所以要 detach 该 fragment，一旦 fragment stop 我们就不需要其中的 view 了，因此 destroyView 。之后用户点击了返回按钮回到了上一个 fragment，由于我们已经 destroyView，因此我们需要创建一个新的 view ，接着进入正常的生命周期，但此时，出现了一个 bug 。这个新 View 不会恢复 `LiveData` 的状态，因为我们使用的是 fragment 的 lifecycle observe 的 `LiveData`

我们有两种选择，在 onCreate 或者在 onCreateView 中使用 fragment 的 lifecycle observe LiveData



![img](https://user-gold-cdn.xitu.io/2020/3/31/17130dfbb8fedbd0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



前者的优点是一次注册，缺点是当 recreate 时有bug；后者优点是能够解决 recreate 的 bug，但会导致重复注册

**该问题的核心是 fragment 拥有两个生命周期：fragment 自身和 fragment 内部 view 的生命周期**

`androidx fragment 1.0` 和 `support library 28` 了 viewLifecycle

**因此，当需要观察 view 相关的 LiveData ，可以在 onCreateView()、onViewCreated() 或 onActivityCreated()  中 LiveData observe 方法中传入 viewLifecycleOwner 而不是传入 this**

### 源码结构

首先来看 `LiveData` 主要的源码结构

- LiveData
- MutableLiveData
- Observer

#### LiveData

`LiveData` 是可以在给定生命周期内观察到的数据持有者类。 这意味着可以将一个`Observer` 与 `LifecycleOwner` 成对添加，并且只有在配对的 `LifecycleOwner` 处于活动状态时，才会向该观察者通知有关包装数据的修改。 如果 LifecycleOwner 的状态为 `Lifecycle.State.STARTED` 或 `Lifecycle.State.RESUMED`，则将其视为活动状态。 通过 `observeForever`（Observer）添加的观察者被视为始终处于活动状态，因此将始终收到有关修改的通知。 对于那些观察者，需要手动调用 `removeObserver`（Observer）

如果相应的生命周期移至 `Lifecycle.State.DESTROYED` 状态，则添加了生命周期的观察者将被自动删除。 这对于 activity 和 fragment 可以安全地观察 `LiveData` 而不用担心泄漏

此外，`LiveData` 具有 onActive() 和 onInactive() 方法，以便在活动观察者的数量在 0 到 1 之间变化时得到通知。这使 `LiveData` 在没有任何活动观察者的情况下可以释放大量资源。

主要方法有：

- T getValue() 获取LiveData 包装的数据
- observe(LifecycleOwner owner, Observer<? super T> observer) 设置观察者（主线程调用）
- setValue(T value)  设值（主线程调用），可见性为 protected 无法直接使用
- postValue(T value) 设置（其他线程调用），可见性为 protected 无法直接使用

#### MutableLiveData

`LiveData` 实现类，公开了 `setValue` 和 `postValue` 方法

#### Observer

接口，内部只有 onChanged(T t) 方法，在数据变化时该方法会被调用

### 源码分析

我们通过源码来看看 `LiveData` 如何实现它的特性的

- 1. 如何控制在 activity 或 fragment 活动状态时接收回调，否则不接收？
- 1. 如何在 activity 或 fragment 销毁时自动取消注册观察者？
- 1. 如何保证 `LiveData` 持有最新的数据？

我们查看 `LiveData` 的 observe 方法

```
// LiveData.java
@MainThread
public void observe(LifecycleOwner owner, Observer<? super T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // 如果 owner 已经是 DESTROYED 状态，则忽略
        return;
    }
    // 使用 LifecycleBoundObserver 包装 owner 和 observer
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    // 如果已经添加过直接 return 
    if (existing != null) {
        return;
    }
   
    owner.getLifecycle().addObserver(wrapper);
}

// LifecycleBoundObserver.java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;
    LifecycleBoundObserver(LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }
}
    
```

通过源码我们知道，当我们调用 observe 方法时，内部是通过 `LifecycleBoundObserver` 将 owner 和 observer 包裹起来并通过 `addObserver` 方法添加观察者的，因而当数据变化时，会调用 `LifecycleBoundObserver` 的 `onStateChanged` 方法

```
// LiveData.LifecycleBoundObserver.java
@Override
public void onStateChanged(@NonNull LifecycleOwner source,
        @NonNull Lifecycle.Event event) {
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
        // 自动移除观察者，问题 2 得到解释
        removeObserver(mObserver);
        return;
    }
    activeStateChanged(shouldBeActive());
}
    
```

当生命周期所有者处于 `DESTROYED` 状态时，会调用 `removeObserver` 方法，因此问题 2  得到解释

我们继续向下看，`activeStateChanged` 方法调用时传入了 `shouldBeActive()`

```
@Override
boolean shouldBeActive() {
    // 至少是 STARTED 状态 返回 true
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}

void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        // 与上次值相同，则直接 return （两次均为活动状态或均为非活动状态）
        return;
    }
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    // 根据 mActive 修改活动状态观察者的数量（加 1 或减 1 ）
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        onActive();
    }
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
    }
    if (mActive) {
        // 如果是活动状态，则发送数据，问题 1 得到解释
        dispatchingValue(this);
    }
}
    
```

这里牵扯了 `Lifecycle State` 比较的知识，[详情在这](https://juejin.im/post/6844904111108800519#heading-11)

只有 `STARTED` 和 `RESUMED` 状态 `shouldBeActive()` 才返回 true，至此问题 1 得到解释

`dispatchingValue` 方法内部调用了 `considerNotify` 方法

```
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // 再次判断生命周期所有者状态
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    // 比较版本号
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    // 调用我们传入的 mObserver 的 onChanged 方法
    observer.mObserver.onChanged((T) mData);
}
    
```

可以看到 `considerNotify` 中比较了 observer 的版本号，如果是最新的数据，直接 return

而 `mVersion` 在 `setValue` 方法中 进行更改

```
@MainThread
protected void setValue(T value) {
    // 每次设置对 mVersion 进行++
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
    
```

因此 `LiveData` 每次都持有最新的数据，问题 3 得到解释

### 总结

回到本文开头的思考，Android 开发者的主要工作是将数据转换成 UI ，而 `LiveData` 本质上是一种「数据驱动」，即通过改变状态数据，来驱动视图树中绑定了相应状态数据的控件重新发生绘制。Flutter 和未来的  Jetpack Compose 采用的都是这种机制。使用 ViewModel + LiveData，可以 **安全地在订阅者的生命周期内分发正确的数据**，使开发者不知不觉中完成了 `UI -> ViewModel -> Data` 的单向依赖。

**所谓架构，很多时候不是使用它能做什么，更多的是不要做什么，使用它时开发者能够得到约束，以便产出更健壮的代码**






