---
title: Weex控制Android返回键解决方案
date: 2016-06-30
tag: "weex"
---

> 正在深入倒腾weex，希望可以将weex用在项目中。这里找出了weex控制Android返回键的方法。

# 需求
项目使用的是Vue+VueRouter的单页应用来写Weex的，现有以下需求。

* 当页面不在首页上时，返回上一页面。 `this.$router.go(-1)`
*  当页面在首页是，关闭当前Android应用

# 解决方案
## Android和Weex的通信
### Android to Weex
使用的是[globalEvent](http://weex.apache.org/cn/references/modules/globalevent.html)来实现的。我们在 Android 的返回按钮事件中触发 `globalEvent`，在 Weex 中监听该 `globalEvent`。

**Android**
```
public void onBackPressed(){
  Map<String,Object> params=new HashMap<>();
  params.put("name","returnmsg");
  mWXSDKInstance.fireGlobalEventCallback("androidback",params);
}
```
**Weex**
```
globalEvent.addEventListener('androidback', function (e) {
  // 这里就可以做返回事件操作了，如返回上一页或退出应用
  // that.$router.go(-1)
  // weex.requireModule('close').closeApp()
})
```
### Weex to Android
而Weex对Android的通信使用[Module扩展](http://weex.apache.org/cn/references/advanced/extend-to-android.html#Module-扩展)来实现。通过在Android中创建WXModule并在Application中注册后，Weex调用该Module触发Android事件。下面我们来一步步实现。

**1. Android中创建CloseModule**
```
public class CloseModule extends WXModule {

    @JSMethod(uiThread = false)
    public void closeApp() {
        LogUtil.e("触发关闭效果");
        CacheActivity.finishActivity();
    }
}
```
**2. 在Application中注册Module**
```
public class WXApplication extends Application {

  @Override
  public void onCreate() {
    super.onCreate();
    InitConfig config = new InitConfig.Builder().setImgAdapter(new ImageAdapter()).build();
    WXSDKEngine.initialize(this, config);
    try {
      ...
      WXSDKEngine.registerModule("close", CloseModule.class);
      ...
    } catch (WXException e) {
      e.printStackTrace();
    }
  }
}
```
**3. 在Weex中使用**
```
weex.requireModule('close').closeApp()
```
这样调用Module之后就可以对Android做许多事情了。

## 退出Activity
这里我还遇到了一个问题，就是在Weex提供的WXModule中如何退出Activity，解决方案为[android 关闭多个或指定activity](http://www.cnblogs.com/jenson138/p/4516568.html),这篇文章让我可以非常优雅的管理我的Activity。简单写下用法.
**1. 在每个Activity的onCreate方法中将Activity对象添加到List中**
```
@Override
protected void onCreate(Bundle savedInstanceState) {
  ...
  CacheActivity.addActivity(NetworkActivity.this);
}
```
**2. 在Module中去关闭Activity**
```
CacheActivity.finishActivity();
```
**3. 当然，别忘了把CacheActivity的代码贴到项目中去**
```
package com.weex.sample.utlis;

import android.app.Activity;
import java.util.LinkedList;
import java.util.List;

public class CacheActivity {
    public static List<Activity> activityList = new LinkedList<Activity>();

    public CacheActivity() {

    }

    /**
     * 添加到Activity容器中
     */
    public static void addActivity(Activity activity) {
        if (!activityList.contains(activity)) {
            activityList.add(activity);
        }
    }

    /**
     * 便利所有Activigty并finish
     */
    public static void finishActivity() {
        for (Activity activity : activityList) {
            activity.finish();
        }
    }

    /**
     * 结束指定的Activity
     */
    public static void finishSingleActivity(Activity activity) {
        if (activity != null) {
            if (activityList.contains(activity)) {
                activityList.remove(activity);
            }
            activity.finish();
            activity = null;
        }
    }

    /**
     * 结束指定类名的Activity 在遍历一个列表的时候不能执行删除操作，所有我们先记住要删除的对象，遍历之后才去删除。
     */
    public static void finishSingleActivityByClass(Class<?> cls) {
        Activity tempActivity = null;
        for (Activity activity : activityList) {
            if (activity.getClass().equals(cls)) {
                tempActivity = activity;
            }
        }

        finishSingleActivity(tempActivity);
    }

}
```

Over！继续倒腾Weex中……遇到问题继续总结。欢迎留言交流~