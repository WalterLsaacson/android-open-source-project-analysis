# Android组件管理框架：Android视图容器Activity

**关于作者**

>郭孝星，程序员，吉他手，主要从事Android平台基础架构方面的工作，欢迎交流技术方面的问题，可以去我的[Github](https://github.com/guoxiaoxing)提issue或者发邮件至guoxiaoxingse@163.com与我交流。

第一次阅览本系列文章，请参见[导读](https://github.com/guoxiaoxing/android-open-source-project-analysis/blob/master/doc/导读.md)，更多文章请参见[文章目录](https://github.com/guoxiaoxing/android-open-source-project-analysis/blob/master/README.md)。

**文章目录**

- 一 Activity的启动流程
- 二 Activity的回退栈
- 三 Activity的生命周期
- 四 Activity的启动模式
- 五 Activity的通信方式

本篇文章我们来分析Android的视图容器Activity，Android源码分析系列的文章终于写到了Activity，这个我们最常用，源码也最复杂的一个组件，之前在网上看到过很多关于Activity源码
分析的文章，这些文章写得都挺好，它们往往是从Activity启动流程这个角度出发，一个一个函数的去分析整个流程。但是这种做法会让文章通篇看去全是源码，而且会让读者产生一个疑问：这么
长的流程，我该如何掌握，掌握了之后有有什么意义吗？🤔

事实上，单纯去分析流程，确实看不出有什么实践意义，因此我们最好能带着日常开发遇到的问题去看源码，例如特殊场景下的生命周期是怎么变化的，为什么会出现ANR，不同启动模式下对Activity
入栈出栈有何影响等。我们带着问题去看看源码里是怎么写的，这样更有目的性，不至于迷失在茫茫多的源码中。

好了，闲话不多说，我们开始吧。😁

Activity作为Android最为常用的组件，它的复杂程度是不言而喻的。当我们点击一个应用的图标，应用的LancherActivity（MainActivity）开始启动，启动请求以一种IPC的方式传入AMS，AMS开始
处理启动请求，伴随着Intent与Flag的解析和Activity栈的进出，Activity的生命周期从onCreate()方法开始变化，最终将界面呈现在用户的面前。

Activity的复杂性主要体现在两个方面：

- 复杂的启动流程，超长的函数调用链。
- Activity栈的管理。
- 启动模式、Flag以及各种场景对Activity生命周期的影响。

针对这些问题，我们来一一分析。

我们先来分析Activity启动流程，对Activity组件有个整体性的认识。

## 一 Activity的启动流程

Activity的启动流程图（放大可查看）如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_start_flow.png" />

整个流程涉及的主要角色有：

- Instrumentation: 监控应用与系统相关的交互行为。
- AMS：组件管理调度中心，什么都不干，但是什么都管。
- ActivityStarter：Activity启动的控制器，处理Intent与Flag对Activity启动的影响，具体说来有：1 寻找符合启动条件的Activity，如果有多个，让用户选择；2 校验启动参数的合法性；3 返回int参数，代表Activity是否启动成功。
- ActivityStackSupervisior：这个类的作用你从它的名字就可以看出来，它用来管理任务栈。
- ActivityStack：用来管理任务栈里的Activity。
- ActivityThread：最终干活的人，是ActivityThread的内部类，Activity、Service、BroadcastReceiver的启动、切换、调度等各种操作都在这个类里完成。

注：这里单独提一下ActivityStackSupervisior，这是高版本才有的类，它用来管理多个ActivityStack，早期的版本只有一个ActivityStack对应着手机屏幕，后来高版本支持多屏以后，就
有了多个ActivityStack，于是就引入了ActivityStackSupervisior用来管理多个ActivityStack。

整个流程主要涉及四个进程：

- 调用者进程，如果是在桌面启动应用就是Launcher应用进程。
- ActivityManagerService等所在的System Server进程，该进程主要运行着系统服务组件。
- Zygote进程，该进程主要用来fork新进程。
- 新启动的应用进程，该进程就是用来承载应用运行的进程了，它也是应用的主线程（新创建的进程就是主线程），处理组件生命周期、界面绘制等相关事情。

有了以上的理解，整个流程可以概括如下：

1. 点击桌面应用图标，Launcher进程将启动Activity（MainActivity）的请求以Binder的方式发送给了AMS。
2. AMS接收到启动请求后，交付ActivityStarter处理Intent和Flag等信息，然后再交给ActivityStackSupervisior/ActivityStack
处理Activity进栈相关流程。同时以Socket方式请求Zygote进程fork新进程。
3. Zygote接收到新进程创建请求后fork出新进程。
4. 在新进程里创建ActivityThread对象，新创建的进程就是应用的主线程，在主线程里开启Looper消息循环，开始处理创建Activity。
5. ActivityThread利用ClassLoader去加载Activity、创建Activity实例，并回调Activity的onCreate()方法。这样便完成了Activity的启动。

👉 注：读者可以发现这上面有很多函数有Locked后缀，这代表这些函数需要进行多线程同步（synchronized）操作，它们会读写一些多线程共享的数据。

## 二 Activity的回退栈

要理解Activity回退栈，我们就要先理解Activity回退栈的功能结构，它的结构图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_stack_structure.png" width="600" />

主要角色有：

- ActivityRecord：描述栈里的Activity相关信息，对应着一个用户界面，是Activity管理的最小单位。
- TaskRecord：是一个栈式管理结构，每个TaskRecord可能包含一个或多个ActivityRecord，栈顶的ActivityRecord表示当前用户可见的页面。
- ActivityStack：是一个栈式管理结构，每个ActivityStack可能包含一个或多个TaskRecord，栈顶的TaskRecord表示当前用户可见的任务。
- ActivityStackSupervisior：管理者多个ActivityStack，当前只会有一个获取焦点（focused）的ActivityStack。
- ProcessReocord：保存着属于用一个进程的所有ActivityRecord，我们知道同一个应用的Activity可以运行在不同的进程里，同一个TaskRecord里的ActivityRecord
可能属于不同ProcessRecord，反之，运行在不同TaskRecord的ActivityRecord可能属于同一个ProcessRecord。

通过上面的图解分析，我想大家应该理解了Activity栈里的数据结构，接下来，我们再简单分析下这个数据类里的字段，字段比较多，大家有个印象就行，不必记住。

### 2.1 ActivityRecord

>ActivityRecord基本上是一个纯数据类，里面包含了Activity的各种信息。

- ActivityInfo	从<activity>标签中解析出来的信息，包含launchMode，permission，taskAffinity等
- mActivityType	Activity的类型有三种：APPLICATION_ACTIVITY_TYPE(应用)、HOME_ACTIVITY_TYPE(桌面)、RECENTS_ACTIVITY_TYPE(最近使用)
- appToken	当前ActivityRecord的标识 // window manager token 提供给windowmangerservice使用
- packageName	当前所属的包名，这是由<activity>静态定义的
- processName	当前所属的进程名，大部分情况都是由<activity>静态定义的，但也有例外
- taskAffinity	相同taskAffinity的Activity会被分配到同一个任务栈中
- intent	启动当前Activity的Intent
- launchedFromUid	启动当前Activity的UID，即发起者的UID
- launchedFromPackage	启动当前Activity的包名，即发起者的包名
- resultTo	在当前ActivityRecord看来，resultTo表示上一个启动它的ActivityRecord，当需要启动另一个ActivityRecord，会把自己作为resultTo，传递给下一个ActivityRecord
- state	ActivityRecord所处的状态，初始值是ActivityState.INITIALIZING
- app	ActivityRecord的宿主进程
- task	ActivityRecord的宿主任务
- inHistory	标识当前的ActivityRecord是否已经置入任务栈中
- frontOfTask	标识当前的ActivityRecord是否处于任务栈的根部，即是否为进入任务栈的第一个ActivityRecord
- newIntents	Intent数组，用于暂存还没有调度到应用进程Activity的Intent

这个对象在ActivityStarter的startActivityLocked()方法里被构建，下面分析ActivityStarter的时候我们会说。

### 2.2 TaskRecord

>TaskRecord的职责就是管理ActivityRecord，事实上，我们平时说的任务栈指的就是TaskRecord，所有ActivityRecord都必须要有宿主任务，如果不存在则新建一个。

- taskid	TaskRecord的唯一标识
- taskType	任务栈的类型，等同于ActivityRecord的类型，是由任务栈的第一个ActivityRecord决定的
- intent	在当前任务栈中启动的第一个Activity的Intent将会被记录下来，后续如果有相同的Intent时，会与已有任务栈的Intent进行匹配，如果匹配上了，就不需要再新建一个TaskRecord了
- realActivity, origActivity	启动任务栈的Activity，这两个属性是用包名(CompentName)表示的，real和orig是为了区分Activity有无别名(alias)的情况，如果AndroidManifest.xml中定义的Activity是一个alias，则此处real表示Activity的别名，orig表示真实的Activity
- affinity	TaskRecord把Activity的affinity记录下来，后续启动Activity时，会从已有的任务栈中匹配affinity，如果匹配上了，则不需要新建TaskRecord
- rootAffinity	记录任务栈中最底部Activity的affinity，一经设定后就不再改变
- mActivities	这是TaskRecord最重要的一个属性，TaskRecord是一个栈结构，栈的元素是ActivityRecord，其内部实现是一个数组mActivities
- stack	当前TaskRecord所在的ActivityStack

我们前面说过TaskRecord是一个栈结构，它里面的函数当然也侧重栈的管理：增删改查。事实上，在内部TaskRecord是用ArrayList来实现的栈的操作。

- getRootActivity()/getTopActivity()：任务栈有根部和顶部，可以通过这两个函数分别获取到根部和顶部的ActivityRecord。获取的过程就是对TaskRecord.mActivities进行
遍历，如果ActivityRecord的状态不是finishing，就认为是有效的ActivityRecord。
- topRunningActivityLocked()：虽然也是从顶至底对任务栈进行遍历获取顶部的ActivityRecord，但这个函数同getTopActivity()有区别：输入参数notTop，表示在遍历的过程中需要排除notTop这个ActivityRecord;
addActivityToTop()/addActivityAtBottom()：将ActivityRecord添加到任务栈的顶部或底部。
- moveActivityToFrontLocked()：该函数将一个ActivityRecord移至TaskRecord的顶部，实现方法就是先删除已有的，再在栈顶添加一个新的，这个和Intent.FLAG_ACTIVITY_REORDER_TO_FRONT相对应。
- setFrontOfTask()：ActivityRecord有一个属性是frontOfTask，表示ActivityRecord是否为TaskRecord的根Activity。该函数设置TaskRecord中所有ActivityRecord的frontOfTask属性，从栈底往上
开始遍历，第一个不处于finishing状态的ActivityRecord的frontOfTask属性置成true，其他都为false。
- performClearTaskLocked()：清除TaskRecord中的ActivityRecord。这个和Intent.FLAG_ACTIVITY_CLEAR_TOP相对应，当启动Activity时，设置了Intent.FLAG_ACTIVITY_CLEAR_TOP参数，那么在宿主
TaskRecord中，待启动ActivityRecord之上的其他ActivityRecord都会被清除。

基本上就是围绕ArrayList进行增删改查操作，再附加上一些状态变化，整个流程还是比较清晰的。

### 2.3 ActivityStack

>ActivityStack的职责是管理多个任务栈TaskRecord。

我们都知道Activity有着很多生命周期状态，这些状态就是由ActivityStack来推动完成的，在ActivityStack里，Activity有九种状态：

- INITIALIZING：初始化
- RESUMED：已显示
- PAUSING：暂停中
- PAUSED：已暂停
- STOPPING：停止中
- STOPPED：已停止
- FINISHING：结束中
- DESTROYING：销毁中
- DESTROYE：已销毁

这些状态的变化示意了Activity生命周期的走向。

我们也来简单看一下ActivityStack里的一些字段的含义：

- stackId	每一个ActivityStack都有一个编号，从0开始递增。编号为0，表示桌面(Launcher)所在的ActivityStack，叫做Home Stack
- mTaskHistory	TaskRecord数组，ActivityStack栈就是通过这个数组实现的
- mPausingActivity	在发生Activity切换时，正处于Pausing状态的Activity
- mResumedActivity	当前处于Resumed状态的ActivityRecord
- mStacks	ActivityStack会绑定到一个显示设备上，譬如手机屏幕、投影仪等，在AMS中，通过ActivityDisplay这个类来抽象表示一个显示设备，ActivityDisplay.mStacks表示当前已经绑定到显示设备
的所有ActivityStack。当执行一次绑定操作时，就会将ActivityStack.mStacks这个属性赋值成ActivityDisplay.mStacks，否则，ActivityStack.mStacks就为null。简而言之，当mStacks不为null时，
表示当前ActivityStack已经绑定到了一个显示设备。

Activity状态发生变化时，出来要调整ActivityRecord.state的状态，还要调整ActivityRecord在栈里的位置，事实上，ActivityStack也是一个栈式的结构，只不过它管理的是TaskRecord，和ActivityRecord
相关的操作也是先找到对应的TaskRecord，再由TaskRecord去完成具体的操作。

我们简单的看一下ActivityStack里面的方法。

- findTaskLocked()：该函数的功能是找到目标ActivityRecord(target)所在的任务栈(TaskRecord)，如果找到，则返回栈顶的ActivityRecord，否则，返回null
- findActivityLocked()：根据Intent和ActivityInfo这两个参数可以获取一个Activity的包名，该函数会从栈顶至栈底遍历ActivityStack中的所有Activity，如果包名匹配成功，就返回
- moveToFront()：该函数用于将当前的ActivityStack挪到前台，执行时，调用ActivityStack中的其他一些判定函数
- isAttached()：用于判定当前ActivityStack是否已经绑定到显示设备
- isOnHomeDisplay()：用于判定当前是否为默认的显示设备(Display.DEFAULT_DISPLAY)，通常，默认的显示设备就是手机屏幕
- isHomeStack()：用于判定当前ActivityStack是否为Home Stack，即判定当前显示的是否为桌面(Launcher)
- moveTaskToFrontLocked()：该函数用于将指定的任务栈挪到当前ActivityStack的最前面。在Activity状态变化时，需要对已有的ActivityStack中的任务栈进行调整，待显示Activity的宿主任务需要挪到前台
- insertTaskAtTop()：将任务插入ActivityStack栈顶

注：这里提到了ActivityDisplay的概念，这里简单说一下，我们知道Android是支持多屏显示的，每个显示屏对应者一个ActivityDisplay，默认是手机屏幕，ActivityDisplay是ActivityStackSupervisior的一个内部类，
ActivityStackSupervisor间接通过ActivityDisplay来维护多个ActivityStack的状态。 ActivityStack有一个属性是mStacks，当mStacks不为空时，表示ActivityStack已经绑定到了显示设备， 其实ActivityStack.mStacks
只是一个副本，真正的对象在ActivityDisplay中，ActivityDisplay的一些属性如下所示：

- mDisplayId	显示设备的唯一标识
- mDisplay	获取显示设备信息的工具类，
- mDisplayInfo	显示设备信息的数据结构，包括类型、大小、分辨率等
- mStacks	绑定到显示设备上的ActivityStack

### 2.4 ActivityStackSupervisior

>ActivityStackSupervisior用来管理ActivityStack。

ActivityStackSupervisior的一些常见属性如下所示：

- mHomeStack	主屏(桌面)所在ActivityStack
- mFocusedStack	表示焦点ActivityStack，它能够获取用户输入
- mLastFocusedStack	上一个焦点ActivityStack
- mActivityDisplays	表示当前的显示设备，ActivityDisplay中绑定了若干ActivityStack。通过该属性就能间接获取所有ActivityStack的信息

ActivityStackSupervisior里有很多方法与ActivityStack里的方法类似，但是ActivityStackSupervisior是针对多个ActivityStack进行操作。例如：findTaskLocked()， findActivityLocked()， topRunningActivityLocked(), 
ensureActivitiesVisibleLocked()。

## 三 Activity的生命周期

Activity的生命周期也是个老生常谈的问题，今天我们从源码的角度去分析Activity的生命周期是如何驱动的，以及它是如何变化的。

这里贴一张[android-lifecycle](https://github.com/xxv/android-lifecycle)项目关于Activity与Fragment生命周期图

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/complete_android_fragment_lifecycle.png"/>

读者可以从上图看出，Activity有很多种状态，状态之间的变化也比较复杂，在众多状态中，只有三种是常驻状态：

- Resumed（运行状态）：Activity处于前台，用户可以与其交互。
- Paused（暂停状态）：Activity被其他Activity部分遮挡，无法接受用户的输入。
- Stopped（停止状态）：Activity被完全隐藏，对用户不可见，进入后台。

其他的状态都是中间状态。

我们再来看看生命周期变化时的整个调度流程，生命周期调度流程图如下所示：

<img src="https://github.com/guoxiaoxing/android-open-source-project-analysis/raw/master/art/app/component/activity_lifecycle_structure.png" />

所以你可以看到，整个流程是这样的：

1. 比方说我们点击跳转一个新Activity，这个时候Activity会入栈，同时它的生命周期也会从onCreate()到onResume()开始变换，这个过程是在ActivityStack里完成的，ActivityStack
是运行在Server进程里的，这个时候Server进程就通过ApplicationThread的代理对象ApplicationThreadProxy向运行在app进程ApplicationThread发起操作请求。
2. ApplicationThread接收到操作请求后，因为它是运行在app进程里的其他线程里，所以ApplicationThread需要通过Handler向主线程ActivityThread发送操作消息。
3. 主线程接收到ApplicationThread发出的消息后，调用主线程ActivityThread执行响应的操作，并回调Activity相应的周期方法。

👉 注：这里提到了主线程ActivityThread，更准确来说ActivityThread不是线程，因为它没有继承Thread类或者实现Runnable接口，它是运行在应用主线程里的对象，那么应用的主线程
到底是什么呢？从本质上来讲启动启动时创建的进程就是主线程，线程和进程处理是否共享资源外，没有其他的区别，对于Linux来说，它们都只是一个struct结构体。

上述这个流程的函数调用链如下所示：

```java
ActivityThread.handleLaunchActivity
    ActivityThread.handleConfigurationChanged
        ActivityThread.performConfigurationChanged
            ComponentCallbacks2.onConfigurationChanged

    ActivityThread.performLaunchActivity
        LoadedApk.makeApplication
            Instrumentation.callApplicationOnCreate
                Application.onCreate

        Instrumentation.callActivityOnCreate
            Activity.performCreate
                Activity.onCreate

        Instrumentation.callActivityonRestoreInstanceState
            Activity.performRestoreInstanceState
                Activity.onRestoreInstanceState

    ActivityThread.handleResumeActivity
        ActivityThread.performResumeActivity
            Activity.performResume
                Activity.performRestart
                    Instrumentation.callActivityOnRestart
                        Activity.onRestart

                    Activity.performStart
                        Instrumentation.callActivityOnStart
                            Activity.onStart

                Instrumentation.callActivityOnResume
                    Activity.onResume
```

其他的生命周期在变化时调用流程和上面是一样的，读者可以自己举一反三。

启动新的Activity发出的消息是LAUNCH_ACTIVITY，这些消息定义在ActivityThread的内部类H（Handler）里，一共有54个，大部分都是我们熟悉的操作。

```java
public static final int LAUNCH_ACTIVITY         = 100;
public static final int PAUSE_ACTIVITY          = 101;
public static final int PAUSE_ACTIVITY_FINISHING= 102;
public static final int STOP_ACTIVITY_SHOW      = 103;
public static final int STOP_ACTIVITY_HIDE      = 104;
public static final int SHOW_WINDOW             = 105;
public static final int HIDE_WINDOW             = 106;
public static final int RESUME_ACTIVITY         = 107;
public static final int SEND_RESULT             = 108;
public static final int DESTROY_ACTIVITY        = 109;
public static final int BIND_APPLICATION        = 110;
public static final int EXIT_APPLICATION        = 111;
public static final int NEW_INTENT              = 112;
public static final int RECEIVER                = 113;
public static final int CREATE_SERVICE          = 114;
public static final int SERVICE_ARGS            = 115;
public static final int STOP_SERVICE            = 116;

public static final int CONFIGURATION_CHANGED   = 118;
public static final int CLEAN_UP_CONTEXT        = 119;
public static final int GC_WHEN_IDLE            = 120;
public static final int BIND_SERVICE            = 121;
public static final int UNBIND_SERVICE          = 122;
public static final int DUMP_SERVICE            = 123;
public static final int LOW_MEMORY              = 124;
public static final int ACTIVITY_CONFIGURATION_CHANGED = 125;
public static final int RELAUNCH_ACTIVITY       = 126;
public static final int PROFILER_CONTROL        = 127;
public static final int CREATE_BACKUP_AGENT     = 128;
public static final int DESTROY_BACKUP_AGENT    = 129;
public static final int SUICIDE                 = 130;
public static final int REMOVE_PROVIDER         = 131;
public static final int ENABLE_JIT              = 132;
public static final int DISPATCH_PACKAGE_BROADCAST = 133;
public static final int SCHEDULE_CRASH          = 134;
public static final int DUMP_HEAP               = 135;
public static final int DUMP_ACTIVITY           = 136;
public static final int SLEEPING                = 137;
public static final int SET_CORE_SETTINGS       = 138;
public static final int UPDATE_PACKAGE_COMPATIBILITY_INFO = 139;
public static final int TRIM_MEMORY             = 140;
public static final int DUMP_PROVIDER           = 141;
public static final int UNSTABLE_PROVIDER_DIED  = 142;
public static final int REQUEST_ASSIST_CONTEXT_EXTRAS = 143;
public static final int TRANSLUCENT_CONVERSION_COMPLETE = 144;
public static final int INSTALL_PROVIDER        = 145;
public static final int ON_NEW_ACTIVITY_OPTIONS = 146;
public static final int CANCEL_VISIBLE_BEHIND = 147;
public static final int BACKGROUND_VISIBLE_BEHIND_CHANGED = 148;
public static final int ENTER_ANIMATION_COMPLETE = 149;
public static final int START_BINDER_TRACKING = 150;
public static final int STOP_BINDER_TRACKING_AND_DUMP = 151;
public static final int MULTI_WINDOW_MODE_CHANGED = 152;
public static final int PICTURE_IN_PICTURE_MODE_CHANGED = 153;
public static final int LOCAL_VOICE_INTERACTION_STARTED = 154;
```

我们前面说到Activity的生命周期是由ActivityStack来驱动的，应用的Activity在切换时，ActivityStack会对相应的任务战TaskRecord进行调整，以前的Activity要出栈
销毁或者移动到后台，要显示的Activity添加到栈顶。伴随着对任务栈的操作，Activity的生命周期也在不断的变化。

在ActivityStack里与Activity生命周期变化有关的函数主要有以下几个：

- startActivityLocked()
- resumeTopActivityLocked()
- completeResumeLocked()
- startPausingLocked()
- completePauseLocked()
- stopActivityLocked()
- activityPausedLocked()
- finishActivityLocked()
- activityDestroyedLocked()

## 四 Activity的启动模式

>启动模式会影响Activity的启动行为，默认情况下，启动一个Activity就是创建一个实例，然后进入回退栈，但是我们可以通过启动模式来改变这种行为，实现不同的交互效果。

那么有哪些设置会影响这种行为呢？🤔

首先是<activity>标签里的参数：

- taskAffinity：指定Activity所在的任务栈，<appliocation>和<activity>标签均可设置，如果没有设置taskAffinity就是当前包名。
- launchMode：Activity的启动模式。
- allowTaskReparenting：当启动 Activity 的任务栈接下来转至前台时，Activity 是否能从该任务栈转移至其他任务栈，“true”表示它可以转移，“false”表示它仍须留在启动它的任务栈。
- clearTaskOnLaunch：是否每当从主屏幕重新启动任务时都从中移除根 Activity 之外的所有 Activity，“true”表示始终将任务清除到只剩其根 Activity；“false”表示不做清除。 默认值为“false”。
该属性只对启动新任务的 Activity（根 Activity）有意义；对于任务中的所有其他 Activity，均忽略该属性。
- alwaysRetainTaskState：系统是否始终保持 Activity 所在任务的状态，“true”表示保持，“false”表示允许系统在特定情况下将任务重置到其初始状态。默认值为“false”。
该属性只对任务的根 Activity 有意义；对于所有其他 Activity，均忽略该属性。
- finishOnTaskLaunch：每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例，“true”表示应关闭，“false”表示不应关闭。默认值为“false”。

注：更多和标签相关的参数可以参见[activity-element](https://developer.android.com/guide/topics/manifest/activity-element.html#aff)。

然后是Intent里的标志位：

- FLAG_ACTIVITY_NEW_TASK：每当用户再次启动其任务（在主屏幕上选择任务）时，是否应关闭（完成）现有 Activity 实例 —“true”表示应关闭，“false”表示不应关闭。 默认值为“false”。
- FLAG_ACTIVITY_SINGLE_TOP：如果正在启动的 Activity 是当前 Activity（位于返回栈的顶部），则 现有实例会接收对 onNewIntent() 的调用，而不是创建 Activity 的新实例。
正如前文所述，这会产生与 "singleTop"launchMode 值相同的行为。
- FLAG_ACTIVITY_CLEAR_TOP：如果正在启动的 Activity 已在当前任务中运行，则会销毁当前任务顶部的所有 Activity，并通过 onNewIntent() 将此 Intent 传递给 Activity 已恢复的
实例（现在位于顶部），而不是启动该 Activity 的新实例。

什么情况下需要设置FLAG_ACTIVITY_NEW_TASK标志位呢？🤔

一般说来，主要有以下四种情况：

- 调用者不是Activity Context；
- 调用者Activity调用single Instance；
- 目标Activity设置的有single Instance或者single task；
- 调用者处于finish状态；

启动模式一共有四种：

- standard：多实例模式，每次启动都会有创建一个实例，默认会进入启动它的那个Activity所属的任务栈的栈顶，读者以前可能使用过Application Context去启动Activity，这是情况下会报错，就是因为
Application Context没有所谓的任务栈，解决的方式就是给它添加一个FLAG_ACTIVITY_NEW_TASK的标志位，创建一个新的任务栈。
- singleTop：栈顶复用模式，如果新启动的Activity已经位于任务栈顶，则不会创建新的实例，而是回调原来Activity实例的onNewIntent()方法，如果新启动的Activity没有位于任务栈顶，则会创建
新的Activity实例。
- singleTask：栈内复用模式，如果新启动的Activity已经位于任务栈内，则不会创建新的实例，而是回调原来Activity实例的onNewIntent()方法并且**清除它之上的Activity（这里需要注意一下：任务栈里
的Activity是永远不会重排序的，所以它会清楚上方所有的Activity来让自己回到栈顶）**，如果新启动的Activity
没有位于任务栈，则新建一个Activity实例。
- singleInstance：单实例模式，和singleTask相似，但是singleTask可以在多个栈里拥有多个实例，而singleInstance在多个栈里只能有唯一实例，这个一般用在特殊场景里，例如电话界面。

我们再来总结一下这些启动模式的区别：

- standard和singleTop的行为很相近，但遇到相同的栈顶Activity实例，standard会再次新建一个，而singleTop不会;
- singleTop和singleTask在找到目标的Activity实例时，会调用其onNewIntent()方法; singleTop可以存在多个相同的Activity实例，而singleTask仅存在一个;
- singleTask和singleInstance都只会存在一个相同的Activity实例，singleTask任务中可以有其他不同的Activity实例，而singleInstance栈中仅有一个Activity实例。

我们前面说过，ActivityRecord里有个ActivityInfo成员变量用来描述Activity的相关信息，ActivityInfo是在解析AndroidManifest.xml里的<activity>标签获取的。
它里面的launchMode对应上面启动模式。

```java
public static final int LAUNCH_MULTIPLE = 0;
public static final int LAUNCH_SINGLE_TOP = 1;
public static final int LAUNCH_SINGLE_TASK = 2;
public static final int LAUNCH_SINGLE_INSTANCE = 3;
```

## 五 Activity的通信方式

Activity之间也经常需要传递数据，这个一般通过以下方式来完成。

原始Activity

```java
startActivityForResult(intent, requestCode, resultCode);

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
}
```

目标Activity

```java
setResult(resultCode, intent);
```

我们来看下setResult()方法的实现。

```java
public final void setResult(int resultCode, Intent data) {
    synchronized (this) {
        mResultCode = resultCode;
        mResultData = data;
    }
}
```
就是个简单的赋值操作，这说明会有方法来去这个变量的值，什么时候来取？🤔根据平时的开发经验，Activity finsh()或者onBackPress()来取，将这两个值通过
onActivityResult(int requestCode, int resultCode, Intent data)返回给原始Activity。

我们来梳理一下整个流程。
