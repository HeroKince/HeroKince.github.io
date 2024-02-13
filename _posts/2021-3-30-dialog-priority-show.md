---
title: Android 弹窗顺序&优先级控制
author: HeroKince
date: 2021-03-30 21:04:00 +0800
categories: [Android]
tags: [Android,Dialog]
---

一般在项目首页中，往往会有多个对话框需要弹出，比如活动弹窗、更新弹窗、评分弹窗等等，而且这些弹窗是有优先级顺序的。这些弹窗一般是通过接口请求后返回结果再显示的，如果只有几个弹窗还好处理，业务逻辑上判断一下先后显示就可以。如果有十几个或者更多，那么处理起来将非常麻烦，而且容易出现问题。

所以封装一个可以按照优先级顺序显示的弹窗功能就非常有必要，首先功能需求如下：

- 按优先级顺序阻塞式显示各种类型弹窗，默认从最高优先级开始显示
- 只有上一个高优先级弹窗显示完或者取消显示，下一个低优先级弹窗才可以显示 
- 指定显示某一个弹窗的前提是没有更高优先级的弹窗需要显示 
- 在显示一个弹窗之前需要判断是否能够或者需要显示
- 根据优先级去查找指定的弹窗，优先级相当于唯一ID
- 弹窗包括多种类型，Dialog、PopupWindow、Activity等等
- 显示过的弹窗可以根据业务逻辑再次显示
- 优先级相同的弹窗则按添加的顺序依次显示

接着开始编码去实现功能，先定一个枚举类，罗列出支持的弹窗类型，包括Dialog、PopupWindow、Activity等等。


```
public enum WindowType {

    DIALOG,
    POUPOWINDOW,
    TOAST,
    SNACKBAR,
    WIDGET,
    ACTIVITY,
    OTHERS

}
```
然后定义弹窗接口IWindow，它定义了弹窗的基本功能。

```
/**
* 窗口约定规则
*/
public interface IWindow {

    /**
     * 弹窗展示
     */
    void show(Activity activity);

    /**
     * 弹窗关闭
     */
    void dismiss();
    
    /**
     * 是否展示
     * @return
     */
    boolean isShowing();

    /**
     * 设置窗口关闭监听
     */
    void setOnWindowDismissListener(OnWindowDismissListener listener);

    /**
     * 设置窗口展示监听
     */
    void setOnWindowShowListener(OnWindowShowListener listener);

}
```
以及弹窗显示和关闭的监听接口，

```
/**
* 窗口关闭监听
*/
public interface OnWindowDismissListener {

    /**
     *
     */
    void onDismiss();

}

/**
* 窗口展示监听
*/
public interface OnWindowShowListener {

    void onShow();

}
```

接下来定义个包裹类WindowWrapper去封装弹窗相关的属性和状态，包括弹窗、优先级、能否显示、窗体类型等等，在处理弹窗显示逻辑时将会用到。


```
/**
* 窗口参数类
*/
public class WindowWrapper {

    /**
     * 窗口
     */
    private IWindow mWindow;

    /**
     * 优先级，值越大优先级越高
     */
    private int mPriority;

    /**
     * 是否满足show的条件
     */
    private boolean isCanShow;

    /**
     * 弹窗类型
     */
    private WindowType mWindowType;

    /**
     * 弹窗名称
     */
    private String mWindowName;

    private WindowWrapper(Builder builder) {
        mWindow = builder.window;
        mPriority = builder.priority;
        mWindowType = builder.windowType;
        isCanShow = builder.isCanShow;
        mWindowName = builder.windowName;
    }

    public IWindow getWindow() {
        return mWindow;
    }

    public void setWindow(IWindow window) {
        this.mWindow = window;
    }

    public int getPriority() {
        return mPriority;
    }

    public void setPriority(int priority) {
        this.mPriority = priority;
    }

    public WindowType getWindowType() {
        return mWindowType;
    }

    public void setWindowType(WindowType mWindowType) {
        this.mWindowType = mWindowType;
    }

    public boolean isCanShow() {
        return isCanShow;
    }

    public void setCanShow(boolean canShow) {
        isCanShow = canShow;
    }

    public String getWindowName() {
        return mWindowName;
    }

    public void setWindowName(String mWindowName) {
        this.mWindowName = mWindowName;
    }

    public static class Builder {

        /**
         * 窗口
         */
        private IWindow window;

        /**
         * 优先级，值越大优先级越高
         */
        private int priority;

        /**
         * 弹窗类型
         */
        private WindowType windowType;

        /**
         * 是否满足show的条件
         */
        private boolean isCanShow;

        /**
         * 弹窗名称
         */
        private String windowName;

        public Builder window(IWindow window) {
            this.window = window;
            return this;
        }

        public Builder priority(int priority) {
            this.priority = priority;
            return this;
        }

        public Builder windowType(WindowType type) {
            this.windowType = type;
            return this;
        }

        public Builder setCanShow(boolean canShow) {
            isCanShow = canShow;
            return this;
        }

        public String getWindowName() {
            return windowName;
        }

        public Builder setWindowName(String windowName) {
            this.windowName = windowName;
            return this;
        }

        public WindowWrapper build() {
            return new WindowWrapper(this);
        }
    }
}
```
最后通过WindowTaskManager类去统一组织管理弹窗的添加、显示、关闭等逻辑，

```
public class WindowTaskManager {
    private List<WindowWrapper> mWindows;


    private static WindowTaskManager mDefaultInstance;


    private WindowTaskManager() {
    }


    /**
     * 获取弹窗管理者
     */
    public static WindowTaskManager getInstance() {
        if (mDefaultInstance == null) {
            synchronized (WindowTaskManager.class) {
                if (mDefaultInstance == null) {
                    mDefaultInstance = new WindowTaskManager();
                }
            }
        }
        return mDefaultInstance;
    }


    /**
     * 添加弹窗
     *
     * @param windowWrapper 待显示的弹窗
     */
    public synchronized void addWindow(Activity activity, WindowWrapper windowWrapper) {
        if (windowWrapper != null) {
            if (mWindows == null) {
                mWindows = new ArrayList<>();
            }


            if (windowWrapper.getWindow() != null) {
                windowWrapper.getWindow().setOnWindowShowListener(new OnWindowShowListener() {
                    @Override
                    public void onShow() {
                        windowWrapper.setShowing(true);
                    }
                });


                windowWrapper.getWindow().setOnWindowDismissListener(new OnWindowDismissListener() {
                    @Override
                    public void onDismiss() {
                        windowWrapper.setShowing(false);
                        mWindows.remove(windowWrapper);
                        showNext(activity);
                    }
                });
            }


            mWindows.add(windowWrapper);
        }
    }


    /**
     * 弹窗满足展示条件
     *
     * @param priority
     */
    public synchronized void enableWindow(Activity activity, int priority, IWindow window) {
        WindowWrapper windowWrapper = getTargetWindow(priority);
        if (windowWrapper != null) {


            if (windowWrapper.getWindow() == null) {
                window.setOnWindowShowListener(new OnWindowShowListener() {
                    @Override
                    public void onShow() {
                        windowWrapper.setShowing(true);
                    }
                });


                window.setOnWindowDismissListener(new OnWindowDismissListener() {
                    @Override
                    public void onDismiss() {
                        windowWrapper.setShowing(false);
                        mWindows.remove(windowWrapper);
                        showNext(activity);
                    }
                });
            }


            windowWrapper.setCanShow(true);
            windowWrapper.setWindow(window);
            show(activity, priority);
        }
    }


    /**
     * 移除不需要显示弹窗
     *
     * @param priority
     */
    public synchronized void disableWindow(int priority) {
        WindowWrapper windowWrapper = getTargetWindow(priority);
        if (windowWrapper != null && windowWrapper.getWindow() != null) {
            if (mWindows != null) {
                mWindows.remove(windowWrapper);
            }
        }
    }


    /**
     * 展示弹窗
     * 从优先级最高的Window开始显示
     */
    public synchronized void show(Activity activity) {
        WindowWrapper windowWrapper = getMaxPriorityWindow();
        if (windowWrapper != null && windowWrapper.isCanShow()) {
            IWindow window = windowWrapper.getWindow();
            if (window != null) {
                window.show(activity);
            }
        }
    }


    /**
     * 显示指定的弹窗
     *
     * @param priorities
     */
    public synchronized void show(Activity activity, int priorities) {
        WindowWrapper windowWrapper = getTargetWindow(priorities);
        if (windowWrapper != null && windowWrapper.getWindow() != null) {
            WindowWrapper topShowWindow = getShowingWindow();
            if (topShowWindow == null) {
                int priority = windowWrapper.getPriority();
                WindowWrapper maxPriorityWindow = getMaxPriorityWindow();
                if (maxPriorityWindow != null && windowWrapper.isCanShow() && priority >= maxPriorityWindow.getPriority()) {
                    if (windowWrapper.getWindow() != null) {
                        windowWrapper.getWindow().show(activity);
                    }
                }
            }
        }
    }


    /**
     * 清除弹窗管理者
     */
    public synchronized void clear() {
        if (mWindows != null) {
            for (int i = 0, size = mWindows.size(); i < size; i++) {
                if (mWindows.get(i) != null) {
                    IWindow window = mWindows.get(i).getWindow();
                    if (window != null) {
                        window.dismiss();
                    }
                }
            }
            mWindows.clear();
        }
        WindowHelper.getInstance().onDestroy();
    }


    /**
     * 清除弹窗管理者
     *
     * @param dismiss 是否同时dismiss掉弹窗管理者维护的弹窗
     */
    public synchronized void clear(boolean dismiss) {
        if (mWindows != null) {
            if (dismiss) {
                for (int i = 0, size = mWindows.size(); i < size; i++) {
                    if (mWindows.get(i) != null) {
                        IWindow window = mWindows.get(i).getWindow();
                        if (window != null) {
                            window.dismiss();
                        }
                    }
                }
            }
            mWindows.clear();
        }
        WindowHelper.getInstance().onDestroy();
    }


    /**
     * 展示下一个优先级最大的Window
     */
    private synchronized void showNext(Activity activity) {
        WindowWrapper windowWrapper = getMaxPriorityWindow();
        if (windowWrapper != null && windowWrapper.isCanShow()) {
            if (windowWrapper.getWindow() != null) {
                windowWrapper.getWindow().show(activity);
            }
        }
    }


    /**
     * 获取当前栈中优先级最高的Window（优先级相同则返回后添加的弹窗）
     */
    private synchronized WindowWrapper getMaxPriorityWindow() {
        if (mWindows != null) {
            int maxPriority = -1;
            int position = -1;
            for (int i = 0, size = mWindows.size(); i < size; i++) {
                WindowWrapper windowWrapper = mWindows.get(i);
                if (i == 0) {
                    position = 0;
                    maxPriority = windowWrapper.getPriority();
                } else {
                    if (windowWrapper.getPriority() >= maxPriority) {
                        position = i;
                        maxPriority = windowWrapper.getPriority();
                    }
                }
            }
            if (position != -1) {
                return mWindows.get(position);
            } else {
                return null;
            }
        }
        return null;
    }


    private synchronized WindowWrapper getTargetWindow(int priority) {
        if (mWindows != null) {
            for (int i = 0, size = mWindows.size(); i < size; i++) {
                WindowWrapper windowWrapper = mWindows.get(i);
                if (windowWrapper != null && windowWrapper.getPriority() == priority) {
                    return windowWrapper;
                }
            }
        }
        return null;
    }


    /**
     * 获取当前处于show状态的弹窗
     */
    private synchronized WindowWrapper getShowingWindow() {
        if (mWindows != null) {
            for (int i = 0, size = mWindows.size(); i < size; i++) {
                WindowWrapper windowWrapper = mWindows.get(i);
                if (windowWrapper != null && windowWrapper.getWindow() != null && windowWrapper.getWindow().isShowing()) {
                    return windowWrapper;
                }
            }
        }
        return null;
    }

}
```
WindowTaskManager类有三个主要方法：

- addWindow(Activity activity, WindowWrapper windowWrapper)
- enableWindow(Activity activity, int priority, IWindow window)
- disableWindow(int priority)
- setBlockTask(boolean show)

需要按顺序显示的对话框统一使用addWindow方法添加，这是还未进行网络请求之前就要调用的。作用是告诉WindowTaskManager一共有多少个弹窗需要按顺序显示。当网络请求返回之后，如果需要显示弹窗就调用enableWindow方法去显示，如果不需要显示弹窗就调用disableWindow方法，将这个弹窗从显示队列中移除。

另外，有时候需要阻塞弹窗的显示，比如引导开启系统权限后，会显示系统的权限授权界面，等授权结束后再继续显示其他弹窗。这时调用setBlockTask设置弹窗任务是否暂停或者继续。

以上就是按顺序显示弹窗的主要逻辑，使用的话窗体先继承IWindow，实现相关方法。然后通过操作WindowTaskManager类就可以了。代码仅供阅读，具体使用方法参见项目。

项目地址：https://github.com/Geekince/PriorityWindow

**彩蛋：**

需要在DialogFragment中显示DialogFragment时候，最好不要直接在DialogFragment启动显示，而是在DialogFragment的消失回调中启动显示。因为当前一个DialogFragment消失的时候，getChildFragmentManager可能会失效，应该在外层使用getFragmentManager。