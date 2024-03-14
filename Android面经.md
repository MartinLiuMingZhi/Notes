# Android面经

## 一、四大组件

### 1、Activity的生命周期

1. 启动Activity：系统会先调用**onCreate**方法，然后调用**onStart**方法，最后调用**onResume**，Activity进入运行状态。
2. 当前Activity被其他Activity覆盖其上或被锁屏：系统会调用**onPause**方法，暂停当前Activity的执行。
3. 当前Activity由被覆盖状态回到前台或解锁屏：系统会调用**onResume**方法，再次进入运行状态。
4. 当前Activity转到新的Activity界面或按Home键回到主屏，自身退居后台：系统会先调用**onPause**方法，然后调用**onStop**方法，进入停滞状态。
5. 用户后退回到此Activity：系统会先调用**onRestart**方法，然后调用**onStart**方法，最后调用**onResume**方法，再次进入运行状态。
6. 当前Activity处于被覆盖状态或者后台不可见状态，即第2步和第4步，系统内存不足，杀死当前Activity，而后用户退回当前Activity：再次调用**onCreate**方法、**onStart**方法、**onResume**方法，进入运行状态。
7. 用户退出当前Activity：系统先调用**onPause**方法，然后调用**onStop**方法，最后调用**onDestory**方法，结束当前Activity。

### 2、Activity的四种启动模式

- **standard（标准模式）**：如果在mainfest中不设置就默认standard。standard就是新建一个Activity就在栈中新建一个activity实例；
- **singleTop（栈顶复用模式）**：与standard相比栈顶复用可以有效减少activity重复创建对资源的消耗，但是这要根据具体情况而定，不能一概而论；
- **singleTask（栈内单例模式）**：栈内只有一个activity实例，栈内已存activity实例，在其他activity中start这个activity，Android直接把这个实例上面其他activity实例踢出栈GC掉；
- **singleInstance（堆内单例）** ：整个手机操作系统里面只有一个实例存在就是内存单例；如APP经常调用的拨打电话、系统通讯录、系统Launcher、锁屏键、来电显示等系统应用。singleInstance适合需要与程序分离开的页面。例如闹铃提醒，将闹铃提醒与闹铃设置分离。

在singleTop、singleTask、singleInstance 中如果在应用内存在Activity实例，并且再次发生startActivity(Intent intent)回到Activity后,由于并不是重新创建Activity而是复用栈中的实例，因此Activity再获取焦点后并没调用onCreate、onStart，而是直接调用了onNewIntent(Intent intent)函数。

## 二、View

### 1、View的绘制流程

视图绘制的起点在 ViewRootImpl 类的 `performTraversals()`方法，在这个方法内其实是按照顺序依次调用了 `mView.measure()、mView.layout()、mView.draw()`

View的绘制流程分为3步：测量、布局、绘制，分别对应3个方法 measure、layout、draw

- **测量阶段**：measure 方法会被父 View 调用，在measure 方法中做一些优化和准备工作后会调用 onMeasure 方法进行实际的自我测量。onMeasure方法在View和ViewGroup做的事情是不一样的：
  - **View**：View 中的 onMeasure 方法会计算自己的尺寸并通过 setMeasureDimension 保存。
  - **ViewGroup**： ViewGroup 中的 onMeasure 方法会调用所有子 View的measure 方法进行自我测量并保存。然后通过子View的尺寸和位置计算出自己的尺寸并保存。
- **布局阶段**：layout 方法会被父View调用，layout 方法会保存父 View 传进来的尺寸和位置，并调用 onLayout 进行实际的内部布局。onLayout 在 View 和 ViewGroup 中做的事情也是不一样的：
  - **View**： 因为 View 是没有子 View 的，所以View的onLayout里面什么都不做。
  - **ViewGroup** ：ViewGroup 中的 onLayout 方法会调用所有子 View 的 layout 方法，把尺寸和位置传给他们，让他们完成自我的内部布局。
- **绘制阶段**：draw 方法会做一些调度工作，然后会调用 onDraw 方法进行 View 的自我绘制。draw 方法的调度流程大致是这样的：
  - **绘制背景**：对应 `drawBackground(Canvas)`方法。
  - **绘制主体**：对应 `onDraw(Canvas)`方法。
  - **绘制子View**： 对应 `dispatchDraw(Canvas)`方法。
  - **绘制滑动相关和前景**： 对应 `onDrawForeground(Canvas)`

### 2、MeasureSpec

MeasureSpec 是 View 的测量规则。通常父控件要测量子控件的时候，会传给子控件 widthMeasureSpec 和 heightMeasureSpec 这两个 int 类型的值。这个值里面包含两个信息，**SpecMode** 和 **SpecSize**。一个 int 值怎么会包含两个信息呢？我们知道 int 是一个4字节32位的数据，在这两个 int 类型的数据中，前面高2位是 **SpecMode** ，后面低30位代表了  **SpecSize**。mode 有三种类型：`UNSPECIFIED`，`EXACTLY`，`AT_MOST`

- **EXACTLY**：精准模式，当 width 或 height 为固定 xxdp 或者为 MACH_PARENT 的时候，是这种测量模式
- **AT_MOST**：当 width 或 height 设置为 warp_content 的时候，是这种测量模式
- **UNSPECIFIED**：父容器对当前 View 没有任何显示，子 View 可以取任意大小。一般用在系统内部，比如：Scrollview、ListView。

### 3、View的事件分发机制

在我们的手指触摸到屏幕的时候，事件其实是通过 `Activity -> ViewGroup -> View` 这样的流程到达最后响应我们触摸事件的 View。

说到事件分发，必不可少的是这几个方法：`dispatchTouchEvent()、onInterceptTouchEvent()、onTouchEvent。`接下来就按照`Activity -> ViewGroup -> View` 的流程来大致说一下事件分发机制。

我们的手指触摸到屏幕的时候，会触发一个 Action_Down 类型的事件，当前页面的 Activity 会首先做出响应，也就是说会走到 Activity 的 `dispatchTouchEvent()` 方法内。在这个方法内部简单来说是这么一个逻辑：

- 调用 `getWindow.superDispatchTouchEvent()。`
- 如果上一步返回 true，直接返回 true；否则就 return 自己的 `onTouchEvent()。` 这个逻辑很好理解，`getWindow().superDispatchTouchEvent()` 如果返回 true 代表当前事件已经被处理，无需调用自己的 onTouchEvent；否则代表事件并没有被处理，需要 Activity 自己处理，也就是调用自己的 onTouchEvent。

`getWindow()`方法返回了一个 Window 类型的对象，这个我们都知道，在 Android 中，PhoneWindow 是Window 的唯一实现类。所以这句本质上是调用了`PhoneWindow`中的`superDispatchTouchEvent()`。

而在 PhoneWindow 的这个方法中实际调用了`mDecor.superDispatchTouchEvent(event)`。这个 mDecor 就是 DecorView，它是 FrameLayout 的一个子类，在 DecorView 中的 `superDispatchTouchEvent()` 中调用的是 `super.dispatchTouchEvent()`。到这里就很明显了，DecorView 是一个 FrameLayout 的子类，FrameLayout 是一个 ViewGroup 的子类，本质上调用的还是 `ViewGroup的dispatchTouchEvent()`。

分析到这里，我们的事件已经从 Activity 传递到了 ViewGroup，接下来我们来分析下 ViewGroup 中的这几个事件处理方法。

在 ViewGroup 中的 `dispatchTouchEvent()`中的逻辑大致如下：

- 通过 `onInterceptTouchEvent()` 判断当前 ViewGroup 是否拦截事件，默认的 ViewGroup 都是不拦截的；
- 如果拦截，则 return 自己的 `onTouchEvent()`；
- 如果不拦截，则根据 `child.dispatchTouchEvent()`的返回值判断。如果返回 true，则 return true；否则 return 自己的 `onTouchEvent()`，在这里实现了未处理事件的向上传递。

通常情况下 ViewGroup 的 `onInterceptTouchEvent()`都返回 false，也就是不拦截。这里需要注意的是事件序列，比如 Down 事件、Move 事件......Up事件，从 Down 到 Up 是一个完整的事件序列，对应着手指从按下到抬起这一系列的事件，如果 ViewGroup 拦截了 Down 事件，那么后续事件都会交给这个 ViewGroup的onTouchEvent。如果 ViewGroup 拦截的不是 Down 事件，那么会给之前处理这个 Down 事件的 View 发送一个 Action_Cancel 类型的事件，通知子 View 这个后续的事件序列已经被 ViewGroup 接管了，子 View 恢复之前的状态即可。

这里举一个常见的例子：在一个 Recyclerview 钟有很多的 Button，我们首先按下了一个 button，然后滑动一段距离再松开，这时候 Recyclerview 会跟着滑动，并不会触发这个 button 的点击事件。这个例子中，当我们按下 button 时，这个 button 接收到了 Action_Down 事件，正常情况下后续的事件序列应该由这个 button处理。但我们滑动了一段距离，这时 Recyclerview 察觉到这是一个滑动操作，拦截了这个事件序列，走了自身的 `onTouchEvent()`方法，反映在屏幕上就是列表的滑动。而这时 button 仍然处于按下的状态，所以在拦截的时候需要发送一个 Action_Cancel 来通知 button 恢复之前状态。

事件分发最终会走到 View 的 `dispatchTouchEvent()`中。在 View 的 `dispatchTouchEvent()` 中没有 `onInterceptTouchEvent()`，这也很容易理解，View 不是 ViewGroup，不会包含其他子 View，所以也不存在拦截不拦截这一说。忽略一些细节，View 的 `dispatchTouchEvent()`中直接 return 了自己的 `onTouchEvent()`。如果 `onTouchEvent()`返回 true 代表事件被处理，否则未处理的事件会向上传递，直到有 View 处理了事件或者一直没有处理，最终到达了 Activity 的 `onTouchEvent()` 终止。

这里经常有人问 onTouch 和 onTouchEvent 的区别。首先，这两个方法都在 View 的 `dispatchTouchEvent()`中，是这么一个逻辑：

- 如果 touchListener 不为 null，并且这个 View 是 enable 的，而且 onTouch 返回的是 true，满足这三个条件时会直接 return true，不会走 `onTouchEvent()`方法。
- 上面只要有一个条件不满足，就会走到 `onTouchEvent()`方法中。所以 onTouch 的顺序是在 onTouchEvent 之前的。

### 4、dp 和 sp 的区别

- dp：一种基于屏幕密度的抽象单位。在每英寸160点的显示器上，1dp=1px。不同设备有不同的显示效果，这个和设备硬件有关，**px= dp （dpi/160）**

- sp：主要用于字体显示，与刻度无关的一种像素，与dp类似，但会随着系统的字体大小改变

## 三、线程与进程

### 1、Bundle机制

Bundle实现了Parcelable接口，所以他可以方便的在不同进程间传输，这里要注意我们传输的数据必须能够被序列化；

- 使用场景：
  - Activity状态数据的保存与恢复涉及到的两个回调：void onSaveInstanceState (Bundle outState)、void onCreate (Bundle savedInstanceState)
  - Fragment的setArguments方法：void setArguments (Bundle args)
  - 消息机制中的Message的setData方法：void setData (Bundle data)
- Bundle底层使用的是ArrayMap，这个集合类存储的也是键值对，但是与HashMap不同的是，HashMap采用的是“数组+链表”的方式存储，而ArrayMap中使用的是两个数组进行存储，一个数组存储key，一个数组存储value，内部的增删改查都将会使用二分查找来进行，这个和SparseArray差不多，只不过SparseArray的key值只能是int型的，而ArrayMap可以是Map型，所以在数据量不大的情况下可以使用这两个集合代替HashMap去优化性能；

### 2、Handler机制

Handler的作用是负责跨线程通信，这是因为在主线程不能做耗时操作，而子线程不能更新 UI，所以当子线程中进行耗时操作后需要更新 UI时，通过 Handler 将有关 UI 的操作切换到主线程中执行。

说到 Handler，就不得不提与之密切相关的这几个类：Message、MessageQueue，Looper。

- **Message**：Message 中有两个成员变量值得关注：target 和 callback。
  - target 其实就是发送消息的 Handler 对象
  - callback 是当调用 `handler.post(runnable)` 时传入的 Runnable 类型的任务。post 事件的本质也是创建了一个 Message，将我们传入的这个 runnable 赋值给创建的Message的 callback 这个成员变量。
- **MessageQueue**: 消息队列很明显是存放消息的队列，值得关注的是 MessageQueue 中的 `next()` 方法，它会返回下一个待处理的消息。
- **Looper**：Looper 消息轮询器其实是连接 Handler 和消息队列的核心。如果想要在一个线程中创建一个 Handler，首先要通过`Looper.prepare()`创建 Looper，之后还得调用`Looper.loop()`开启轮询。
  - **`prepare()`**： 这个方法做了两件事：首先通过`ThreadLocal.get()`获取当前线程中的Looper，如果不为空，则会抛出一个RunTimeException，意思是一个线程不能创建2个Looper。如果为null则执行下一步。第二步是创建了一个Looper，并通过 `ThreadLocal.set(looper)。`将我们创建的Looper与当前线程绑定。这里需要提一下的是消息队列的创建其实就发生在Looper的构造方法中。
  - **`loop()`**： 这个方法开启了整个事件机制的轮询。它的本质是开启了一个死循环，不断的通过 `MessageQueue的next()`方法获取消息。拿到消息后会调用 `msg.target.dispatchMessage()`来做处理。其实我们在说到 Message 的时候提到过，`msg.target` 其实就是发送这个消息的 handler。这句代码的本质就是调用 `handler的dispatchMessage()。`
- **Handler**：Handler 的分析着重在两个部分：发送消息和处理消息。
  - **发送消息**：其实发送消息除了 sendMessage 之外还有 sendMessageDelayed 和 post 以及 postDelayed 等等不同的方式。但它们的本质都是调用了 sendMessageAtTime。在 sendMessageAtTime 这个方法中调用了 enqueueMessage。在 enqueueMessage 这个方法中做了两件事：通过`msg.target = this`实现了消息与当前 handler 的绑定。然后通过`queue.enqueueMessage`实现了消息入队。
  - **处理消息**： 消息处理的核心其实就是`dispatchMessage()`这个方法。这个方法里面的逻辑很简单，先判断 `msg.callback` 是否为 null，如果不为空则执行这个 runnable。如果为空则会执行我们的`handleMessage`方法。

### 3、Looper死循环为什么不会导致应用卡死，即使不会的话，它会慢慢的消耗越来越多的资源吗？

- 对于线程即是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，例如，`binder`线程也是采用死循环的方法，通过循环方式不同与`Binder`驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。真正会卡死主线程的操作是在回调方法`onCreate/onStart/onResume`等操作时间过长，会导致掉帧，甚至发生`ANR`，`looper.loop`本身不会导致应用卡死。
- 主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到`Linux pipe/epoll`机制，简单说就是在主线程的`MessageQueue`没有消息时，便阻塞在`loop`的`queue.next()`中的`nativePollOnce()`方法里，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的`epoll`机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

### 4、主线程的消息循环机制是什么？

事实上，会在进入死循环之前便创建了新binder线程，在代码`ActivityThread.main()`中：

```java
public static void main(String[] args) {
  //创建Looper和MessageQueue对象，用于处理主线程的消息
  Looper.prepareMainLooper();
   //创建ActivityThread对象
   ActivityThread thread = new ActivityThread(); 
   //建立Binder通道 (创建新线程)
   thread.attach(false);
   Looper.loop(); //消息循环运行
   throw new RuntimeException("Main thread loop unexpectedly exited");
   }
```

**Activity的生命周期都是依靠主线程的** `Looper.loop`，当收到不同`Message`时则采用相应措施：一旦退出消息循环，那么你的程序也就可以退出了。 从消息队列中取消息可能会阻塞，取到消息会做出相应的处理。如果某个消息处理时间过长，就可能会影响UI线程的刷新速率，造成卡顿的现象。

`thread.attach(false)`方法函数中便会创建一个Binder线程（具体是指`ApplicationThread`，`Binder`的服务端，用于接收系统服务`AMS`发送来的事件），该Binder线程通过`Handler`将`Message`发送给主线程。「Activity 启动过程」

比如收到`msg=H.LAUNCH_ACTIVITY`，则调用`ActivityThread.handleLaunchActivity()`方法，最终会通过反射机制，创建`Activity`实例，然后再执行`Activity.onCreate()`等方法；

再比如收到`msg=H.PAUSE_ACTIVITY`，则调用`ActivityThread.handlePauseActivity()`方法，最终会执行`Activity.onPause()`等方法。

主线程的消息又是哪来的呢？当然是App进程中的其他线程通过Handler发送给主线程进程。

### 5、Handler同步屏障是什么

- Handler Message 种类

  Handler的Message种类分为3种：

  - 普通消息

  - 屏障消息

  - 异步消息

​		其中普通消息又称为同步消息，屏障消息又称为同步屏障。

​		我们通常使用的都是普通消息，而屏障消息就是在消息队列中插入一个屏障，在屏障之后的所有普通消息都会被挡着，不能被处理。		不过异步消息却例外，屏障不会挡住异步消息，因此可以这样认为：屏障消息就是为了确保异步消息的优先级，设置了屏障后，只能		处理其后的异步消息，同步消息会被挡住，除非撤销屏障。

- 屏障消息如何插入消息队列

  同步屏障是通过MessageQueue的postSyncBarrier方法插入到消息队列的。源码如下：

  ```java
  private int postSyncBarrier(long when) {
          synchronized (this) {
              final int token = mNextBarrierToken++;
              //1、屏障消息和普通消息的区别是屏障消息没有tartget。
              final Message msg = Message.obtain();
              msg.markInUse();
              msg.when = when;
              msg.arg1 = token;
  
              Message prev = null;
              Message p = mMessages;
              //2、根据时间顺序将屏障插入到消息链表中适当的位置
              if (when != 0) {
                  while (p != null && p.when <= when) {
                      prev = p;
                      p = p.next;
                  }
              }
              if (prev != null) { // invariant: p == prev.next
                  msg.next = p;
                  prev.next = msg;
              } else {
                  msg.next = p;
                  mMessages = msg;
              }
              //3、返回一个序号，通过这个序号可以撤销屏障
              return token;
          }
  }
  ```

  postSyncBarrier方法就是用来插入一个屏障到消息队列的，可以看到它很简单，从这个方法我们可以知道如下：

  - 屏障消息和普通消息的区别在于屏障没有tartget，普通消息有target是因为它需要将消息分发给对应的target，而屏障不需要被分发，它就是用来挡住普通消息来保证异步消息优先处理的。
  - 屏障和普通消息一样可以根据时间来插入到消息队列中的适当位置，并且只会挡住它后面的同步消息的分发。
  - postSyncBarrier返回一个int类型的数值，通过这个数值可以撤销屏障。
  - postSyncBarrier方法是私有的，如果我们想调用它就得使用反射。
  - 插入普通消息会唤醒消息队列，但是插入屏障不会。

- 屏障消息的工作原理

  我们知道MessageQueue是通过next方法来获取消息的

  ```java
  Message next() {
      //1、如果有消息被插入到消息队列或者超时时间到，就被唤醒，否则阻塞在这。
      nativePollOnce(ptr, nextPollTimeoutMillis);
      synchronized (this) {
          Message prevMsg = null;
          Message msg = mMessages;
          if (msg != null && msg.target == null) {//2、遇到屏障  msg.target == null
              do {
                  prevMsg = msg;
                  msg = msg.next;
              } while (msg != null && !msg.isAsynchronous());//3、遍历消息链表找到最近的一条异步消息
          }
          if (msg != null) {
              //4、如果找到异步消息
              if (now < msg.when) {//异步消息还没到处理时间，就在等会（超时时间）
                  nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
              } else {
                  //异步消息到了处理时间，就从链表移除，返回它。
                  mBlocked = false;
                  if (prevMsg != null) {
                      prevMsg.next = msg.next;
                  } else {
                      mMessages = msg.next;
                  }
                  msg.next = null;
                  if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                  msg.markInUse();
                  return msg;
              }
          } else {
              // 如果没有异步消息就一直休眠，等待被唤醒。
              nextPollTimeoutMillis = -1;
          }
          //...
      }
  }
  ```

  在注释2如果碰到屏障就遍历整个消息链表找到最近的一条异步消息，在遍历的过程中只有异步消息才会被处理执行到 `if (msg != null && msg.target == null){}`中的代码。

  屏障消息就是通过这种方式就挡住了所有的普通消息。

- 同步屏障的作用

  我们的手机屏幕刷新频率有不同的类型，60Hz、120Hz等。60Hz表示屏幕在一秒内刷新60次，也就是每隔16.6ms刷新一次。屏幕会在每次刷新的时候发出一个 VSYNC 信号，通知CPU进行绘制计算。具体到我们的代码中，可以认为就是执行`onMesure()`、`onLayout()`、`onDraw()`这些方法。view绘制的起点是在 `viewRootImpl.requestLayout()` 方法开始，这个方法会去执行上面的三大绘制任务，就是测量、布局、绘制。但是调用`requestLayout()`方法之后，并不会马上开始进行绘制任务，而是会给主线程设置一个同步屏障，并设置 VSYNC 信号监听。当 VSYNC 信号的到来，会发送一个异步消息到主线程Handler，执行我们上一步设置的绘制监听任务，并移除同步屏障。

### 6、什么是IdleHandler？

- **介绍**：

  `IdleHandler` 是 `MessageQueue` 内定义的一个接口，一般可用于做性能优化。当消息队列内没有需要立即执行的 `message` 时，会主动触发 `IdleHandler` 的 `queueIdle()` 方法。返回值为 false，即只会执行一次；返回值为 true，即每次当消息队列内没有需要立即执行的消息时，都会触发该方法。

  ```java
  public final class MessageQueue {
      public static interface IdleHandler {
          boolean queueIdle();
      }
  }
  ```

- **使用方式**：

  通过获取 `looper` 对应的 `MessageQueue` 队列注册监听。

  ```java
  Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override
      public boolean queueIdle() {
          // doSomething()
          return false;
      }
  });
  ```

- **源码解析**：

  IdleHandler 的执行源码很短。

  ```java
  Message next() {
      // 隐藏无关代码...
      int pendingIdleHandlerCount = -1; // -1 only during first iteration
      int nextPollTimeoutMillis = 0;
      for (; ; ) {
          // 隐藏无关代码...
          // If first time idle, then get the number of idlers to run.
          // Idle handles only run if the queue is empty or if the first message
          // in the queue (possibly a barrier) is due to be handled in the future.
          if (pendingIdleHandlerCount < 0
                  && (mMessages == null || now < mMessages.when)) {
              pendingIdleHandlerCount = mIdleHandlers.size();
          }
          if (pendingIdleHandlerCount <= 0) {
              // No idle handlers to run.  Loop and wait some more.
              mBlocked = true;
              continue;
          }
          if (mPendingIdleHandlers == null) {
              mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
          }
          mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
      }
      // Run the idle handlers.
      // We only ever reach this code block during the first iteration.
      for (int i = 0; i < pendingIdleHandlerCount; i++) {
          final IdleHandler idler = mPendingIdleHandlers[i];
          mPendingIdleHandlers[i] = null; // release the reference to the handler
          boolean keep = false;
          try {
              keep = idler.queueIdle();
          } catch (Throwable t) {
              Log.wtf(TAG, "IdleHandler threw exception", t);
          }
          if (!keep) {
              synchronized (this) {
                  mIdleHandlers.remove(idler);
              }
          }
      }
      // Reset the idle handler count to 0 so we do not run them again.
      pendingIdleHandlerCount = 0;
      // While calling an idle handler, a new message could have been delivered
      // so go back and look again for a pending message without waiting.
      nextPollTimeoutMillis = 0;
  }
  ```

  1. 在 `MessageQueue` 里 `next()` 方法的 for 死循环内，获取 `mIdleHandlers` 的数量 `pendingIdleHandlerCount`；
  2. 通过 `mMessages == null || now < mMessages.when` 判断当前消息队列为空或者目前没有需要执行的消息时，给 `pendingIdleHandlerCount` 赋值；
  3. 当数量大于 0，遍历取出数组内的 `IdleHandler`，执行 `queueIdle()` ；
  4. 返回值为 `false` 时，主动移除监听 `mIdleHandlers.remove(idler)`；

- **使用场景**：

  - 如果启动的 `Activity`、`Fragment`、`Dialog` 内含有大量数据和视图的加载，导致首次打开时动画切换卡顿或者一瞬间白屏，可将部分加载逻辑放到 `queueIdle()` 内处理。例如引导图的加载和弹窗提示等；
  - 系统源码中 `ActivityThread` 的 `GcIdler`，在某些场景等待消息队列暂时空闲时会尝试执行 GC 操作；
  - 系统源码中  `ActivityThread` 的 `Idler`，在 `handleResumeActivity()` 方法内会注册 `Idler()`，等待`handleResumeActivity` 后视图绘制完成，消息队列暂时空闲时再调用 `AMS` 的 `activityIdle` 方法，检查页面的生命周期状态，触发 `activity` 的 `stop` 生命周期等。这也是为什么我们 `BActivity` 跳转 `CActivity` 时，`BActivity` 生命周期的 `onStop()` 会在 `CActivity` 的 `onResume()` 后。
  - 一些第三方框架 `Glide` 和 `LeakCanary` 等也使用到 `IdleHandler`；

### 7、什么是HandlerThread？

- HandlerThread本质上是一个线程类，它继承了Thread；
- HandlerThread有自己的内部Looper对象，可以进行looper循环；
- 通过获取HandlerThread的looper对象传递给Handler对象，可以在handleMessage方法中执行异步任务；
- 创建HandlerThread后必须先调用HandlerThread.start()方法，Thread会先调用run方法，创建Looper对象。
  HandlerThread其实就是在一个子线程内部自己创建并管理了一个Looper。
- 如果经常要开启线程，接着又是销毁线程，这是很耗性能的，HandlerThread 很好的解决了这个问题
- HandlerThread 由于异步操作是放在 Handler 的消息队列中的，所以是串行的，但只适合并发量较少的耗时操作

### 8、进程间的通信方式有哪些？

- **Bundle/Intent传递数据**：

  可传递基本类型，String，实现了Serializable或Parcellable接口的数据结构。Serializable是Java的序列化方法，Parcellable是Android的序列化方法，前者代码量少（仅一句），但I/O开销较大，一般用于输出到磁盘或网卡；后者实现代码多，效率高，一般用户内存间序列化和反序列化传输。

- **文件共享**：

  对同一个文件先后写读，从而实现传输，Linux机制下，可以对文件并发写，所以要注意同步。顺便一提，Windows下不支持并发读或写。

- **Messenger**：

  Messenger是基于AIDL实现的，服务端（被动方）提供一个Service来处理客户端（主动方）连接，维护一个Handler来创建Messenger，在onBind时返回Messenger的binder。

- **AIDL**：

  AIDL通过定义服务端暴露的接口，以提供给客户端来调用，AIDL使服务器可以并行处理，而Messenger封装了AIDL之后只能串行运行，所以Messenger一般用作消息传递。通过编写aidl文件来设计想要暴露的接口，编译后会自动生成响应的java文件，服务器将接口的具体实现写在Stub中，用iBinder对象传递给客户端，客户端bindService的时候，用asInterface的形式将iBinder还原成接口，再调用其中的方法。

- **ContentProvider**：

  系统四大组件之一，底层也是Binder实现，主要用来为其他APP提供数据，可以说天生就是为进程通信而生的。自己实现一个ContentProvider需要实现6个方法，其中onCreate是主线程中回调的，其他方法是运行在Binder之中的。自定义的ContentProvider注册时要提供authorities属性，应用需要访问的时候将属性包装成Uri.parse("content://authorities")。还可以设置permission，readPermission，writePermission来设置权限。 ContentProvider有query，delete，insert等方法，看起来貌似是一个数据库管理类，但其实可以用文件，内存数据等等一切来充当数据源，query返回的是一个Cursor，可以自定义继承AbstractCursor的类来实现。

- **Socket**

### 9、ANR发生的原因及其解决办法

ANR的全称是application not responding，是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。一般地，这时往往会弹出一个提示框，告知用户当前xxx未响应，用户可选择继续等待或者Force Close。

首先ANR的发生是有条件限制的，分为以下三点：

- 只有主线程才会产生ANR，主线程就是UI线程；
- 必须发生某些输入事件或特定操作，比如按键或触屏等输入事件，在BroadcastReceiver或Service的各个生命周期调用函数；
- 上述事件响应超时，不同的context规定的上限时间不同
  1. 主线程对输入事件5秒内没有处理完毕
  2. 主线程在执行BroadcastReceiver的onReceive()函数时10秒内没有处理完毕
  3. 主线程在前台Service的各个生命周期函数时20秒内没有处理完毕（后台Service 200s）

那么导致ANR的根本原因是什么呢？简单的总结有以下两点：

- 主线程执行了**耗时操作**，比如数据库操作或网络编程，I/O操作
- 其他进程（就是其他程序）占用CPU导致本进程**得不到CPU时间片**，比如其他进程的频繁读写操作可能会导致这个问题

那么如何避免ANR的发生呢或者说ANR的解决办法是什么呢？

- 避免在主线程执行耗时操作，所有耗时操作应新开一个子线程完成，然后再在主线程更新UI
- BroadcastReceiver要执行耗时操作时应启动一个service，将耗时操作交给service来完成
- 避免在Intent Receiver里启动一个Activity，因为它会创建一个新的画面，并从当前用户正在运行的程序上抢夺焦点。如果你的应用程序在响应Intent广播时需要向用户展示什么，你应该使用Notification Manager来实现

## 四、项目架构

### 1、MVC

M——模型层（Model）负责处理数据的加载或者存储

V——视图层（View）负责界面数据的展示，与用户进行交互

C——控制器层（Controller）负责逻辑业务的处理

<img src="C:\Users\zzp\AppData\Roaming\Typora\typora-user-images\image-20230312230131742.png" alt="image-20230312230131742" style="zoom:80%;" />

在MVC模式中，View层可以直接访问Model层和Controller层，所以View层包含Model层信息和Controller层的业务逻辑处理

- 优点：
  - 耦合性低，生命周期成本低，部署快，可维护性高，适用于快速开发的小型项目

- 缺点：
  - 不适合大型，中等项目，View层Controller层连接过于紧密
  - View层对Model层的访问效率低
  - 一般的高级UI页面工具和构造器不支持MVC模式
- 注意：
  - Activity和Fragment既有View的性质，又具有Controller的性质，导致Acyivity和Fragment很重
  - MVC中View层与Model层直接交互，所以Activity和Fragment与Model的耦合性很高

### 2、MVP

M——数据层（Model）负责对数据的存取操作，例如对数据库的读写，网络的数据的请求等

V——视图层（View）负责对数据的展示，提供友好的界面与用户进行交互，Android通常将Activity或者Fragment作为View层

P——控制器层（Presenter）连接View层与Model层的桥梁并对业务逻辑进行处理

<img src="C:\Users\zzp\AppData\Roaming\Typora\typora-user-images\image-20230312230639460.png" alt="image-20230312230639460" style="zoom:80%;" />

 **MVP执行流程：**

- View层收到用户的操作
- View层把用户的操作交给Presenter
- Presenter直接操作Model层进行业务逻辑处理
- Model层处理完毕后，通知Presenter
- Presenter收到通知后，去更新View层

在MVP模式中，Model与View无法直接进行交互，所以Presenter层会从Model层获得数据，适当处理后交给View层进行显示，Presenter层将View层和Model层进行隔离，使View和Model之间不存在耦合，同时将业务逻辑从View层剥离

- 优点：
  - 模型与视图完全分离，修改View而不Model
  - 可以更高效的使用Model，所有的交互都发生在——Presenter内部
  - 将一个Presenter用于多个视图，而不需要改变Presenter的逻辑，View变化比Model变化频繁
  - 逻辑结构清晰，View层代码不再臃肿

- 缺点：
  - MVP模式基于接口设计，会增加很多类，代码逻辑虽然清晰，但代码量庞大
  - MVP适用于中小型项目，大型项目慎用
- MVC和MVP的主要区别：
  - MVP中View与Model并不直接交互，而是通过与Presenter交互来与Model间接交互
  - MVC中Controller是基于行为的，并且可以被多个View共享，Controller可以负责决定显示哪个View
  - MVP中Presenter与View的交互是通过接口来进行的，更有利于添加单元测试。
  - MVC中View可以与Model直接交互，通常View与Presenter是一对一的，但复杂的View可能绑定多个Presenter来处理逻辑 
- 注意：
  - MVP中将这三层分别抽象到各自的接口当中，通过接口将层次之间进行隔离
  - Presenter对View和Model的相互依赖也是依赖于各自的接口，符合了接口隔离原则
  - Presenter层中包含了一个View接口，并且依赖于Model接口，从而将Model层与View层联系在一起
  - View层会持有一个Presenter成员变量并且只保留对Presenter接口的调用，具体业务逻辑全部交由Presenter接口实现类中处理

### 3、MVVM

M——Model（模型）实体模型，定义实体类，获取业务数据模型，如通过数据库或者网络来操作数据等

V——View（视图）布局文件(XML），主要进行控件的初始化设置

VM——ViewModel（控制器）：连接 View 与 Model 的中间桥梁，ViewModel 与 Model 直接交互，通过DataBinding将数据变化反应给View
![image-20230312231248908](C:\Users\zzp\AppData\Roaming\Typora\typora-user-images\image-20230312231248908.png)

**注：**ViewModel可以理解为View与Presenter的合成体

**优点：**

- 低耦合
  - MVVM 模式中，数据处理逻辑是独立于 UI 层的
  - ViewModel 只负责提供数据和处理数据，不会持有 View 层的引用
  - View 层只负责对数据变化的监听，不会处理任何跟数据相关的逻辑
  - View 层的 UI 发生变化时，也不需要像 MVP 模式那样修改对应接口和方法实现，一般情况下ViewModel 不需要做太多的改动
- 数据驱动
  - UI 的展现是依赖于数据的，数据的变化会自然的引发 UI 的变化，而 UI 的改变也会使数据 Model 进行对应的更新
  - ViewModel 只需要处理数据，而 View 层只需要监听并使用数据进行 UI 更新
- 异步线程更新Model
  - Model 数据可以在异步线程中发生变化，此时调用者不需要做额外的处理
  - 数据绑定框架会将异步线程中数据的变化通知到 UI 线程中交给 View去更新
- 方便协作
  - View 层和逻辑层几乎没有耦合，在团队协作的过程中，可以一个人负责 UI，一个人负责数据处理。并行开发，保证开发进度
- 易于单元测试
  - ViewModel 层只负责处理数据，在进行单元测试时，测试不需要构造一个 fragment/Activity/TextView 等来进行数据层的测试
  - View 层也一样，只需要输入指定格式的数据即可进行测试，而且两者相互独立，不会互相影响
- 数据复用
  - ViewModel 层对数据的获取和处理逻辑，尤其是使用 Repository 模式时，获取数据的逻辑完全是可以复用的
  - 开发者可以在不同的模块，多次方便的获取同一份来源的数据
  - 同样的一份数据，在版本功能迭代时，逻辑层不需要改变，只需要改变 View 层即可

## 五、Kotlin

### 1、扩展函数原理

扩展函数实际上就是一个对应 Java 中的静态函数，这个静态函数参数为接收者类型的对象，然后利用这个对象就可以访问这个类中的成员属性和方法了，并且最后返回一个这个接收者类型对象本身。这样在外部感觉和使用类的成员函数是一样的：

```java
// 这个类名就是顶层文件名+“Kt”后缀
public final class ExtendsionTextViewKt {
   // 扩展函数 isBold 对应实际上是 Java 中的静态函数，并且传入一个接收者类型对象作为参数
   @NotNull
   public static final TextView isBold(@NotNull TextView $receiver) {
      Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
      $receiver.getPaint().setFakeBoldText(true); // 设置加粗
      return $receiver; // 最后返回这个接收者对象自身，以致于我们在Kotlin中完全可以使用 this 替代接收者对象或者直接不写。
   }
}
```

### 2、协程原理
