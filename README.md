# Android海外发行

> 如果需要修改自定义SDK名称可以在 `settings.gradle/gradle.properties/local.properties` 之中修改

这里针对的是海外版本推广处理, 主要是 `Google Play` 游戏商城上架等事务.

首先必须要说明的是发行 `SDK` 除非官方已经强制要求升级, 否则尽量采用 `JDK1.8` 支持

> `JDK1.8` 是国内普遍应用的版本, 直接采用对接过程能够省下很多联调功夫

需要说明 `AGP(Android Gradle Plugin)` 需要选择对应选择兼容版本:

- `Gradle 7.0 → AGP 7.0.0+`
- `Gradle 7.1 → AGP 7.1.0+`
- `Gradle 7.2 → AGP 7.2.0+`
- `Gradle 7.3 → AGP 7.3.0+`
- `Gradle 7.4 → AGP 7.4.0+`
- `Gradle 7.5 → AGP 7.4.2+`

> 注意: Gradle 7.x 以上版本推荐构建环境为 JDK11, 而编译目标为 JDK1.8

这里按照自身需求选定打包插件(AGP):

```groovy
buildscript {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/jcenter' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        google()
        mavenCentral()
    }
    dependencies {
        // 使用与Gradle 7.x兼容的AGP版本
        // 注意: 采用Gradle 7.5版本
        //noinspection AndroidGradlePluginVersion,AndroidGradlePluginVersion
        classpath "com.android.tools.build:gradle:${AndroidBuildGradleToolsVersion}"
    }
}

allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/jcenter' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        google()
        mavenCentral()
    }
}

// 新版本命令注册写法, 注意好兼容性
tasks.register('clean', Delete) {
    delete rootProject.buildDir
}
```

而主要引入之后的输出编译配置则要求如下:

```groovy
apply plugin: 'com.android.library'


android {
    namespace AndroidPackageName
    compileSdk Integer.parseInt(AndroidCompileSdkVersion)

    defaultConfig {
        minSdk Integer.parseInt(AndroidMinSdkVersion)
        //noinspection OldTargetApi
        targetSdk Integer.parseInt(AndroidTargetSdkVersion)
        versionCode Integer.parseInt(AndroidVersionCode)
        versionName AndroidVersionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false // 关闭混淆, 由CP方统一处理
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    // 编译选项配置 - 目标为JDK 1.8
    compileOptions {
        // 源代码兼容性
        sourceCompatibility JavaVersion.VERSION_1_8
        // 目标代码兼容性
        targetCompatibility JavaVersion.VERSION_1_8
        // 启用desugar支持Java 8+特性在旧设备上运行
        coreLibraryDesugaringEnabled true
    }

    packagingOptions {
        exclude 'META-INF/DEPENDENCIES.txt'
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
        exclude 'META-INF/NOTICE'
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/DEPENDENCIES'
        exclude 'META-INF/notice.txt'
        exclude 'META-INF/license.txt'
        exclude 'META-INF/dependencies.txt'
        exclude 'META-INF/LGPL2.1'
        exclude 'META-INF/versions/9/module-info.class'
    }

    buildTypes {

        debug {
            // 调试开关：允许 Debugger 附加（必须为 true，否则无法调试）
            debuggable true
            // 代码混淆：关闭（混淆会破坏调试信息，开发阶段无需混淆）
            minifyEnabled false
            // 自定义 Debug 版本的版本名后缀（如 1.0.0-debug）
            versionNameSuffix "-debug"
            setArchivesBaseName "${rootProject.name}-${AndroidVersionName}"
        }


        release {
            // 调试开关：关闭（防止正式包被调试，保障安全性）
            debuggable false
            // 混淆规则文件：默认的 Android 优化规则 + 自定义的项目规则
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            // 自定义 Release 版本的版本名后缀（如 1.0.0-release）
            versionNameSuffix "-release"
            setArchivesBaseName "${rootProject.name}-${AndroidVersionName}"
        }
    }
}

dependencies {
    // 外部扩展库
    api fileTree(include: ['*.jar', '*.aar'], dir: 'extra')

    // AndroidX核心库
    implementation "androidx.core:core:1.17.0"
    implementation "androidx.appcompat:appcompat:1.7.1"

    // 支持Java 8+特性的desugar库
    coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:2.0.3'

    // 测试库
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.3.0'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
}
```

> `Google` 在 `AndroidSdk31` 添加新特性, 导致低于 `30` 版本会弹出 `android:attr/lStar` 异常

上面可以看到用到很多的外部常量, 这部分其实是习惯问题(不希望写死配置, 所以定义在 `gradle.properties` 文件中):

```properties
## For more details on how to configure your build environment visit
# http://www.gradle.org/docs/current/userguide/build_environment.html
#
# Specifies the JVM arguments used for the daemon process.
# The setting is particularly useful for tweaking memory settings.
# Default value: -Xmx1024m -XX:MaxPermSize=256m
# org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
#
# When configured, Gradle will run in incubating parallel mode.
# This option should only be used with decoupled projects. For more details, visit
# https://developer.android.com/r/tools/gradle-multi-project-decoupled-projects
# org.gradle.parallel=true
#Sat Oct 18 14:43:48 CST 2025
AndroidPackageName=com.easy.game.sdk
AndroidBuildGradleToolsVersion=7.4.2
AndroidCompileSdkVersion=31
AndroidMinSdkVersion=21
AndroidTargetSdkVersion=33
AndroidVersionCode=1
AndroidVersionName=1.0.0
android.enableJetifier=true
android.useAndroidX=true
```


在 `gradle.properties` 中通过 `org.gradle.java.home` 指定 JDK 11 路径,
这部分需要 `JDK11` 打包编译, 但是输出的采用 `JDK1.8` 保证了引入的安全兼容性

> Google Play 自 2023 年 8 月起要求新应用必须使用 `targetSdk 33+`, 所以要留意好版本上架的最低要求

而按照最小需要原则, 那么原则上生成的 `AndroidManifest.xml` 最简单版本如下:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.easy.game.sdk"
          android:versionCode="1"
          android:versionName="1.0.0">

    <!-- 基础网络权限 -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>

    <!-- 高版本不支持IMEI获取, 需要 OAID 替代方案权限 -->
    <uses-permission android:name="com.android.launcher.permission.READ_SETTINGS"/>

    <!-- 特殊权限：需在隐私政策说明用途 -->
    <uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
    <uses-permission android:name="android.permission.VIBRATE"/>

    <!-- 敏感权限 -->
    <uses-permission
            android:name="android.permission.READ_PHONE_STATE"
            android:maxSdkVersion="28"/>
    <uses-permission
            android:name="android.permission.READ_EXTERNAL_STORAGE"
            android:maxSdkVersion="28"/>
    <uses-permission
            android:name="android.permission.WRITE_EXTERNAL_STORAGE"
            android:maxSdkVersion="28"/>
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>


    <application>

        <!-- HTTP 兼容配置 -->
        <uses-library
                android:name="org.apache.http.legacy"
                android:required="false"/>

    </application>

</manifest>
```

之后我们就可以简单编写个单例的 `Manager` 方法, 让 `CP` 方调用回调即可.


> 另外工具类尽可能直接用 apache.common 相关工具就行, 不要去手动编写无意义的轮子代码

这里首先需要抽象应用层实现, 也就是实现自己的 `android.app.Application` 用于暴露给CP去继承

```java
import android.app.Application;
import android.content.Context;
import android.content.res.Configuration;

import androidx.annotation.NonNull;


/**
 * 重写 Android 应用类
 */
public class EasyGameApplication extends Application {


    /**
     * todo: 一般需要抽离出来单独的放置的抽象结构, 暴露给 CP 方做应用实现
     */
    public interface ApplicationHandler {
        default <T extends Application> void setOnCreate(T clazz){};

        default void setAttachBaseContext(Context context){};

        default void setOnTerminate(){};

        default void setOnConfigurationChanged(Configuration configuration){};

        default void setOnLowMemory(){}

        default void setOnTrimMemory(int level){}
    }

    protected ApplicationHandler handler;


    public EasyGameApplication() {
        // todo: 目前仅仅作为抽象出来对象实现
        this.handler = new ApplicationHandler(){

        };
    }

    /**
     * 应用创建
     */
    @Override
    public void onCreate() {
        super.onCreate();
        handler.setOnCreate(EasyGameApplication.this);
    }

    /**
     * 切换应用上下文
     *
     * @param base Android上下文
     */
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        handler.setAttachBaseContext(base);
    }

    /**
     * 应用退出
     */
    @Override
    public void onTerminate() {
        super.onTerminate();
        handler.setOnTerminate();
    }

    /**
     * 配置变动
     *
     * @param newConfig 应用配置变动
     */
    @Override
    public void onConfigurationChanged(@NonNull Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        handler.setOnConfigurationChanged(newConfig);
    }


    /**
     * 内存极度过低 - 低精度
     */
    @Override
    public void onLowMemory() {
        super.onLowMemory();
        handler.setOnLowMemory();
    }

    /**
     * 系统内存不足提示 - 高精度
     * @param level 系统不足提示
     */
    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        handler.setOnTrimMemory(level);
    }
}
```

需要注意的是, 如果在国内编写的SDK, 除了常规的 `Application` 接入还需要处理 `闪屏接入`.

> 海外发行闪屏不是必须, 实现闪屏反而会导致体验下降(包括国内的防沉迷和未成年验证)

之后就是窗口必须要去实现的生命周期配置:

```plain
// 首次创建 Activity 时触发
EasyGame.getInstance().onCreate(Activity activity);

// Activity 还在栈中时再次启动
EasyGame.getInstance().onNewIntent(Activity activity, intent);

// 当 Activity 进入“已开始”状态时
EasyGame.getInstance().onStart(Activity activity);

// Activity 不再位于前台
EasyGame.getInstance().onPause(Activity activity);

// Activity 从后台恢复 
EasyGame.getInstance().onRestart(Activity activity);

// Activity 会在进入“已恢复”状态时来到前台
EasyGame.getInstance().onResume(Activity activity);

// Activity 不再对用户可见，已进入“已停止”状态
EasyGame.getInstance().onStop(Activity activity);

// 销毁 Ativity 之前
EasyGame.getInstance().onDestroy(Activity activity);

// 获取 Activity 的结果
EasyGame.getInstance().onActivityResult(Activity activity, int requestCode, int resultCode, Intent data);

// 请求应用权限的结果
EasyGame.getInstance().onRequestPermissionsResult(Activity activity, int requestCode, String[] permissions, int[] grantResults);
```

这些都是要求实现的绑定接口传递, 所以这部分也编写成统一的回调接口处理:

```java
import android.app.Activity;
import android.content.Intent;

/**
 * 必须接入的生命周期回调
 * 这些接口都可以在 Activity 类之中查看到, 发行只需要关注常用到那些就行
 */
public interface ApplicationLifecycle {

    /**
     * 首次创建 Activity 时触发
     */
    void onCreate(Activity activity);

    /**
     * Activity 还在栈中时再次启动
     */
    void onNewIntent(Activity activity, Intent intent);

    /**
     * 当 Activity 进入 "已开始" 状态时
     */
    void onStart(Activity activity);

    /**
     * Activity 不再位于前台
     */
    void onPause(Activity activity);

    /**
     * Activity 从后台恢复
     */
    void onRestart(Activity activity);

    /**
     * Activity 会在进入 "已恢复" 状态时来到前台
     */
    void onResume(Activity activity);

    /**
     * Activity 不再对用户可见，已进入 "已停止" 状态
     */
    void onStop(Activity activity);

    /**
     * 销毁 Activity 之前
     */
    void onDestroy(Activity activity);

    /**
     * 获取 Activity 的结果
     */
    void onActivityResult(Activity activity, int requestCode, int resultCode, Intent data);

    /**
     * 请求应用权限的结果
     */
    void onRequestPermissionsResult(Activity activity, int requestCode, String[] permissions, int[] grantResults);
}
```

这些都是系统内部需要甚至的回调, 之后就是业务层面回调接口处理:

- `EasyGame.getInstance().init(Activity)`: 初始化SDK, 需要用户同意相关协议后再调用初始化接口
- `EasyGame.getInstance().login(Activity)`: 唤起登录, 直接启动用户登录页面接口, 允许玩家中止自动登录或切换账号
- `EasyGame.getInstance().report(Activity, [上报数据类])`: 数据上报, 用来监控玩家的行为日志
- `EasyGame.getInstance().logout(activity)`: 游戏注销, 相当于清理账号登录状态
- `EasyGame.getInstance().exit(activity)`: 退出应用, 清理账号登录状态并且退出当前应用
- `EasyGame.getInstance().pay(activity, [订单信息类], [玩家角色类])`: 唤起支付创建发行的支付订单

游戏比较特殊的就是需要 `打开WebView` 等接口, 这部分比较特殊所以暂且先留着只做比较通用部分.

