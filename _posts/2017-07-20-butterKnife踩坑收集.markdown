---
layout:     post
title:      "butterKnife踩坑收集"
subtitle:   " \"butterKnife踩坑收集\""
date:       2017-07-20 21:36:00
author:     "wanghao"
header-img: "img/butterknife.jpg"
tags:
    - Android
    - butterKnife
    - 笔记
    
---


在学习使用 **butterKnife** 框架遭遇问题：
AndroidStudio提示：
> **Warning:Using incompatible plugins for the annotation processing: android-apt. This may result in an unexpected behavior.**

#### **导致问题：**
此时使用bindView完全不起效果

```Java
public class MainActivity extends AppCompatActivity
implements IWeatherInfoPage {
    @BindView(R.id.tv_content)
    TextView tvContent;
    @BindView(R.id.btn_request_data)
    Button btnRequestData;
    @BindView(R.id.pgBar)
    ProgressBar progressBar;
    private static final String TAG = MainActivity.class.getSimpleName();
    @OnClick(R.id.btn_request_data)
    void onClick(View view){
        presenter.doLoadWeatherInfo("London");
    }
```
此时的Gradle配置为

**project:**

```Groovy
buildscript {
    repositories {
        mavenCentral() // add repository
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'
       classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
        classpath 'org.greenrobot:greendao-gradle-plugin:3.2.2'
        classpath 'com.jakewharton:butterknife-gradle-plugin:8.7.0'
    }
}
```

**app:**

```Groovy
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
apply plugin: 'org.greenrobot.greendao' 
//apply plugin: 'com.android.library'
apply plugin: 'com.jakewharton.butterknife'

 compile 'com.google.dagger:dagger:2.4'
 apt 'com.google.dagger:dagger-compiler:2.4'
    //java注解
compile 'org.glassfish:javax.annotation:10.0-b28'
compile 'com.orhanobut:logger:2.1.1'
compile 'org.greenrobot:greendao:3.2.2' // add library
compile 'com.facebook.stetho:stetho:1.5.0'//加入调试工具
compile 'com.facebook.stetho:stetho-okhttp3:1.5.0'//加入网络调试
compile 'com.jakewharton:butterknife:8.7.0'
annotationProcessor 'com.jakewharton:butterknife-compiler:8.7.0'
```

#### **导致原因：**
1.此时的Android Studio 更新为 2.3.3 version
2. Annotation Processing 已经集成为Android studio gradle 插件，所以在使用这个版本的gradle或者更高加载，不需要再使用任何其他插件，所以
>classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
>apply plugin: 'com.neenbedankt.android-apt'

可以被移除，而：
>apt 'com.google.dagger:dagger-compiler:2.4'
>annotationProcessor 'com.jakewharton:butterknife-compiler:8.7.0'

此类需要apt插件的部分，都可以改为：
>annotationProcessor 'com.google.dagger:dagger-compiler:2.4'
>annotationProcessor 'com.jakewharton:butterknife-compiler:8.7.0'

此时，重新编译问题便解决了！如果想要开启关闭annotationProcessor
可以通过as设置：
>Settings > Build, Execution, Deployment > Compiler > Annotation Processors

感谢stackoverflow帮助： [https://stackoverflow.com/questions/42632662/android-studio-warning-using-incompatible-plugins-for-the-annotation-processing](https://stackoverflow.com/questions/42632662/android-studio-warning-using-incompatible-plugins-for-the-annotation-processing) 
