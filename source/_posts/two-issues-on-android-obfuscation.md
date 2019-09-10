---
title: Android代码混淆问题处理两则
date: 2019-09-10 21:33:28
tags: [Android, ProGuard]
---

代码混淆是Android开发的必经之路，尤其是SDK开发，启用混淆一定程度上加大了制品被逆向工程的难度，同时也能减小制品体积。

然而代码混淆之路并非一帆风顺，而且往往在运行到目标代码之前，无法确定混淆是否出了问题。本文记载两则代码混淆中遇到的问题和处理方案。

本文提及的代码可以在[https://github.com/perqin/ProGuardBugTest](https://github.com/perqin/ProGuardBugTest)找到。

## ProGuard处理可选依赖错误

在我维护的SDK项目中的一个版本中，我需要支持多个SDK之间不强制依赖。Base模块为基础库；Core和GUI模块均依赖Base库，但GUI模块对Core的依赖是可选的。

在实现上，我让Web对Core的依赖为`compileOnly`：

```groovy
// gui/build.gradle
apply plugin: 'java-library'

dependencies {
    implementation project(':base')
    compileOnly project(':core')
}

sourceCompatibility = "7"
targetCompatibility = "7"

```

在Base模块中有`Account`类，在Core模块中则有一个接口使用了该类：

```java
package com.example.core;

import com.example.base.Account;

public interface AccountStore {
    Account getAccount();
}

```

接下来，GUI模块中有一段对Core模块的选择性调用逻辑：

```java
package com.example.gui;

import com.example.core.AccountStore;
import com.example.core.CoreMain;

public class GuiMain {
    private static final boolean HAS_CORE;
    static {
        boolean hasCore;
        try {
            new CoreMain();
            hasCore = true;
        } catch (NoClassDefFoundError e) {
            hasCore = false;
        }
        System.out.println("hasCore: " + hasCore);
        HAS_CORE = hasCore;
    }

    private AccountStore accountStore = null;

    public void setAccountStore(AccountStore accountStore) {
        this.accountStore = accountStore;
    }

    public String tryGetAccountName() {
        if (HAS_CORE) {
            if (accountStore != null) {
                return accountStore.getAccount().name;
            }
        }
        return "";
    }
}
```

该类通过Core中才有的`CoreMain`类来判断可选依赖的类是否存在，`tryGetAccountName`中进行了判断，如果Core模块不存在，必定不会引用到`AccountStore`接口，也不应该会出现任何问题。

接下来，我们将Base和Gui分别打包为jar并给app模块引用，注意Core未被引用。

然后我们**启用代码混淆**，注意添加以下规则避免混淆失败：

```
# ProGuard rule file
-dontwarn com.example.**
```

随后在`gradle.properties`中禁用R8（原因后面会提到）：

```properties
android.enableR8=false
```

最后我们进行简单的调用：

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        System.out.println("account: " + new Account());
        System.out.println("account name: " + new GuiMain().tryGetAccountName());
    }
}
```

不出意外的话，上面这段人畜无害的代码在Android 8.0会翻车：

```
2019-09-10 21:56:10.327 3667-3667/? E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.example.proguardbugtest, PID: 3667
    java.lang.VerifyError: Verifier rejected class com.example.proguardbugtest.MainActivity: void com.example.proguardbugtest.MainActivity.onCreate(android.os.Bundle) failed to verify: void com.example.proguardbugtest.MainActivity.onCreate(android.os.Bundle): [0x3D] cannot access instance field java.lang.String com.example.a.a.a from object of type Unresolved Reference: com.example.base.Account (declaration of 'com.example.proguardbugtest.MainActivity' appears in /data/app/com.example.proguardbugtest-5HQbNJwMRzeKoIg1ZzDmjA==/base.apk)
        at java.lang.Class.newInstance(Native Method)
        at android.app.Instrumentation.newActivity(Instrumentation.java:1173)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2708)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2892)
        at android.app.ActivityThread.-wrap11(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1593)
        at android.os.Handler.dispatchMessage(Handler.java:105)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6541)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:767)
```

在谷歌搜索了很久，对于`VerifyError`这个简单的错误很难找到明确的答案，只能拔出Android Studio大宝剑，分析apk字节码一探究竟。

我将`MainActivity`的`onCreate`的完整字节码粘贴到下面，其中`########`开头的是我补充的注释：

```
.method public onCreate(Landroid/os/Bundle;)V
    .registers 5

    invoke-super {p0, p1}, Landroidx/appcompat/app/c;->onCreate(Landroid/os/Bundle;)V

    const p1, 0x7f0a001c

    invoke-virtual {p0, p1}, Lcom/example/proguardbugtest/MainActivity;->setContentView(I)V

    sget-object p1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    new-instance v0, Ljava/lang/StringBuilder;

    const-string v1, "account: "

    invoke-direct {v0, v1}, Ljava/lang/StringBuilder;-><init>(Ljava/lang/String;)V

    ######## 1
    new-instance v1, Lcom/example/a/a;

    invoke-direct {v1}, Lcom/example/a/a;-><init>()V

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/Object;)Ljava/lang/StringBuilder;

    invoke-virtual {v0}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0

    invoke-virtual {p1, v0}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    sget-object p1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    new-instance v0, Ljava/lang/StringBuilder;

    const-string v1, "account name: "

    invoke-direct {v0, v1}, Ljava/lang/StringBuilder;-><init>(Ljava/lang/String;)V

    new-instance v1, Lcom/example/b/a;

    invoke-direct {v1}, Lcom/example/b/a;-><init>()V

    sget-boolean v2, Lcom/example/b/a;->a:Z

    if-eqz v2, :cond_40

    iget-object v2, v1, Lcom/example/b/a;->b:Lcom/example/core/AccountStore;

    if-eqz v2, :cond_40

    iget-object v1, v1, Lcom/example/b/a;->b:Lcom/example/core/AccountStore;

    ######## 2
    invoke-interface {v1}, Lcom/example/core/AccountStore;->getAccount()Lcom/example/base/Account;

    move-result-object v1

    iget-object v1, v1, Lcom/example/a/a;->a:Ljava/lang/String;

    goto :goto_42

    :cond_40
    const-string v1, ""

    :goto_42
    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v0}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0

    invoke-virtual {p1, v0}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    return-void
.end method
```

可以看到，「1」处注释的`new-instance`指令显然是要构造`Account`对象，从这里我们也可以得知`com.example.base.Account`被混淆成了`com.example.a.a`，这个也可以在ProGuard生成的mapping文件中中得到证实。

然而，「2」处注释的`invoke-interface`却出现了`com/example/base/Account`！之所以这段代码出现在了MainActivity中，是因为ProGuard进行了优化，但已经被混淆的类却以原名的形式出现在这里，这不科学。

好吧，就算这样，但我们的Core包并不存在，也不会执行到这里，不应该出现崩溃才对，而这也是为什么我前面特别提到Android 8.0了。经过验证，这段代码在Android 8.0上必定崩溃，但在我的Android Q上却稳如老狗，可以认为不同版本的Android对类的校验逻辑不同，Android 8.0的校验显然过分严格。

问题的真相调查清楚，接下来就是解决了，我们当然可以直接将`Account`加入到`-keep`规则中，但这只是治标不治本。

另一个选择则是放弃辣鸡ProGuard，转向R8。R8是谷歌开发的混淆、优化工具，用以代替ProGuard，从Android 3.4开始成为默认的混淆工具，我们只需要将前面提到的“禁用R8”的开关去掉即可。但是要注意，R8目前也未必稳定，就在不久前，我就遇到过R8对枚举类型的混淆错误导致崩溃的情况；同时R8的一些默认行为也和ProGuard不同，例如它会默认将未被`-keep`的类的包名进行重定向，即`com.example.app.Core`在ProGuard中可能会被混淆为`com.example.a.a`，但在R8中会被混淆为`a.a.a.a`，这些需要注意。

至于我，思前想后，还是觉得这样的可选依赖有些歪门邪道，风险难以控制，干脆让Gui强制依赖Core了（摊手

## `Serializable`对象混淆后不可用

接下来的问题会在Android 7.0上翻车。

在开发中，我们有一个数据结构需要在两个Activity之间传递，于是我们直接让它实现了`Serializable`接口，简单粗暴地丢进了Intent里，谁知后来测试同学就反馈SDK在启动该Activity时在魅族手机上闪退了。

由于~~贫穷的~~测试组同学只反馈了这部魅族手机有问题，其他手机正常，所以我一开始以为魅族又魔改系统用力过猛，好在后来自己新建了一个Android 7.0的模拟器，竟然复现了这个问题，才有机会找到问题的真正原因。

首先，我们有这样一段简单的数据结构的代码：

```java
package com.example.proguardbugtest;

import java.io.Serializable;

class SerializableMeta implements Serializable {
    private static final long serialVersionUID = -7822771794489130246L;

    private String nickname;
    private int age;

    SerializableMeta(String nickname, int age) {
        this.nickname = nickname;
        this.age = age;
    }

    public String getNickname() {
        return nickname;
    }

    public int getAge() {
        return age;
    }
}
```

以及这样一段启动Activity的代码：

```java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.startButton).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainActivity.this, MainActivity.class);
                intent.putExtra("meta", new SerializableMeta("Perqin", 233));
                startActivity(intent);
            }
        });
    }
}

```


然后，我们再简单地配置代码混淆规则：
```
-keepclassmembers class * implements java.io.Serializable {
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}
```

以上规则来自ProGuard的官方文档，但**后面就会发现这里是有问题的**。

最后，我们简单地启动，简单地点击按钮，然后：


```
2019-09-10 22:56:02.456 6516-6516/com.example.proguardbugtest E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.example.proguardbugtest, PID: 6516
    java.lang.InternalError
        at java.io.ObjectStreamClass.<init>(ObjectStreamClass.java:509)
        at java.io.ObjectStreamClass.lookup(ObjectStreamClass.java:354)
        at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1165)
        at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:346)
        at android.os.Parcel.writeSerializable(Parcel.java:1521)
        at android.os.Parcel.writeValue(Parcel.java:1474)
        at android.os.Parcel.writeArrayMapInternal(Parcel.java:723)
        at android.os.BaseBundle.writeToParcelInner(BaseBundle.java:1408)
        at android.os.Bundle.writeToParcel(Bundle.java:1133)
        at android.os.Parcel.writeBundle(Parcel.java:763)
        at android.content.Intent.writeToParcel(Intent.java:8655)
        at android.app.ActivityManagerProxy.startActivity(ActivityManagerNative.java:3052)
        at android.app.Instrumentation.execStartActivity(Instrumentation.java:1518)
        at android.app.Activity.startActivityForResult(Activity.java:4224)
        at android.app.Activity.startActivityForResult(Activity.java:4183)
        at android.app.Activity.startActivity(Activity.java:4507)
        at android.app.Activity.startActivity(Activity.java:4475)
        at com.example.proguardbugtest.MainActivity$1.onClick(Unknown Source)
        at android.view.View.performClick(View.java:5610)
        at android.view.View$PerformClick.run(View.java:22265)
        at android.os.Handler.handleCallback(Handler.java:751)
        at android.os.Handler.dispatchMessage(Handler.java:95)
        at android.os.Looper.loop(Looper.java:154)
        at android.app.ActivityThread.main(ActivityThread.java:6077)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:866)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:756)
```

这个崩溃甚至只有类名，连异常消息都不给了。如果此时去分析字节码，是不会发现什么异常的。

于是，我根据错误去读源码，以下是抛出异常的代码（地址：[http://androidxref.com/7.0.0_r1/xref/libcore/ojluni/src/main/java/java/io/ObjectStreamClass.java#509](http://androidxref.com/7.0.0_r1/xref/libcore/ojluni/src/main/java/java/io/ObjectStreamClass.java#509)）：

```java
    private ObjectStreamClass(final Class<?> cl) {
        this.cl = cl;
        name = cl.getName();
        isProxy = Proxy.isProxyClass(cl);
        isEnum = Enum.class.isAssignableFrom(cl);
        serializable = Serializable.class.isAssignableFrom(cl);
        externalizable = Externalizable.class.isAssignableFrom(cl);

        Class<?> superCl = cl.getSuperclass();
        superDesc = (superCl != null) ? lookup(superCl, false) : null;
        localDesc = this;

        if (serializable) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    if (isEnum) {
                        suid = Long.valueOf(0);
                        fields = NO_FIELDS;
                        return null;
                    }
                    if (cl.isArray()) {
                        fields = NO_FIELDS;
                        return null;
                    }

                    suid = getDeclaredSUID(cl);
                    try {
                        fields = getSerialFields(cl);
                        computeFieldOffsets();
                    } catch (InvalidClassException e) {
                        serializeEx = deserializeEx =
                            new ExceptionInfo(e.classname, e.getMessage());
                        fields = NO_FIELDS;
                    }

                    if (externalizable) {
                        cons = getExternalizableConstructor(cl);
                    } else {
                        cons = getSerializableConstructor(cl);
                        writeObjectMethod = getPrivateMethod(cl, "writeObject",
                            new Class<?>[] { ObjectOutputStream.class },
                            Void.TYPE);
                        readObjectMethod = getPrivateMethod(cl, "readObject",
                            new Class<?>[] { ObjectInputStream.class },
                            Void.TYPE);
                        readObjectNoDataMethod = getPrivateMethod(
                            cl, "readObjectNoData", null, Void.TYPE);
                        hasWriteObjectData = (writeObjectMethod != null);
                    }
                    writeReplaceMethod = getInheritableMethod(
                        cl, "writeReplace", null, Object.class);
                    readResolveMethod = getInheritableMethod(
                        cl, "readResolve", null, Object.class);
                    return null;
                }
            });
        } else {
            suid = Long.valueOf(0);
            fields = NO_FIELDS;
        }

        try {
            fieldRefl = getReflector(fields, this);
        } catch (InvalidClassException ex) {
            // field mismatches impossible when matching local fields vs. self
            throw new InternalError();
        }

        if (deserializeEx == null) {
            if (isEnum) {
                deserializeEx = new ExceptionInfo(name, "enum type");
            } else if (cons == null) {
                deserializeEx = new ExceptionInfo(name, "no valid constructor");
            }
        }
        for (int i = 0; i < fields.length; i++) {
            if (fields[i].getField() == null) {
                defaultSerializeEx = new ExceptionInfo(
                    name, "unmatched serializable field(s) declared");
            }
        }
    }
```

追溯`getReflector`下去，最终我们定位到抛出异常的地方：

```java
private static ObjectStreamField[] matchFields(ObjectStreamField[] fields,
                                                   ObjectStreamClass localDesc)
        throws InvalidClassException
    {
        ObjectStreamField[] localFields = (localDesc != null) ?
            localDesc.fields : NO_FIELDS;

        /*
         * Even if fields == localFields, we cannot simply return localFields
         * here.  In previous implementations of serialization,
         * ObjectStreamField.getType() returned Object.class if the
         * ObjectStreamField represented a non-primitive field and belonged to
         * a non-local class descriptor.  To preserve this (questionable)
         * behavior, the ObjectStreamField instances returned by matchFields
         * cannot report non-primitive types other than Object.class; hence
         * localFields cannot be returned directly.
         */

        ObjectStreamField[] matches = new ObjectStreamField[fields.length];
        for (int i = 0; i < fields.length; i++) {
            ObjectStreamField f = fields[i], m = null;
            for (int j = 0; j < localFields.length; j++) {
                ObjectStreamField lf = localFields[j];
                if (f.getName().equals(lf.getName())) {
                    if ((f.isPrimitive() || lf.isPrimitive()) &&
                        f.getTypeCode() != lf.getTypeCode())
                    {
                        throw new InvalidClassException(localDesc.name,
                            "incompatible types for field " + f.getName());
                    }
                    if (lf.getField() != null) {
                        m = new ObjectStreamField(
                            lf.getField(), lf.isUnshared(), false);
                    } else {
                        m = new ObjectStreamField(
                            lf.getName(), lf.getSignature(), lf.isUnshared());
                    }
                }
            }
            if (m == null) {
                m = new ObjectStreamField(
                    f.getName(), f.getSignature(), false);
            }
            m.setOffset(f.getOffset());
            matches[i] = m;
        }
        return matches;
    }
```

可以看到，对`fields`和`localFields`进行比较的时候，当两个成员变量中至少有一个是基本类型，并且他们的变量名相同但类型不同的时候，就会抛出异常。

此时，我们再次拔出大宝剑，会看到`SerializableMeta`的两个成员变量名字是相同的：

![](fields-with-the-same-name.png)

查阅这个类在Android 6.0的代码（[http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/io/ObjectStreamClass.java](http://androidxref.com/6.0.1_r10/xref/libcore/luni/src/main/java/java/io/ObjectStreamClass.java)）后发现，这个类在旧版本中甚至没有`matchFields`方法，应该是进行了较大的重构或更新；而查阅Android 7.1的代码（[http://androidxref.com/7.1.1_r6/xref/libcore/ojluni/src/main/java/java/io/ObjectStreamClass.java](http://androidxref.com/7.1.1_r6/xref/libcore/ojluni/src/main/java/java/io/ObjectStreamClass.java)），会发现谷歌爸爸很快就修复了这个问题：

```java
// ...
// Android-changed: We can have fields with a same name and a different type.
if (f.getName().equals(lf.getName()) &&
  f.getSignature().equals(lf.getSignature())) {
  if (lf.getField() != null) {
// ...
```

实际上，Android和JVM都允许成员变量名称相同，混淆为尽可能相同的名字可以减小体积并增大阅读难度。

但**实际上ProGuard默认并不会启用如此激进的混淆方式**（写这篇文章的时候我艰难地尝试复现而不断失败……）。在我的GitHub代码上会看到我在混淆的规则中藏了一条：

```
-overloadaggressively
```

查阅文档之后就会知道，启用这个选项后，才会导致成员变量被混淆为同样的名字。

继续搜索，发现[这篇帖子](https://sourceforge.net/p/proguard/bugs/274/)，有人反馈了这个问题，作者表示：我并不太想给JRE的错误实现擦屁股，在代码里硬编码绕过这个问题（I'm not really eager to hard-code a workaround for what looks like a bug in the JRE implementation of serialization, but I'll consider it.）。

最后是对于该问题的解决，移除`-overloadaggressively`当然有效，但如果这个规则来自某个依赖库的Consumer ProGuard规则的话，就不那么好用了。而另一个方法是使用`-useuniqueclassmembernames`，这个选项会避免生成同名成员，但带来的副作用就是增大体积，另外，[这篇帖子](https://sourceforge.net/p/proguard/bugs/413/)也提到这个选项不只是刚好关掉了`-overloadaggressively`，还有其他副作用。

因此，最后我选择了RTFM：重新看了一下ProGuard官方文档中[关于Serializable的规则建议](https://www.guardsquare.com/en/products/proguard/manual/examples#serializable)。使用它提供的完整规则后，会发现Serializable中的变量都被保留并不再导致问题出现了。