+++
title = 'Qt Thread Main Event Loop'
date = 2024-01-14T05:16:27+08:00
draft = false
+++

## 场景举例：

- 没有main()启动主线程，但希望用Qt的类和信号槽
- 不希望app.exec()阻塞主线程

比如我最近在制作一个Windows SDK，封装成dll，只暴露两个简单全局函数作为接口：

```C++
extern "C" __declspec(dllexport) int start(const char*, const char*);
extern "C" __declspec(dllexport) void stop();
```

没有主事件循环，我就无法使用信号槽机制，我甚至无法正常使用QTimer、QProcess等等

那如何启动主事件循环呢？

```C++
QCoreApplication app(argc, argv);

...

app.exec();
```


但我没有main()入口，怎么办？

线程中启动主事件循环就可以——主事件循环不一定要在主线程中启动哦

注意不能使用QThread，因为QThread的正常使用也依赖事件循环

使用 `std::thread`

---

## 我的实现：

1、定义全局对象

```C++
static MyObject* m = nullptr;
```

2、定义启动参数对象

```C++
struct InputArgs{
  int argc;
  char **argv;
};
```

3、获取启动参数——调用程序的可执行文件路径

```C++
WCHAR exePath[MAX_PATH+1];
String exeFullPath;
DWORD len = GetModuleFIleNameW(nullptr, exePath, MAX_PATH);    // #include <Windows.h>
if (len>0)
    exeFullPath = QString::fromWCharArray(exePath);
else
    return EXIT_FAILURE;

QFileInfo exeInfo(exeFullPath);
char* v[1];
// 解决中文路径或路径带空格的问题，两种方法选一个即可
1. 修改默认编码
QTextCodec::setCodecForLocale(QTextCodec::codecForName(“GBK”);
QByteArray ba = exeInfo.absoluteFilePath().toUtf8();    // 这个对象必须创建
v[0]=ba.data();
2. 利用std::string
std::string ba = exeInfo.absoluteFilePath().toStdString();
v[0]=const_cast<char*>(ba.c_str());

InputArgs args = {1, v};

// 启动线程，传入参数
std::thread appThread(StartAppThread, static_cast<void*>(&args));
appThread.detach();
```

4、 定义线程函数

```C++
void StartAppThread(void* threadArg)
{
    InputArgs* args = static_cast<struct InputArgs*>(threadArg);
    …
    QCoreApplication app(args->argc, args->argv);
    m = new MyObject(…);    // 在主事件循环里创建该对象
    QObject::connect(m, &MyObject::sigExit, &app, &QCoreApplication::exit, Qt::QueuedConnection);    // 这样通过在m中emit sigExit(int code);即可退出程序
    …
    app.exec();
}
```

5、 通过m退出线程

在需要退出时，调用 `m->sigExit(EXIT_SUCCESS)`，即可退出主事件循环，从而退出线程。

注意⚠️：sigExit后，m->deleteLater()就不起作用啦，要用 `delete m`; 才能正常调用析构函数！

注意⚠️：如果在m中有需要退出，但app.exec()还没有调用，也就是主事件循环还未启动，这时调用QCoreApplication::exit(EXIT_SUCCESS)或QCoreApplication::quit()是无法正常退出的，触发的是一个无操作，因为这两个方法的含义是：退出正在运行的事件循环。都还没有启动主事件循环，怎么退出呢？对吧。
