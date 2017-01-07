---
title: Tumoji开发记录 (1)
date: 2017-01-08 00:52:12
tags: [tumoji, android, mvp]
thumbnail: /images/covers/tumoji-develop-blogpost-1.png
---

这篇博客算是一个多月的Tumoji开发的总结报告（当然也的确被小修改之后作为了大作业的个人总结，毕竟我懒）。截止这篇博客撰写的时候，Tumoji仅仅是填上了课堂展示时挖的坑，虽然的确可以使用了，但是距离能够正式发布还有很多坑没有填上。

由于开发过程中对遇到的问题没有及时记录，所以本文是想到什么写什么……

本项目的客户端和服务器端源码托管在GitHub：

*  客户端：[zhengqi-big-god-take-me-fly/Tumoji-Android: Android client for Tumoji.](https://github.com/zhengqi-big-god-take-me-fly/Tumoji-Android)
*  服务器：[zhengqi-big-god-take-me-fly/Tumoji-Server: Server side for Tumoji.](https://github.com/zhengqi-big-god-take-me-fly/Tumoji-Server)

## 起源

Tumoji是我这学期的Android开发课程的大作业，由我们这个开发组4个人，经过接近两个月的时间，共同完成。最开始的时候，我们本来想设计一个匿名聊天的应用，不过后来我对隐私性有所顾虑所以放弃了。后来突然想起来之前曾经用过一个叫*斗图神器*的app，因此萌生了这个想法。当然，发现*斗图神器*的时间已不可考，所以整个开发过程都是围绕我自己所确定的需求，应该不会有抄袭这回事吧，哈哈。

总之呢，Tumoji的定位是一款表情包共享应用，大家可以从上面下载表情包，也可以上传自己的表情包。

## MVP

### 简介

说起来我也算是Android的老油条了，最开始接触Android开发应该是在高考毕业的暑假，不过当时对Android的开发毫无了解，所以是下载了几个G的视频，然后没日没夜地看，艰难地敲出了Hello World。我想大部分编程初学者都有过这种感觉，照着教程写出了Hello World，虽然成功运行，但是整个代码里只有`Hello World`是你知道用来干嘛的，其他部分就不知道目的了。再后来，由于没什么定力，一直没有继续认真学，导致直到现在，Android的开发水平还是渣渣……

卖完情怀，开始说MVP。很久以前无意之间了解到了MVP，但是由于自己水平有限，一直看不懂网上的MVP教程。直到这学期，无意间看到一篇博客，突然看懂了一些，然后又去看了一下Google的MVP示范（[googlesamples/android-architecture at todo-mvp](https://github.com/googlesamples/android-architecture/tree/todo-mvp/)），然后直接在某一周的Android开发每周作业上用上了MVP。很庆幸在每周的作业中有几次MVP的尝试，终于慢慢了解了MVP的基本框架，再经过创新赛的时候的运用，终于熟悉起来了，于是在这次项目的开发中，我作为组长，一开始就确立了MVP的开发模式，到现在看来，这个模式在多人开发中绝对大有裨益。

在MVP之前，我们的代码往往全部堆积在Activity类中。即使打算抽离出逻辑，往往也仅仅是创建一些操作数据的类，但是UI的交互几乎全部都写在Activity中，再加上Activity本身的生命周期，这会导致Activity变成万能对象（God Object），想想在几百行代码里面找bug的感觉吧。在涉及多人写作开发的时候，问题就更加严重了。多人开发我所知道的模式大概是两种：分模块开发和分层开发，前者类似与我负责登录注册、你负责下载上传这样，而后者类似于我负责UI交互，你负责数据库存取这样。显然，在全部代码都堆积在Activity中的时候，是难以通过后面这种方式开发的，因为负责UI、数据、交互逻辑等部分的代码都混杂在一个文件里，多人同时修改的话冲突很严重。而我认为后面的开发方式是更加合理的，因为每个人的习惯、风格不同，如果按照模块开发的话，很容易出现好几个人分别各自造轮子的情况，而相对来说，每一层之间的耦合度都是比较低的，所以可以分别按照各自的习惯开发。

接下来的内容是我对MVP模式的理解了。在MVP模式中，V代表视图，M是模型，P是呈现者，这就把应用进行了分层。M一般指代的是数据层，包括数据库、文件读写、网络访问等等，他们往往是多个界面共享的（比如主界面和个人资料界面都会需要展示用户的信息）。V一般指的是一个视图单位，也就是指一个能够提供模块化的UI交互的界面，比如一个Activity、一个Fragment甚至一个Dialog。P即Presenter，我一开始的时候并不明白为什么要叫“Presenter”，后来终于明白了：一个View往往对应一个Presenter，而Presenter的作用就是控制View的变化。我们可以类比为PPT演讲，View就是PPT，而演讲者就是Presenter，Model当然就是计算机里的PPT数据啦，这样一来，就很容易理解：Presenter从Model获取数据，然后根据获取到的数据决定如何更改View的内容。我简单地画了一幅画：

````
                                User
                                | ^
                                v |
+-------------------+  +-------------------+  +-------------------+
|     Login View    |  |     Posts View    |  | Post Content View |
+-------------------+  +-------------------+  +-------------------+
       a | ^                    | ^                    | ^
         v | d                  v |                    v |
+-------------------+  +-------------------+  +-------------------+
|       Login       |  |       Post        |  |    Post Content   |
|     Presenter     |  |     Presenter     |  |     Presenter     |
+-------------------+  +-------------------+  +-------------------+
       b | ^                    | ^                    | ^
         v | c                  v |                    v |
+-----------------------------------------------------------------+
|      SQLite                  File                    API        |
|     Database               Storage                 Service      |
+-----------------------------------------------------------------+

a. Dispatch user's input
b. Get data from Model layer
c. Send data to Presenter layer
d. Update view content
````

从上图可以看出，用户的每一个操作都会被View传递给Presenter，Presenter从Model层获取对应的数据，并借此转交数据给View，View再将数据更新到UI上。

我们可以发现MVP的两个特点：数据层是共享的、Presenter将Model和View完全隔离。数据层的共享是理所当然的，因为从本质上来说，一个app只是一堆数据的部分的呈现而已，而这些数据本身是自洽的，即使没有View和Presenter，Model里的数据和数据间关系都是有意义的。而Presenter将Model和View完全隔离带来的好处就是保证数据的操作都来自于Presenter，View只负责分发动作和接受新内容。

在使用MVP的过程中，我还发现，Presenter和Model承载了不同种类的逻辑。何为不同种类的逻辑？举个例子来说，当我们创建一个Post的时候，下次获取PostCount时就应该比原来多1，这是数据层的逻辑，他们在Model中实现，这是为了保证不论外部（Presenter）发来怎样的调用，我都能保证我的数据在逻辑上是自洽的，不会出现诸如Post的作者id不存在、PostCount和真实Post数量不一致等问题；另一类逻辑则是UI上的逻辑，比如点击按钮A之后，如果没有登录就打开登录界面，否则打开编辑界面，这部分逻辑在Presenter中实现。

### 如何搭建MVP框架

当然，如果我不会MVP，我看到这里还是不知道MVP怎么写，因为前面说的都是MVP这个模式是怎么把一个app肢解的，但是具体要怎么写呢？按照Google的例子，我这样设计了我们的项目工程的包组织：

*  `com.tumoji.tumoji`
   *  `account`: 帐号模块，包括个人资料界面、登录注册界面
      *  `activity`: Activity类
      *  `adapter`: 各种Adapter
      *  `contract`: MVP中定义接口的契约类
      *  `fragment`: 各种Fragment
      *  `presenter`: 各种Presenter的实现
      *  `view`: 各种View的实现
   *  `common`: 通用的类、模块
   *  `data`: 数据层的表层实现，供Presenter使用，里面的每个子模块结构和`user`模块都是类似的
      *  `auth`: 登录、注册、token存取
      *  `meme`: 表情
      *  `settings`: 设置项存取
      *  `tag`: 标签
      *  `user`: 用户
         *  `model`: 存储各种数据对象的模型
            *  `UserModel`
         *  `repository`: 用于user数据操作的单例类
            *  `IUserRepository`: 暴露的接口
            *  `MockUserRepository`: 用来进行Presenter测试的`IUserRepository`实现
            *  `UserRepository`: 实现user数据操作的`IUserRepository`实现
         *  `store`
   *  `memes`: 表情模块，如表情列表、表情详细信息等，内部的结构和前面的`account`是类似的
   *  `network`: 网络模块，数据层中网络部分的定义和实现
   *  `storage`: 本地存储模块，数据层中SQLite数据库、SharedPreferences等的实现
   *  `utils`: 工具类
   *  `TumojiApp.java`: Application类

接下来我说说上面的目录结构是如何一步一步搭建起来的。

首先，对于某个模块里的某个界面，比如account模块的个人资料界面，我们需要分析：

*  我们会提供给用户哪些操作——Presenter的接口
*  每个操作会在UI上产生哪些变化——View的接口

有趣的是，用户的操作却是Presenter的接口，而操作的结果却是View的接口。事实上，由于Presenter和View成对出现，每个Presenter会持有自己的View的引用，每个View也持有自己的Presenter的引用。于是，用户的操作会被View获知，然后根据操作调用Presenter的对应方法；而Presenter得到数据之后，则需要根据数据来决定View如何更新，即调用View的哪个方法。

多说无益，我们以个人资料为例：

*  用户的操作
   *  点击性别切换按钮（意图更改自己的性别……）
*  UI的变化
   *  性别一栏切换至另一性别
   *  提示切换失败（比如说没有网络连接导致）

上面的分析是自然而然的，一个操作可能导致两个结果，于是我们可以这样设计一个契约类：

````java
public interface ProfileContract {
  interface Presenter {
    void changeGender(bool isMale);
  }
  
  interface View {
    void refreshGender(bool isMale);
    void showNetworkError();
  }
}
````

在`changeGender`的实现中，我们调用数据层的相关方法，向服务器发送请求更改个人性别资料，如果成功了，就调用`refreshGender`方法，否则调用`showNetworkError`。而对于View的实现而言，他只需要在`refreshGender`中设置某个RadioGroup的被选中项，在`showNetworkError`的实现中弹出一个Toast即可，而不需要考虑什么时候会被调用。

定义了契约类之后，我们就可以开始实现了。实现View的一般是Fragment或Activity。一般而言，用Fragment实现是官方推荐的方法，因为我前面说过，MVP中View代表的是一个视图单位，而一个Activity可能有多个视图单位，比如在平板上，可能左栏是一个列表，右栏是详情页。另一个好处是Activity可以成为一个总的管理者，负责Presenter和View的实例化（嗯，这回真的成为God了）。

而Presenter的实现，直接创建一个类实现对应Presenter接口即可，如前所述，在Activity中可以实例化一个Presenter对象。

由于View由Fragment实现，因为更新视图元素的任务就能轻松完成了。但是Presenter需要和数据层打交道，所以我们接下来需要考虑repository了。

repository顾名思义，就是数据仓库，在Google的例子中，每种数据都有各自的仓库（如Meme、User、Tag等等），并对外提供操作这类数据的方法。这里我并没有和Google用同样的设计，而是做了一些修改。Google的例子中，一个Repository对象内部持有两个DataSource的引用，这两个DataSource分别负责网络和本地数据的操作，然后由Repository进行判断什么时候从哪里拿数据，以及做一些缓存。事实上我也是这么做的，但是Google让Repository和DataSource这三个对象都实现IDataSource接口，我觉得这个有些过度设计了，因为有不少数据操作是仅限于网络或本地的，Google的抽象方法虽然合理，但是实际使用却会造成不方便。因此，我仅创建了IRepository接口让Repository去实现，而两个DataSource（在我的项目中叫做Store）是直接各自提供方法给Repository的。事实上，我特意让Repository去实现一个接口，和前面特意设计Presenter和View的接口的原因是一样的，方便增删接口和多人合作，这一点后面我会提到。

Repository内部的实现我就不再细说，我想从源码包组织中已经能看出不少了。

### MVP框架搭建中的坑、提示和MVP框架的演进

设计契约类是个脑力活，这要求我一开始就想好这个界面的最终效果，还要考虑到很多方面，我在实际的设计中也遇到了一些问题。

首先就是Dialog、BottomSheetDialogFragment等的问题。在Tumoji中，主界面主体是一个ViewPager，可以左右滑动两个分页，每个分页各有一个表情列表，是不同的排序。而点击某个表情，会弹出一个BottomSheet，展示表情的详情。最开始的时候，我试图把这个BottomSheet也作为整个主界面的View，但是事实上，这是很蛋疼的一件事，因为本身主界面承担的用户操作就已经够多了，而表情详情又要承担很多操作（点赞、举报、刷新、下载），这样会有十几个接口，百行代码找bug的痛苦又要来了！不仅如此，Android的Fragment是个很恶心的玩意儿，现在你还要在一个Fragment里面操作另一个Fragment，从那里获知用户操作以及把新数据分发过去……简直痛不欲生！

于是，我做了一个英明的决定，把详情页独立为一个视图单位，为它设计契约。这样，代码终于被分拆开来，能看了许多，而展开这个详情页的时候，主界面这个Fragment就充当了原来Activity的角色：实例化Fragment和Presenter、互相绑定、显示Fragment。

在MVP框架设计中，还需要避免过度设计的问题。有些操作，其实是完全没有必要通过Presenter代理的，比如点击菜单按钮弹出侧滑抽屉，如果还特意搞一个`Presenter.wantToOpenDrawer`、`View.openDrawer`那就有点多余了。一般来说，不涉及数据操作的行为都可以直接在View中完成。

还有一个小提示是生命周期的管理，这个在网上也有一些争论，不过我还只是初学者，就没有考虑太深，我把Fragment的生命周期也理解成了用户操作，比如`onResume`可以理解成用户把app调回了前台，因此我们可以在Presenter中增加这样的接口，从而完成一些与UI的生命周期有关的逻辑（比如初始化UI等等）。

最后，我还想说说MVP框架的演进。这个说法比较中二，其实是因为在实际的开发过程中，契约并非一成不变。毕竟人非圣贤，一开始设计的东西后面很可能要改（你说我区区一个程序员，怎么就开始改需求了呢？）就拿Tumoji的开发来说，我们就先后发生了很多改动，多亏了MVP框架，使得这些改动对其他开发者的影响减到了最小。

首先，是接口的增删。我们的团队是使用Git进行项目版本控制的，每个人从dev分支开新分支完成某个feature然后合并回dev分支，因此为了不影响其他人，dev分支的代码是必须可以正确运行的。但是我们知道，增加一个接口之后，接口的实现类如果不去实现，是编译都过不了的，而实现类是其他开发者的代码，擅自更改很容易出事，怎么办呢？其实这就是使用接口的目的了。我在Android Studio的设置中，设置了自动生成的Implement Method内容为

````java
// TODO
throw new UnsupportedOperationException("Method not implemented");
````

这样，我对对方的代码修改就仅限于增加一些会抛异常、带TODO注释的方法，把代码冲突减到最小。

当然，删除接口就更加容易了，直接在interface中把对应的接口加上`@deprecated`的JavaDoc或者注解，然后告知对应的开发者不必实现即可，完全不需要更改其他开发者的代码。

使用接口带来的好处就是可以乱改需求- -一开始的时候，由于我不熟悉RxJava，所以所有Repository暴露的接口都是用Listener回调的：

````java
public interface IUserRepository {
  void getUser(String id, GetUserListener callback);
}
````

但是后来，我从数据层的队友那里（我负责Presenter层和框架搭建）学到了RxJava的用法，一下子觉得Listener太不优雅了，于是，我开始把所有的Listener的接口都deprecated了，然后增加了新的接口……

````java
public interface IUserRepository {
  /**
   * @deprecated Use {@link #getUser(String)} instead.
   */
  void getUser(String id, GetUserListener callback);
  
  Observable<UserModel> getUser(String id);
}
````

所以，我没被队友砍死，我挺感动的……

### MVP的其他缺点和解决方案

MVP的优点，从前面就看的很清楚了：非常适合多人协作开发，而且使得项目的逻辑非常清晰，不容易出bug（很容易解决bug）。但是，MVP有一个缺点：测试不方便。

我们平时写代码的时候，写一点跑一下是很轻松的，但是MVP不行，因为三个层被分开来了，对于每个界面而言，只要有一层没有实现，就没法看到效果，甚至没法运行起来，比如你的Presenter和Model都写完了，可是View还没实现，那你运行起来连调用Presenter都做不到。这带来的问题就是自己写了之后没法测试。当然，单元测试可以解决这个问题，但是很遗憾我们都没有这方面的知识，所以我得出了另一个方案：mock实现。

拿Presenter来说，由于数据层的大佬比较忙，进度比我慢一些，导致我没有办法调用数据层，就没有办法测试了。此时，我可以实现一个Mock的类：

````java
public class MockUserRepository implements IUserRepository {
  @Override
  public Observable<UserModel> getUser(String id) {
    return Observable.just(new UserModel().withId(id));
  }
}
````

然后，在可用的UserRepository可用之前，我先使用这个我自己编写的MockUserRepository进行测试即可。同理，View也可以写MockPresenter进行测试，而Repository由于不面向任何一个View，所以只能使用单元测试了（嗯，所以我们项目没有测试Repository……）。

一开始我就是这么写的，但是后来发现，还有更好的写法。试想，Repository已经实现了一部分了，如何部分接入呢？可以这样：

````java
public class UserRepository implements IUserRepository {
  private IUserRepository mDelegate;
  
  // This is a Singleton
  private UserRepository(Context context) {
    mDelegate = new MockUserRepository(context);
    // Other initialization....
  }
  
  @Override
  public Observable<UserModel> getUser(String id) {
    // Unimplemented method
    return mDelegate.getUer(id);
  }
  
  @Override
  public Observable<List<UserModel>> getUsers() {
    // Implemented method
    return mRemote.getUsers();
  }
}
````

如上所示，我们把MockUserRepository作为一个代理，被真正的UserRepository实现类持有。这样，数据层开发者每完成一个方法，就可以去掉代理。而对Presenter而言，他还是使用UserRepository实例。

## 第三方库与框架的使用

终于说完了MVP。Tumoji的开发中使用了很多框架。由于Tumoji是我们的个人作品，所以可以自由使用很多也许还不够成熟的第三方框架，他们为我们的开发节约了大量的时间和精力。

### Retrofit

Retrofit这个神器我已经用得比较熟练了。当初写UWP应用的时候，我需要给每个API写一个函数，而他们都长得几乎一样：

````
public SomeType someApiMethod(params...) {
  // Set URI
  // Set HTTP method (GET, POST, ...)
  // Set Query (access token, ...)
  // Set Header (Content-Type, ...)
  // Create request body
  // Send request
  // Check whether the response is successful
  // Convert response body to String or other data type
}
````

可以想象十几个接口的情况下，这项工作有多么蛋疼。但是，使用Retrofit之后，我们只需要非常简单的定义接口即可：

````java
@GET("users/{id}")
Call<UserModel> getUserById(@Path("id") String id, @Query("access_token") String token);
````

大大简化了代码！

### RxAndroid

RxAndroid是Facebook的ReactX框架在Android平台的实现，前面提到的Listener模式麻烦之处在于每个请求都需要定义Listener，而且有的时候还会出现回调地狱（Callback Hell），非常容易出bug，甚至想想，如果需要根据一个数组，每个元素发一个异步请求，那代码就更加难看了。

后来接触到了RxJava。在RxJava中，有一个被观察者和订阅者。被观察者一旦有人订阅，就会开始某些任务，这些任务可能会获取到一个或多个结果，这些结果作为数据由被观察者发出，再由订阅者接收。举个例子，手机充电，手机就是订阅者，插座就是被观察者，插上电源的过程就是订阅，此时插座源源不断地发出数据（电），手机的显示屏上就会得到这些数据，显示在电池电量数据上。

在Tumoji中，大量使用了RxAndroid简化代码，举个例子：

````java
public class MemeRepository implements IMemeRepository {
  @Override
  public Observable<MemeModel> getMeme(File parentDir, String memeId) {
    return mRemote.getMemeById(memeId)
            .map(memeModel -> mLocal.fulfillDownloaded(parentDir, memeModel));
  }
}
````

这个方法的目的是从服务器获取指定表情的元数据，然后根据ID查找本地数据库是否有该表情，由于已下载的表情的路径和ID会被保存到数据库中，因此可以通过`fulfillDownloaded`，在本地已有的情况下，把表情的Uri从下载地址改为文件路径Uri，从而节约一些流量。

在`mRemote.getMemeById`中，实现非常简单：

````java
public Observable<MemeModel> getMemeById(String memeId) {
  return mMemeApi.getMemeById(memeId).compose(ApplySchedulers.network());
}
````

如上，通过Retrofit配合RxJava，发送请求并把结果作为数据发送，返回发送该数据的Observable。后面的`ApplySchedulres.network`即类似`.subcribeOn(Schedulres.io()).observeOn(AndroidSchedulers.mainThread())`。

而`mLocal.fulfillDownloaded`的实现如下：

````java
public MemeModel fulfillDownloaded(File parentDir, MemeModel memeModel) {
  String filename = mDb.getMemeFileNameById(memeModel.getMemeId());
  if (filename == null) {
    memeModel.setDownloaded(false);
    memeModel.setMemeUri(Uri.parse(memeModel.getImageUrl()));
  } else {
    memeModel.setDownloaded(true);
    memeModel.setMemeUri(Uri.fromFile(new File(parentDir, filename)));
  }
  return memeModel;
}
````

`mDb`提供了从数据库中获取对应ID的本地路径的方法。

可以看到，虽然各自的实现有些复杂，但是在Repository暴露的接口上非常简洁，即使应对一系列请求也是如此，例如当我需要为每个tag发起一个添加标签的请求的时候：

````java
@Override
public Observable<Void> addTagsForMeme(String token, String memeId, List<TagModel> tagModels) {
  ArrayList<Observable<TagModel>> observables = new ArrayList<>();
  for (TagModel tagModel : tagModels) {
    observables.add(tagApi.relTagToMeme(memeId, tagModel.getTagName(), token).compose(ApplySchedulers.network()));
  }
  return Observable.merge(observables).toList().map(new Func1<List<TagModel>, Void>() {
    @Override
    public Void call(List<TagModel> tagModels) {
      return null;
    }
  );
}
````

代码的逻辑很明了。

## 关于团队开发

说起团队开发，我也算是半个老司机了吧，之前就和他们开发了[WoChat](https://github.com/zhengqi-big-god-take-me-fly/WoChat)，一个UWP平台的IM应用和一个模仿泡泡堂的Cocos2d-x游戏[天天爱打泡](https://github.com/zhengqi-big-god-take-me-fly/FancyBubbling)（手动滑稽），现在我们又开发了Tumoji。WoChat的开发我们使用了MVVM框架，不过由于对UWP平台不熟悉，所以基本上是依葫芦画瓢；天天爱打泡的开发则参考了MVC，但是由于Cocos2d平台本身是多语言的，所以不容易找到针对C++的MVC框架；而Android上的MVP算是很热门也比较成熟了，所以开发起来压力比前面都要小一些。

在团队开发的工程中，最大的感慨莫过于意识到扎实的基本功对架构设计的重要性。虽然学Android的时间很长，但是一直都只是皮毛，其实我个人认为我现在接触MVP还是有点早，有点霸王硬上弓了，在设计过程中就会发现，很多时候由于Android本身的机制所限，并不能单纯地划分MVP逻辑，而对Android开发的基本功不够扎实就导致框架设计中遇到了很多问题。

借助Git的分支管理策略和MVP的框架，我们得以非常轻松地进行团队合作。当然，团队合作也并非易事，最大的问题在于进度，四个人都有自己的事情，而且临近期末，所以大家的开发进度不一，如前所述，进度的不一致给各自的测试也带来了一些问题，但是总体来说，我们的团队在合作上还是很顺利的。身为组长我也就不邀功了，就做了三件事：

*  把MVP的框架搭建了起来
*  完成了Presenter的任务
*  在另外几位大佬忙于复习的时候，帮忙填了一些View和Model的坑

当然，Android客户端的顺利开发离不开[@Tidyzq](https://github.com/Tidyzq)提供的健壮又接口丰富的服务器，虽然由于使用了Loopback而拒绝提供文档，对API使用者不太友好……不过程序员嘛……都不爱写文档的，哈哈哈。

当然，也因为时间的确有些赶，项目的进度并不算乐观，目前仅仅完成了最基础的功能，而且有大量bug没有处理，还有不少坑需要填啊！

## 关于产品

由于整个项目的起源来自我，所以在整个设计中，会发现很多关系到代码之外的事情，比如缓存数据的事情。一开始的时候，我计划每次都将列表最前的若干项缓存到数据库，以便下次打开应用的时候能立刻加载缓存数据。其实到后面我发现，这样的意义很小，因为这个数据不是递增的而是被频繁替换的，Tumoji是严重依赖网络的，因此离线缓存的重要性其实并不大，因为离线的时候缓存的东西都没有什么用。因此，导致了开发工程中一次很大的变动，对Model层的开发还是有不小的影响。同时，为了节约用户的流量，我们在Retrofit上配置了缓存。

开发Tumoji的过程让我体会到一些产品设计的困难，诸如缓存的权衡、对网络的请求应该如何限制、以及一些用户上的交互等等。

## 结语

坦率的讲，Tumoji应该是我（参与）开发的第一款能用的app（当然之前做过一次外包，但是主要是一个师兄带我），因此我也计划将它维护下去，相信能从中学到很多东西。另外，开发进度的延误和单元测试的缺漏还是这个项目中的一些遗憾。

那么，废话就说到这里，毕竟我接下来还有两科考试要挂……希望Tumoji早日在应用商店与大家见面，到时候我应该会再写一篇博客写写准备发布应用和在商店上架应用的事情。