---
layout:     post
title:      "Android style & Theme 再探析"
subtitle:   " \"你真的懂Style和Theme\""
date:       2018-08-15 20:25:00
author:     "wanghao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Android
    - 知识记录
    
---

> Style 和 Theme 在初学Android都是被各种视频一笔带过，对其理解就是style可以类似html的css 可以继承，可以各种地方直接引用做到直接管控，而theme更是一笔带过，然而当我学到了一个android 国外的style 和 Theme课程才发现，一切并非如此......


**Android Theme目前的应用情况**
一般情况下，我了解到的和平常搜索到的应用场景，一般情况都以处理window的标题栏和背景透明一类的内容

```java
private void dealLogin(final Button button) {
        button.setOnClickListener(new OnClickListener() {

            @Override
            public void onClick(View v) {
                    if (userID == null || userID.trim().length() == 0) {
                        DialogUtils.showDialog(LoginAbstractActivity.this, "输入错误", "用户名不能为空");
                    } else if (passWord == null || passWord.trim().length() == 0) {
                        DialogUtils.showDialog(LoginAbstractActivity.this, "输入错误", "密码不能为空");
                    } 
            }
        });
    }
```
类似这种敏感页面的hardcode，就算在进行代码混淆也无法将中文混淆掉，很容易被hacker推断出当前处于怎样的界面，就像我们的登录，仔细阅读，很容易让hacker得到登录的破解逻辑！

所以，小伙伴们我们的偷懒的hardcode有时也许不止是国际化不容易，甚至可能造成可怕的安全问题哦！

2.防截屏防止录屏
一些中毒的手机会被在后台开启截屏或者录屏操作，所以我们在一些敏感的例如支付或者登陆界面，建议一定开始防止录屏设置

```java
    public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //开启设置
    getWindow().setFlags(LayoutParams.FLAG_SECURE, LayoutParams.FLAG_SECURE);

    setContentView(R.layout.main);
  }
```
3. 清单文件的
> android:allowBackup = false

保证软件中的用户数据不会被通过adb restore被盗取出来

4. 登陆界面使用透明UI覆盖，进行钓取用户数据
![透明UI覆盖](/img/in-post/post-android-hacker/hack-interface-hoding.JPG)

解决方案：
A.启动一个service时刻检查栈顶的activity的包名
非自己的app的弹窗提示
B.在onActivityPaused使用handler发生一个message

```java
  final int mNum = getNextNum();
                Message message = Message.obtain();
                message.arg1 = mNum;
                h.sendMessageDelayed(message, 1200);
                Context context = activity.getApplicationContext();
```
在onActivityResumed中判断，是自己的界面就取消message
不是则弹窗处于后台警告

```java
 registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            private int num = 0;
            private Activity mActivity;
            private synchronized int getNextNum() {
                num++;
                return num;
            }

            private Handler h = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    if (mActivity != null && !mActivity.isFinishing() && msg.arg1 == num) {
                        Toast.makeText(mActivity, R.string.main_app_in_background_toast_message, Toast.LENGTH_SHORT).show();
                    }
                }
            };

            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

            }

            @Override
            public void onActivityStarted(Activity activity) {
               // countActivity++;
            }

            @Override
            public void onActivityResumed(Activity activity) {
                mActivity = activity;//UI挟持，避免有返回的时候有界面挟持
                getNextNum();
            }

            @Override
            public void onActivityPaused(Activity activity) {
                final int mNum = getNextNum();
                Message message = Message.obtain();
                message.arg1 = mNum;
                h.sendMessageDelayed(message, 1200);
                Context context = activity.getApplicationContext();
                context.startService(new Intent(context, AlarmService.class));
            }

            @Override
            public void onActivityStopped(Activity activity) {

            }

            @Override
            public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

            }

            @Override
            public void onActivityDestroyed(Activity activity) {
            }
        });

public class AlarmService extends Service {

    boolean isStart = false;
    Handler handler = new Handler();

    Runnable alarmRunnable = new Runnable() {
        @Override
        public void run() {
            if (!checkActivity(getApplicationContext())){
                Toast.makeText(getApplicationContext(), R.string.main_app_in_background_toast_message, Toast.LENGTH_LONG).show();
            }
        }
    };

    @Override
    public int onStartCommand(Intent intent, int flag, int startId) {
        super.onStartCommand(intent, flag, startId);
        if (!isStart) {
            isStart = true;
            //启动alarmRunnable
            handler.postDelayed(alarmRunnable, 1000);
            stopSelf();
        }
        return START_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * 检测当前Activity是否安全
     */
    public static boolean checkActivity(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {//获取系统api版本号,如果是5x系统就用这个方法获取当前运行的包名
            return myAppIsFront(context);
        }
        boolean safe = false;
        ActivityManager activityManager = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        String runningActivityPackageName = activityManager.getRunningTasks(1).get(0).topActivity.getPackageName();
        if (runningActivityPackageName != null) {
            if (runningActivityPackageName.equals(context.getPackageName())) {
                safe = true;
            }
        }
        return safe;
    }


    public static boolean myAppIsFront(Context context) {//5x系统以后利用反射获取当前栈顶activity的包名.
        Field field;
        int START_TASK_TO_FRONT = 2;
        try {
            field = ActivityManager.RunningAppProcessInfo.class.getDeclaredField("processState");//通过反射获取进程状态字段.
            ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
            List appList = am.getRunningAppProcesses();
            ActivityManager.RunningAppProcessInfo app;

            for (int i = 0; i < appList.size(); i++) {
                app = (ActivityManager.RunningAppProcessInfo) appList.get(i);
                if (app.importance == ActivityManager.RunningAppProcessInfo.IMPORTANCE_FOREGROUND) {//表示前台运行进程.
                    int state = 0;
                    try {
                        state = field.getInt(app);//反射调用字段值的方法,获取该进程的状态.
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    if (state != START_TASK_TO_FRONT) {//我的app前台进程是否在前台
                        return false;
                    }else {
                        return true;
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }
}

```

5. 本地数据xml以及数据库的数据是否有加密保护，尤其我们喜欢将用户数据进行缓存，这些都是非常危险。
解决方案：对本地敏感数据进行加密处理

6. app是否存在进程保护或者内存保护，高端的hacker会使用动态代码注入工具，以及dump工具dump客户端的内存
解决方案：检查root到直接关闭app(未找到根本性解决方案)

7. 运行时是否有敏感日志从控制台打印
解决方案：使用统一封装logger类，判断环境为生产环境关闭log

好了，今天就总结到这里了，但是更多的安全问题还未总结，希望大家期待本系列的第二篇