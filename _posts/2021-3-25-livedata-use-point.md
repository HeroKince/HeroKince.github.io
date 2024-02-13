---
title: Android LiveData使用注意事项
author: HeroKince
date: 2021-03-25 20:00:00 +0800
categories: [Android]
tags: [Android,LiveData]
---

关于LiveData是什么以及基本使用方式，请参考官方文档：https://developer.android.com/topic/libraries/architecture/livedata?hl=zh-cn。

简单来说，LiveData是一个可被观察的数据容器类。它将数据包装起来，使得数据成为“被观察者”，页面成为“观察者”。当ViewModel存放页面所需要的各种数据发生变化时，通过LiveData的方式实现对页面的通知，完成ViewModel与页面组件之间的通信。

那么在使用时发现有以下几个地方需要注意：

**1.回调通知**

LiveData的观察者会在每次进入活跃态时收到回调，比如从后台回到前台，界面重启等等。如果只想收到一次回调的话，[可以使用SingleLiveEvent](https://github.com/android/architecture-samples/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/SingleLiveEvent.java)。要注意SingleLiveEvent仅限于一个观察者。如果添加了多个则只会调用一个，并且不能保证哪一个。

**2.数据倒灌**

所谓数据倒灌是一种形象的说法，它是指先setValue/postValue，后调用observe(new Obs())，至此收到了回调。然后再调用observe(new anotherObs())，如果还能收到第一次的回调，也就是旧数据。解决方案可以参考开源项目：[UnPeek-LiveData](https://github.com/KunMinX/UnPeek-LiveData)。

**3.事件包装**

上面提到SingleLiveEvent仅限于一个观察者，如果需要多个观察者该如何处理呢，方案就是使用事件包装。定义一个数据包装器，内部判断事件是否消费了，被消费后则不再回调通知。代码如下：

```
/**
* 事件包装器，明确地管理事件是否已经被处理
* @param <T> ViewModel中的数据，比如： 
* MutableLiveData<LiveEventWrapper<String>>()
*/
public class LiveEventWrapper<T> {

    private T content;
    private boolean hasBeenHandled;

    public LiveEventWrapper(T content) {
        this.content = content;
    }

    /**
     * Returns the content and prevents its use again.
     */
    public T getContentIfNotHandled() {
        if (hasBeenHandled) {
            return null;
        } else {
            hasBeenHandled = true;
            return content;
        }
    }

    /**
     * Returns the content, even if it's already been handled.
     */
    public T peekContent() {
        return content;
    }

    public boolean isHasBeenHandled() {
        return hasBeenHandled;
    }

}
```

或者

```
import androidx.annotation.NonNull;
import androidx.lifecycle.LifecycleOwner;
import androidx.lifecycle.LiveData;
import androidx.lifecycle.Observer;

public class CleanLiveData<T> extends LiveData<T> {
    private boolean hasModified = false;

    @Override
    public void observe(@NonNull LifecycleOwner owner, @NonNull final Observer<? super T> observer) {
        super.observe(owner, new Observer<T>() {
            private boolean hasIntercept = false;

            @Override
            public void onChanged(T t) {
                if (!hasModified || hasIntercept) {
                    observer.onChanged(t);
                }
                hasIntercept = true;
            }
        });
    }

    @Override
    public void observeForever(@NonNull final Observer<? super T> observer) {
        super.observeForever(new Observer<T>() {
            private boolean hasIntercept = false;

            @Override
            public void onChanged(T t) {
                if (!hasModified || hasIntercept) {
                    observer.onChanged(t);
                }
                hasIntercept = true;
            }
        });
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
        hasModified = true;
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
        hasModified = true;
    }

}
```
总结来看，以上现象基本都是由于LiveData的粘性特性引发的，因此在使用LiveData的时一定要搞清楚它的概念和原理。