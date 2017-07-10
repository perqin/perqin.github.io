---
title: Kotlin将只读lambda表达式作为监听器使用的坑
date: 2017-06-27 16:04:03
tags: [kotlin, android]
---

谷歌爸爸在今年的I/O大会上公布Kotlin成为官方支持的Android开发语言，于是我也学习了一个，并试着用Kotlin写了一个监听剪切板的应用。谁知上架商店之后没几天就发现出现了玄学的bug。

我的应用有一个开关，可以开启或关闭一个Service，这个Service在开启的时候会把剪切板的监听器添加到`ClipboardManager`，而在停止的时候则会移除该监听器。奇怪的事情是，添加是可以的，移除却失败了。

百思不得其解之下，我开始查看`ClipboardManager`的源码，其中添加和移除监听器的源码如下：

```java
    public void addPrimaryClipChangedListener(OnPrimaryClipChangedListener what) {
        synchronized (mPrimaryClipChangedListeners) {
            if (mPrimaryClipChangedListeners.size() == 0) {
                try {
                    getService().addPrimaryClipChangedListener(
                            mPrimaryClipChangedServiceListener, mContext.getOpPackageName());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            mPrimaryClipChangedListeners.add(what);
        }
    }

    public void removePrimaryClipChangedListener(OnPrimaryClipChangedListener what) {
        synchronized (mPrimaryClipChangedListeners) {
            mPrimaryClipChangedListeners.remove(what);
            if (mPrimaryClipChangedListeners.size() == 0) {
                try {
                    getService().removePrimaryClipChangedListener(
                            mPrimaryClipChangedServiceListener);
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
        }
    }
```

而这个`mPrimaryClipChangedListeners`不过是个`ArrayList`而已。

如果排除了这是Android系统的bug，那么唯一的解释就是：移除监听器的时候传递的对象和一开始添加的监听器并不是同一个！（请自行脑补名侦探柯南BGM）

那么就让我们来看看这个Service被编译成什么样子了吧。下面是原来的Kotlin代码（省略了业务逻辑，只保留几个Log）：

```kotlin
class CopyListenerService : Service() {
    override fun onBind(p0: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        clipboardManager.addPrimaryClipChangedListener(onPrimaryClipChangedListener)
        return START_STICKY
    }

    override fun onDestroy() {
        super.onDestroy()
        clipboardManager.removePrimaryClipChangedListener(onPrimaryClipChangedListener)
    }

    val clipboardManager by lazy { getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager }
    val onPrimaryClipChangedListener = {
        Log.d(TAG, "Clip Item count: " + clipboardManager.primaryClip.itemCount)
        Unit
    }

    companion object {
        val TAG = "CopyListenerService"
    }
}
```

可以看到，我的`onPrimaryClipChangedListener`只是一个纯洁的lambda表达式而已。

我们点击菜单`Tools - Kotlin - Show Kotlin Bytecode`，右边会多出一个编辑器，我们点击那个编辑器顶部的`Decompile`按钮，左边的编辑器就多了一个`CopyListenerService.decompiled.java`的文件。由于是反编译的，所以非常丑而且还有很多IDE飘红，不过这不影响我们找出元凶。下面是反编译结果，巨长：

```java
// CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3.java
package com.perqin.copyshare;

import android.content.ClipboardManager.OnPrimaryClipChangedListener;
import kotlin.Metadata;
import kotlin.jvm.functions.Function0;
import kotlin.jvm.internal.Intrinsics;

@Metadata(
   mv = {1, 1, 6},
   bv = {1, 0, 1},
   k = 3
)
final class CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 implements OnPrimaryClipChangedListener {
   // $FF: synthetic field
   private final Function0 function;

   CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3(Function0 var1) {
      this.function = var1;
   }

   // $FF: synthetic method
   public final void onPrimaryClipChanged() {
      Intrinsics.checkExpressionValueIsNotNull(this.function.invoke(), "invoke(...)");
   }
}
// CopyListenerService.java
package com.perqin.copyshare;

import android.app.Service;
import android.content.ClipboardManager;
import android.content.Intent;
import android.content.ClipboardManager.OnPrimaryClipChangedListener;
import android.os.IBinder;
import android.util.Log;
import kotlin.Lazy;
import kotlin.LazyKt;
import kotlin.Metadata;
import kotlin.TypeCastException;
import kotlin.Unit;
import kotlin.jvm.functions.Function0;
import kotlin.jvm.internal.DefaultConstructorMarker;
import kotlin.jvm.internal.PropertyReference1Impl;
import kotlin.jvm.internal.Reflection;
import kotlin.reflect.KProperty;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

@Metadata(
   mv = {1, 1, 6},
   bv = {1, 0, 1},
   k = 1,
   d1 = {"\u00006\n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0018\u0002\n\u0002\b\u0005\n\u0002\u0018\u0002\n\u0002\u0010\u0002\n\u0002\b\u0003\n\u0002\u0018\u0002\n\u0000\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0010\b\n\u0002\b\u0005\u0018\u0000 \u00182\u00020\u0001:\u0001\u0018B\u0005¢\u0006\u0002\u0010\u0002J\u0014\u0010\u000e\u001a\u0004\u0018\u00010\u000f2\b\u0010\u0010\u001a\u0004\u0018\u00010\u0011H\u0016J\b\u0010\u0012\u001a\u00020\u000bH\u0016J\"\u0010\u0013\u001a\u00020\u00142\b\u0010\u0015\u001a\u0004\u0018\u00010\u00112\u0006\u0010\u0016\u001a\u00020\u00142\u0006\u0010\u0017\u001a\u00020\u0014H\u0016R\u001b\u0010\u0003\u001a\u00020\u00048FX\u0086\u0084\u0002¢\u0006\f\n\u0004\b\u0007\u0010\b\u001a\u0004\b\u0005\u0010\u0006R\u0017\u0010\t\u001a\b\u0012\u0004\u0012\u00020\u000b0\n¢\u0006\b\n\u0000\u001a\u0004\b\f\u0010\r¨\u0006\u0019"},
   d2 = {"Lcom/perqin/copyshare/CopyListenerService;", "Landroid/app/Service;", "()V", "clipboardManager", "Landroid/content/ClipboardManager;", "getClipboardManager", "()Landroid/content/ClipboardManager;", "clipboardManager$delegate", "Lkotlin/Lazy;", "onPrimaryClipChangedListener", "Lkotlin/Function0;", "", "getOnPrimaryClipChangedListener", "()Lkotlin/jvm/functions/Function0;", "onBind", "Landroid/os/IBinder;", "p0", "Landroid/content/Intent;", "onDestroy", "onStartCommand", "", "intent", "flags", "startId", "Companion", "production sources for module app"}
)
public final class CopyListenerService extends Service {
   @NotNull
   private final Lazy clipboardManager$delegate = LazyKt.lazy((Function0)(new Function0() {
      // $FF: synthetic method
      // $FF: bridge method
      public Object invoke() {
         return this.invoke();
      }

      @NotNull
      public final ClipboardManager invoke() {
         Object var10000 = CopyListenerService.this.getSystemService("clipboard");
         if(var10000 == null) {
            throw new TypeCastException("null cannot be cast to non-null type android.content.ClipboardManager");
         } else {
            return (ClipboardManager)var10000;
         }
      }
   }));
   @NotNull
   private final Function0 onPrimaryClipChangedListener = (Function0)(new Function0() {
      // $FF: synthetic method
      // $FF: bridge method
      public Object invoke() {
         this.invoke();
         return Unit.INSTANCE;
      }

      public final void invoke() {
         Log.d(CopyListenerService.Companion.getTAG(), "Clip Item count: " + CopyListenerService.this.getClipboardManager().getPrimaryClip().getItemCount());
      }
   });
   @NotNull
   private static final String TAG = "CopyListenerService";
   // $FF: synthetic field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.property1(new PropertyReference1Impl(Reflection.getOrCreateKotlinClass(CopyListenerService.class), "clipboardManager", "getClipboardManager()Landroid/content/ClipboardManager;"))};
   public static final CopyListenerService.Companion Companion = new CopyListenerService.Companion((DefaultConstructorMarker)null);

   @Nullable
   public IBinder onBind(@Nullable Intent p0) {
      return null;
   }

   public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
      ClipboardManager var10000 = this.getClipboardManager();
      CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 var10001 = new CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3;
      Function0 var10003 = this.onPrimaryClipChangedListener;
      if(this.onPrimaryClipChangedListener == null) {
         Object var10002 = null;
      } else {
         var10001.<init>(var10003);
      }

      var10000.addPrimaryClipChangedListener((OnPrimaryClipChangedListener)var10001);
      return 1;
   }

   public void onDestroy() {
      super.onDestroy();
      ClipboardManager var10000 = this.getClipboardManager();
      CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 var10001 = new CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3;
      Function0 var10003 = this.onPrimaryClipChangedListener;
      if(this.onPrimaryClipChangedListener == null) {
         Object var10002 = null;
      } else {
         var10001.<init>(var10003);
      }

      var10000.removePrimaryClipChangedListener((OnPrimaryClipChangedListener)var10001);
   }

   @NotNull
   public final ClipboardManager getClipboardManager() {
      Lazy var1 = this.clipboardManager$delegate;
      KProperty var3 = $$delegatedProperties[0];
      return (ClipboardManager)var1.getValue();
   }

   @NotNull
   public final Function0 getOnPrimaryClipChangedListener() {
      return this.onPrimaryClipChangedListener;
   }

   @Metadata(
      mv = {1, 1, 6},
      bv = {1, 0, 1},
      k = 1,
      d1 = {"\u0000\u0014\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000e\n\u0002\b\u0003\b\u0086\u0003\u0018\u00002\u00020\u0001B\u0007\b\u0002¢\u0006\u0002\u0010\u0002R\u0014\u0010\u0003\u001a\u00020\u0004X\u0086D¢\u0006\b\n\u0000\u001a\u0004\b\u0005\u0010\u0006¨\u0006\u0007"},
      d2 = {"Lcom/perqin/copyshare/CopyListenerService$Companion;", "", "()V", "TAG", "", "getTAG", "()Ljava/lang/String;", "production sources for module app"}
   )
   public static final class Companion {
      @NotNull
      public final String getTAG() {
         return CopyListenerService.TAG;
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}
```

接下来，我们来简单分析试试。

首先，我们的lambda监听器被转换成什么代码了呢？

```java
   @NotNull
   private final Function0 onPrimaryClipChangedListener = (Function0)(new Function0() {
      // $FF: synthetic method
      // $FF: bridge method
      public Object invoke() {
         this.invoke();
         return Unit.INSTANCE;
      }

      public final void invoke() {
         Log.d(CopyListenerService.Companion.getTAG(), "Clip Item count: " + CopyListenerService.this.getClipboardManager().getPrimaryClip().getItemCount());
      }
   });
```

我们发现，我们在监听器里的实现被转换成了一个`Function0`对象，具体的实现代码被放在了`invoke`方法里。

接下来，我们来看看我们添加监听器的代码被转换成的样子：

```java
   public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
      ClipboardManager var10000 = this.getClipboardManager();
      CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 var10001 = new CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3;
      Function0 var10003 = this.onPrimaryClipChangedListener;
      if(this.onPrimaryClipChangedListener == null) {
         Object var10002 = null;
      } else {
         var10001.<init>(var10003);
      }

      var10000.addPrimaryClipChangedListener((OnPrimaryClipChangedListener)var10001);
      return 1;
   }
```

可以看到，我们添加监听器的时候传递的对象是`var10001`，而它是一个`CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3`类的实例。这个类又是个什么鬼？我们接着看：

```java
final class CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 implements OnPrimaryClipChangedListener {
   // $FF: synthetic field
   private final Function0 function;

   CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3(Function0 var1) {
      this.function = var1;
   }

   // $FF: synthetic method
   public final void onPrimaryClipChanged() {
      Intrinsics.checkExpressionValueIsNotNull(this.function.invoke(), "invoke(...)");
   }
}
```

原来如此，这个名字巨长的类正是实现了`OnPrimaryClipChangedListener`的类，它持有一个`Function0`的引用，而它的`onPrimaryClipChanged`实现其实不过是调用这个对象的`invoke`方法。

现在再回去看看添加监听器的代码，我们就可以整理出这样的思路：

 * 首先把lambda里的实现语句封装到一个`Function0`类
 * 然后让这个Service拥有这个`Function0`类的实例，这也就对应Kotlin代码里的`val`变量定义
 * 需要添加监听器的时候，构造一个实现了监听器接口的`CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3`对象
 * 然后把我们的`Function0`对象丢进去
 * 最后把这个构造出来的对象作为真正的监听器丢进去

等等，好像有哪里不对……构造一个对象？！？让我们看看这行代码：

```java
      CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 var10001 = new CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3;
```

这是个局部变量啊！！！丢进去之后就拿不到它了啊！那我还怎么remove它呢？？

来看看移除监听器是怎么写的：

```java
   public void onDestroy() {
      super.onDestroy();
      ClipboardManager var10000 = this.getClipboardManager();
      CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3 var10001 = new CopyListenerServiceKt$sam$OnPrimaryClipChangedListener$15d0add3;
      Function0 var10003 = this.onPrimaryClipChangedListener;
      if(this.onPrimaryClipChangedListener == null) {
         Object var10002 = null;
      } else {
         var10001.<init>(var10003);
      }

      var10000.removePrimaryClipChangedListener((OnPrimaryClipChangedListener)var10001);
   }
```

咳咳，这么骚的吗？

所以，我们终于得出了结论：虽然我们的lambda表达式对应的是不变的`Function0`对象，但是每次从它得到的监听器却不是同一个监听器！

有了这个结论，也就有了一个解决方法：我们不要让`onPrimaryClipChangedListener`成为lambda表达式，而是一个正正经经的监听器匿名内部类对象，我们改改代码：

```kotlin
    val onPrimaryClipChangedListener = ClipboardManager.OnPrimaryClipChangedListener {
        Log.d(TAG, "Clip Item count: " + clipboardManager.primaryClip.itemCount)
        Unit
    }
```

额额，好像只是加了两个单词……接下来看看反编译的代码：

```java
   @NotNull
   private final OnPrimaryClipChangedListener onPrimaryClipChangedListener = (OnPrimaryClipChangedListener)(new OnPrimaryClipChangedListener() {
      public final void onPrimaryClipChanged() {
         Log.d(CopyListenerService.Companion.getTAG(), "Clip Item count: " + CopyListenerService.this.getClipboardManager().getPrimaryClip().getItemCount());
      }
   });

   public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
      this.getClipboardManager().addPrimaryClipChangedListener(this.onPrimaryClipChangedListener);
      return 1;
   }

   public void onDestroy() {
      super.onDestroy();
      this.getClipboardManager().removePrimaryClipChangedListener(this.onPrimaryClipChangedListener);
   }
```

这次，`this.onPrimaryClipChangedListener`终于不再是一个`Function0`对象，而是一个监听器了！

~~然而，过了那么久都没有人在应用商店反馈这个bug，我该高兴还是不高兴呢……~~