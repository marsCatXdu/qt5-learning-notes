## 关于指针

- 通过父子关系，安全使用指针：

  对于创建在堆上的、有父子关系的对象，在父对象被销毁时，子对象也会被一并带走。确定父子关系有两种方式：其一是在构造时作为 parent* 参数传入，其二时在运行时使用 `QObject::setParent(QObject *parent)` 来指定。

  想要获取父子，可以用 `QObject *QObject::parent() const` 和 `const QObjectList & QObject::children() const`。

  给一个 widget 设置了 parent，则该 widget 的指针所有权就会被转移给 qt 进行处理。

- 指针用后，通过 `deleteLater()` 删除：

  `deleteLater()` 会将 `delete` 这件事放入事件循环队列，排到了才会删。所以像下面这样的操作是危险的，会搞出野指针：

  ```c++
  myObj->deleteLater();
  myObj = nullptr;
  ```


### QPointer

QPointer 大致用法如下：

```C++
QPointer<QPushButton> button1 = new QPushButton;
QPointer<QPushButton> button2 = button1;

button2->setText("Hello world!");
button2->show();

delete button1;
delete button2;
```

好处：删除内存之后指针值自动置空，防止野指针出现，而且多次 delete 无危险。



## 事件循环

`QApplication` 维护的事件循环被称为"主事件循环"（Main EvnetLoop）。`QApplication::exec` 就是 `QEventLoop::exec` 。

事件的发出方式：

- `sendEvent` ： 发出的事件会立即处理（即同步处理）；

- `postEvent` ： 发出的事件加入事件队列，在下一轮事件循环中处理（即异步处理）；

- `sendPostedEvents` ： 将已经加入队列、准备异步执行的事件立即执行。

  流畅运行的应用界面，需要由主线程（ UI 线程）的持续流畅执行来保证，流畅的程度至少要比人的反应速度低，才会看起来一直都在流畅的运行。但某些情况下我们不得已需要在 UI 线程做耗时操作，而在主线程在忙于计算时是顾不上处理事件队列中的事件的，这就会导致应用卡顿。这种情况可以通过在耗时计算流程中插入 `processEvents()` ，来要求 Qt 去立即处理一下事件循环

除了主事件循环之外，我们还可以根据需求来声明和管理我们自己的事件循环（称为“本地事件循环”）。比如下面使用网络请求进行登录的例子。

```C++
bool login(const QString &userName, const QString &passwdHash, const QString &slat)
{
    //声明本地EventLoop
    QEventLoop loop;
    bool result = false;
    //先连接好信号
    connect(&network, &Network::result, [&](bool r, const QString &info){
        result = r;
        qDebug() << info;
        //槽中退出事件循环
        loop.quit();
    });
    //发起登录请求
    sendLoginRequest(userName, passwdHash, slat);
    //启动事件循环。阻塞当前函数调用，但是事件循环还能运行。
    //这里不会再往下运行，直到前面的槽中，调用loop.quit之后，才会继续往下走
    loop.exec();
    //返回result。loop退出之前，result中的值已经被更新了。
    return result;
}
```



## 信号

信号和事件，可以是实现同样目的的两种不同方式。

信号的发送和处理主要是同步的（在跨线程信号以及手动设置的情况下，也可以是异步的），和 GUI 的事件循环没有关系：一个信号被触发时，与其相连的槽函数会立即执行。如果一个信号连接了多个槽函数，那这些槽函数会以随机顺序逐个执行。如果被连接的槽函数又触发了新的信号，那就会递归跑下去，直到全部返回。

对于 `QueuedConnection`，触发信号时该信号会被包装成一个 `QEvent` 并放入事件队列。在不指定连接方式的情况下，如果是跨线程 connect 则都会被默认设置为这种连接方式。

一般情况下不用指明连接方式。在涉及到多个线程之间的协作时，需要多加关注。

- directConnection 会将槽函数线程代码插入到信号发射线程中执行；

- queuedConnection 会让槽函数线程代码留在自己的线程中执行，而不会跳到信号发射线程那里去跑

  > 下面举例一些需要手动指定连接类型的场景：
  >
  > 例1-跨多个线程：
  >
  > A线程中写connect，让B线程中的信号连到C线程的槽中，希望C的槽在C中执行。
  >
  > 这种情况要明确指定QueuedConnection，不写的话按照Auto处理，C中的槽会在A中执行。
  >
  > 例2-跨线程DirectConnection
  >
  > (这种用法在Qml的渲染引擎SceneGraph中比较常见)。
  >
  > A线程为内部代码，不能修改，一些特定的节点会有信号发出。
  >
  > B线程为用户代码，有一些功能函数，希望在A线程中去执行。
  >
  > 这种情况，将A的信号连接到B的函数，连接方式指定为DirectConnection，就可以把B的函数插入到A线程发信号的地方了。
  >
  > 效果类似于子类重载父类的函数。

  

## Thread 和 Signal-Slot

> 在 Qt 的编程模型里，thread 和 signal- slot 都是实现异步编程的手段，signal-slot的背后是消息循环，而每个thread都是一个独立的消息循环。
>
> 理想的 Qt 编程模型是独立的一个个任务，各自使用自己的 thread 和 thread 内部消息循环，在需要互相通讯的时候，使用 signal-slot ，这些 signal 和 slot 就是明确定义的消息接口，除此之外，最好不共享其它状态。



