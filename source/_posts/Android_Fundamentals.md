---
title: Android Fundamentals V2 Learn points
date: 2018-09-28 19:39:49
---

There are some notable  knowledge point which been found when I reviewed the Fundamentals.

---

1. Activity manifest attribute：`android:parentActivityName=com.codeideal.demo.MainActivity`:  
this define a relationship between two activities for upward navigation. For old version(SDK_INT < 16 ), you can define a meta-data as follows:
 ```java
<activity android:name=".SecondActivity"
      android:label = "Second Activity"
      android:parentActivityName=".MainActivity">
      <meta-data
          android:name="android.support.PARENT_ACTIVITY"
          android:value="com.codeideal.demo.MainActivity" />
</activity>
```

There is a scene that when you click a notifcation to jump to a WebActivity to show promotion, and then you click back to back to the HomeActivity.

There are two situations :
  1.  The App is on, click notification
  2.  The App is off, click notification

In situations 1, we click back in the WebActivi ty, we don't need rebuild the back stack and directly jump to HomeActivity.

In situations 2, we need to rebuild the back stack to go back to the HomeActivity

with These config, we can write thoes code to auto rebuild back stack to go to the HomeActivity:
```java
@Override
public void onBackPressed() {
    // 获得指向父级activity的intent，NavUtils在support v4 包中
    Intent upIntent = NavUtils.getParentActivityIntent(this);
    // 判断是否需要重建任务栈
    if (NavUtils.shouldUpRecreateTask(this, upIntent)) {
        // 这个activity不是这个app任务的一部分, 所以当向上导航时创建
        // 用合成后退栈(synthesized back stack)创建一个新任务。
        TaskStackBuilder.create(this)
                // 添加这个activity的所有父activity到后退栈中
                .addNextIntentWithParentStack(upIntent)
                // 向上导航到最近的一个父activity
                .startActivities();
    } else {
        // 这个activity是这个app任务的一部分, 所以
        // 向上导航至逻辑父activity.
        NavUtils.navigateUpTo(this, upIntent);
    }
    super.onBackPressed();
}
```
