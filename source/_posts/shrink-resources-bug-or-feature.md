---
title: shrinkResources：这是bug还是feature？
date: 2017-06-18 11:52:23
tags: [android, proguard]
---

前两天在酷安上架了一个很简单的应用（[复制分享(com.perqin.copyshare)_0.0.2_Android应用_酷安网](http://www.coolapk.com/apk/com.perqin.copyshare)），感觉酷安对个人开发者还是非常友好的，活跃用户多、评论区一片祥和。后来有评论说安装包体积太大了，于是就打算处理一下这个问题了。

根据官方文档[Shrink Your Code and Resources | Android Studio](https://developer.android.com/studio/build/shrink-code.html)，我启用了ProGuard代码混淆：

```groovy
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

但是构建之后立刻就挂掉了！而如果将`shrinkResources`设置为`false`则可以正常启动。

于是，我开始找出元凶。

首先，我得拿到release构建运行时崩溃的log。编辑AndroidManifest.xml，在application标签加入如下属性：

```xml
  <application
    android:debuggable="true"
    ... />
  ...
```

然后编辑应用的build.gradle文件，在`android`标签禁用检查，否则会无法通过编译：

```groovy
  android {
    lintOptions {
      checkReleaseBuilds false
    }
  }
```

然后再编译运行release版的应用，就能看到崩溃的日志。

但是由于启用了代码混淆，你会看到一大堆abcd，因此需要反混淆。根据官方文档，ProGuard会生成一个mapping.txt，里面包含了所有混淆的对应表，Android SDK提供了反混淆工具，因此我执行如下命令：

```shell
~/.local/lib/android_sdk/tools/proguard/bin/retrace.sh ~/workspaces/CopyShare/CopyShare/app/build/outputs/mapping/release/mapping.txt ~/trace.txt > ~/deob_trace.txt
```

上述命令将保存在`~/trace.txt`中的崩溃日志反混淆之后输出到`~/debo_trace.txt`。

接下来我们打开输出的日志：

```
06-18 00:50:12.547 18537-18537/com.perqin.copyshare E/AndroidRuntime: FATAL EXCEPTION: main
      Process: com.perqin.copyshare, PID: 18537
      java.lang.RuntimeException: Unable to start activity ComponentInfo{com.perqin.copyshare/com.perqin.copyshare.SettingsActivity}: android.view.InflateException: Binary XML file line #25: Binary XML file line #1: Error inflating class x
          at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2762)
          at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2848)
          at android.app.ActivityThread.-wrap12(ActivityThread.java)
          at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1552)
          at android.os.Handler.dispatchMessage(Handler.java:102)
          at android.os.Looper.loop(Looper.java:154)
          at android.app.ActivityThread.main(ActivityThread.java:6324)
          at java.lang.reflect.Method.invoke(Native Method)
          at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:886)
          at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:776)
       Caused by: android.view.InflateException: Binary XML file line #25: Binary XML file line #1: Error inflating class x
       Caused by: android.view.InflateException: Binary XML file line #1: Error inflating class x
       Caused by: java.lang.ClassNotFoundException: Didn't find class "android.view.x" on path: DexPathList[[zip file "/data/app/com.perqin.copyshare-1/base.apk"],nativeLibraryDirectories=[/data/app/com.perqin.copyshare-1/lib/arm64, /system/lib64, /vendor/lib64]]
          at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:56)
          at java.lang.ClassLoader.loadClass(ClassLoader.java:380)
          at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
          at android.view.LayoutInflater.createView(LayoutInflater.java:609)
          at android.view.LayoutInflater.onCreateView(LayoutInflater.java:700)
          at com.android.internal.policy.PhoneLayoutInflater.onCreateView(PhoneLayoutInflater.java:68)
          at android.view.LayoutInflater.onCreateView(LayoutInflater.java:717)
          at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:785)
          at android.view.LayoutInflater.parseInclude(LayoutInflater.java:964)
          at android.view.LayoutInflater.rInflate(LayoutInflater.java:854)
          at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:821)
          at android.view.LayoutInflater.inflate(LayoutInflater.java:518)
          at android.view.LayoutInflater.inflate(LayoutInflater.java:426)
          at android.view.LayoutInflater.inflate(LayoutInflater.java:377)
          at android.support.v7.app.AppCompatDelegateImplV9.createSubDecor(Unknown Source)
          at android.support.v7.app.AppCompatDelegateImplV9.ensureSubDecor(Unknown Source)
          at android.support.v7.app.AppCompatDelegateImplV9.onCreate(Unknown Source)
                                                            findViewById
                                                            onConfigurationChanged
                                                            setContentView
                                                            setContentView
                                                            onSubDecorInstalled
                                                            onPanelClosed
                                                            onMenuItemSelected
                                                            onMenuModeChange
                                                            startSupportActionModeFromWindow
                                                            onKeyShortcut
                                                            dispatchKeyEvent
                                                            shouldInheritContext
                                                            callActivityOnCreateView
                                                            openPanel
                                                            initializePanelDecor
                                                            reopenMenu
                                                            closePanel
                                                            callOnPanelClosed
                                                            findMenuPanel
                                                            getPanelState
                                                            performPanelShortcut
          at android.support.v7.app.AppCompatActivity.findViewById(Unknown Source)
          at android.app.Activity$HostCallbacks.onFindViewById(Activity.java:7273)
          at android.app.BackStackRecord.configureTransitions(BackStackRecord.java:1303)
          at android.app.BackStackRecord.beginTransition(BackStackRecord.java:1024)
          at android.app.BackStackRecord.run(BackStackRecord.java:729)
          at android.app.FragmentManagerImpl.execPendingActions(FragmentManager.java:1578)
          at android.app.FragmentController.execPendingActions(FragmentController.java:371)
          at android.app.Activity.performStart(Activity.java:6776)
          at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2725)
          at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2848)
          at android.app.ActivityThread.-wrap12(ActivityThread.java)
          at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1552)
          at android.os.Handler.dispatchMessage(Handler.java:102)
          at android.os.Looper.loop(Looper.java:154)
          at android.app.ActivityThread.main(ActivityThread.java:6324)
          at java.lang.reflect.Method.invoke(Native Method)
          at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:886)
          at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:776)
06-18 00:50:12.547 18537-18537/com.perqin.copyshare D/AppTracker: App Event: crash
06-18 00:50:12.558 18537-18537/com.perqin.copyshare I/Process: Sending signal. PID: 18537 SIG: 9

```

根据上述日志，我发现，崩溃原因是在`android.support.v7.app.AppCompatDelegateImplV9`这个类的`createSubDecor`方法里面调用了`LayoutInflater.inflate`方法，结果找不到一个叫x的View。我们来看看这个`createSubDecor`方法（在Android Studio中按Ctrl键可以一路追踪每个类的实现）：

```java
    private ViewGroup createSubDecor() {
        TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);

        if (!a.hasValue(R.styleable.AppCompatTheme_windowActionBar)) {
            a.recycle();
            throw new IllegalStateException(
                    "You need to use a Theme.AppCompat theme (or descendant) with this activity.");
        }

        if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
            requestWindowFeature(Window.FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
            requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
        }
        mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
        a.recycle();

        // Now let's make sure that the Window has installed its decor by retrieving it
        mWindow.getDecorView();

        final LayoutInflater inflater = LayoutInflater.from(mContext);
        ViewGroup subDecor = null;


        if (!mWindowNoTitle) {
            if (mIsFloating) {
                // If we're floating, inflate the dialog title decor
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);

                // Floating windows can never have an action bar, reset the flags
                mHasActionBar = mOverlayActionBar = false;
            } else if (mHasActionBar) {
                /**
                 * This needs some explanation. As we can not use the android:theme attribute
                 * pre-L, we emulate it by manually creating a LayoutInflater using a
                 * ContextThemeWrapper pointing to actionBarTheme.
                 */
                TypedValue outValue = new TypedValue();
                mContext.getTheme().resolveAttribute(R.attr.actionBarTheme, outValue, true);

                Context themedContext;
                if (outValue.resourceId != 0) {
                    themedContext = new ContextThemeWrapper(mContext, outValue.resourceId);
                } else {
                    themedContext = mContext;
                }

                // Now inflate the view using the themed context and set it as the content view
                subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                        .inflate(R.layout.abc_screen_toolbar, null);

                mDecorContentParent = (DecorContentParent) subDecor
                        .findViewById(R.id.decor_content_parent);
                mDecorContentParent.setWindowCallback(getWindowCallback());

                /**
                 * Propagate features to DecorContentParent
                 */
                if (mOverlayActionBar) {
                    mDecorContentParent.initFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
                }
                if (mFeatureProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_PROGRESS);
                }
                if (mFeatureIndeterminateProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_INDETERMINATE_PROGRESS);
                }
            }
        } else {
            if (mOverlayActionMode) {
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_screen_simple_overlay_action_mode, null);
            } else {
                subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
            }

            if (Build.VERSION.SDK_INT >= 21) {
                // If we're running on L or above, we can rely on ViewCompat's
                // setOnApplyWindowInsetsListener
                ViewCompat.setOnApplyWindowInsetsListener(subDecor,
                        new OnApplyWindowInsetsListener() {
                            @Override
                            public WindowInsetsCompat onApplyWindowInsets(View v,
                                    WindowInsetsCompat insets) {
                                final int top = insets.getSystemWindowInsetTop();
                                final int newTop = updateStatusGuard(top);

                                if (top != newTop) {
                                    insets = insets.replaceSystemWindowInsets(
                                            insets.getSystemWindowInsetLeft(),
                                            newTop,
                                            insets.getSystemWindowInsetRight(),
                                            insets.getSystemWindowInsetBottom());
                                }

                                // Now apply the insets on our view
                                return ViewCompat.onApplyWindowInsets(v, insets);
                            }
                        });
            } else {
                // Else, we need to use our own FitWindowsViewGroup handling
                ((FitWindowsViewGroup) subDecor).setOnFitSystemWindowsListener(
                        new FitWindowsViewGroup.OnFitSystemWindowsListener() {
                            @Override
                            public void onFitSystemWindows(Rect insets) {
                                insets.top = updateStatusGuard(insets.top);
                            }
                        });
            }
        }

        if (subDecor == null) {
            throw new IllegalArgumentException(
                    "AppCompat does not support the current theme features: { "
                            + "windowActionBar: " + mHasActionBar
                            + ", windowActionBarOverlay: "+ mOverlayActionBar
                            + ", android:windowIsFloating: " + mIsFloating
                            + ", windowActionModeOverlay: " + mOverlayActionMode
                            + ", windowNoTitle: " + mWindowNoTitle
                            + " }");
        }

        if (mDecorContentParent == null) {
            mTitleView = (TextView) subDecor.findViewById(R.id.title);
        }

        // Make the decor optionally fit system windows, like the window's decor
        ViewUtils.makeOptionalFitsSystemWindows(subDecor);

        final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);

        final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
        if (windowContentView != null) {
            // There might be Views already added to the Window's content view so we need to
            // migrate them to our content view
            while (windowContentView.getChildCount() > 0) {
                final View child = windowContentView.getChildAt(0);
                windowContentView.removeViewAt(0);
                contentView.addView(child);
            }

            // Change our content FrameLayout to use the android.R.id.content id.
            // Useful for fragments.
            windowContentView.setId(View.NO_ID);
            contentView.setId(android.R.id.content);

            // The decorContent may have a foreground drawable set (windowContentOverlay).
            // Remove this as we handle it ourselves
            if (windowContentView instanceof FrameLayout) {
                ((FrameLayout) windowContentView).setForeground(null);
            }
        }

        // Now set the Window's content view with the decor
        mWindow.setContentView(subDecor);

        contentView.setAttachListener(new ContentFrameLayout.OnAttachListener() {
            @Override
            public void onAttachedFromWindow() {}

            @Override
            public void onDetachedFromWindow() {
                dismissPopups();
            }
        });

        return subDecor;
    }
```

上面的代码一共引用到了以下layout资源：

*  R.layout.abc_dialog_title_material
*  R.layout.abc_screen_toolbar
*  R.layout.abc_screen_simple_overlay_action_mode
*  R.layout.abc_screen_simple

解包aar文件之后查看这些xml文件，会发现他们有一个共同的特点：

```xml
        <include
            layout="@layout/abc_screen_content_include"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_weight="1"/>
```

好的，这个时候我们将构建的会崩溃的apk拖进Android Studio查看其内容，我们找到`res/layout/abc_screen_content_include.xml`：

![](apk-analyze.png)

至此，真相大白！可以看到，`shrinkResources`这个特性似乎还是有一点bug（总不能是feature吧？？！），没有正确识别`include`标签的引用，导致这个资源被认为是无用的，内容也被

```xml
<?xml version="1.0" encoding="utf-8"?>
<x />
```

所代替，所以运行的时候，当渲染这个资源的时候就会报找不到android.view.x类的错误。

比较不幸的是，即使按照官方说明将这个资源手动加入到keep列表中：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources
    xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/abc_screen_content_include"
    tools:ignore="PrivateResource" />
```

但构建出来的apk仍然删除了这个资源，所以我只能提了一个issue：[AppCompat V7 crash when shrinkResources is enabled [62744324] - Visible to Public - Issue Tracker](https://issuetracker.google.com/issues/62744324)，不过目前还没有任何回应……

第一次尝试代码混淆，没想到就遇到了这么个坑，真是刺激！