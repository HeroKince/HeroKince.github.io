---
title: Android 应用前后台切换监听
author: HeroKince
date: 2021-03-17 20:00:00 +0800
categories: [Android]
tags: [Android,Activity]
---

应用开发中有一种场景，在主页面点击HOME键退回到桌面，然后再回到应用主页。再次回到主页的时候，需要更新一些数据或者做一些操作。

目前有三种方案可以实现：

- 监听Home键事件
- 获取最上层Activity判断
- 注册Activity生命周期监听回调

虽然前两中方案可以实现需求，但是有着明显的缺点和问题。监听Home键无法保证每个机型都能正常监听，不同ROM会有兼容性问题；获取最上层Activity在6.0之后需要开启权限，而且还需要后台不短轮训判断，耗费性能。

好在Android 4.0以后提供了ActivityLifecycleCallbacks类，它可以监听到应用内所有Activity 的生命周期方法，通过观察主界面退到后台以及主界面跳转应用其他界面的区别，就可以分析出监听应用前后台切换的方案。在这个场景中ActivityLifecycleCallbacks方法回调日志如下：


```
1.主页面 -> 桌面 -> 主页面

生命周期如下：... --(点HOME键)--> 主页面#onPause --(重新打开应用)--> 主页面#onResume -> ...

2.子页面 -> 主页面

生命周期如下：... -> 子页面#onPause -> ... -> 主页面#onResume ->...
```


所以，只要知道上一次是主页面调用了onPause，这一次是主页面调用了onResume，那么就可以确定应用是从桌面或者其他APP返回。主要代码如下：


```
public class AppStatusCallbacks implements Application.ActivityLifecycleCallbacks {

private boolean mMainOnStoped = false;
private boolean mMainOnResumed = false;

@Override
public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

}

@Override
public void onActivityStarted(Activity activity) {

}

@Override
public void onActivityResumed(Activity activity) {
mMainOnResumed = (activity instanceof MainActivity);
if (mMainOnPaused && mMainOnResumed) {
   // 应用从桌面或者其他应用回来
  
}
}

@Override
public void onActivityPaused(Activity activity) {
 
}

@Override
public void onActivityStopped(Activity activity) {
     mMainOnStoped= (activity instanceof MainActivity);
}

@Override
public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

}

@Override
public void onActivityDestroyed(Activity activity) {

}
}
```

另外，应用一般会存储Activity的任务栈，可以通过判断当前任务栈数量，来进一步优化判断方案。当停留在主界面的时候，说明Activity的任务栈的数量是1。把这个逻辑加入到上面的判断中，这样代码的容错性会更强一些。完整代码如下：


```
public class AppStatusCallbacks implements Application.ActivityLifecycleCallbacks {

private boolean mMainOnStoped = false;
private boolean mMainOnResumed = false;
private int activityCount = 0;

@Override
public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

}

@Override
public void onActivityStarted(Activity activity) {
  activityCount++;
}

@Override
public void onActivityResumed(Activity activity) {
mMainOnResumed = (activity instanceof MainActivity);
if (mMainOnPaused && mMainOnResumed && activityCount == 1) {
   // 应用从桌面或者其他应用回来
  
}
}

@Override
public void onActivityPaused(Activity activity) {
}

@Override
public void onActivityStopped(Activity activity) {
     mMainOnStoped= (activity instanceof MainActivity);
     activityCount--;
}

@Override
public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

}

@Override
public void onActivityDestroyed(Activity activity) {

}
}
```

参考：

- [Android 实现监听应用从后台回到前台](https://juejin.cn/post/6844903473780097038)
- [Android前后台状态判断及锁屏操作](https://starry90.github.io/Blog//coding/Android前后台状态判断及锁屏操作.html)
- [Android 监听APP进入后台或切换到前台方案对比](https://www.cnblogs.com/zhujiabin/p/9336663.html)

