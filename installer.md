### 前言

在沙盒的世界里会运行着各种各样的Android应用程序，每个用户空间互不干扰，同时在同一个用户空间的程序需要进行交互，那么制作 一个应用程序安装器就显得相当重要了。我们需要安装器来对APK文件进行解析且对应用程序的组件进行缓存，从而使应用程序之间能够快速且便捷地进行交互。

### 知识准备

在` Android Framework`中有一个非常重要的类` android.content.pm.PackageParser.java`，此类负责着` Android`的包解析工作，不过此类在Android S之后就被弃用了，取而代之的则是` android.content.pm.parsing.ParsingPackageUtils.java`，不过实现大致相同，所以这里不作区分且会直接使用` PackageParser.java`。

查看` PackageParser.java`可以看到：

```java
/**
  * Parse the package at the given location. Automatically detects if the
  * package is a monolithic style (single APK file) or cluster style
  * (directory of APKs).
  * <p>
  * This performs checking on cluster style packages, such as
  * requiring identical package name and version codes, a single base APK,
  * and unique split names.
  * <p>
  * Note that this <em>does not</em> perform signature verification; that
  * must be done separately in {@link #collectCertificates(Package, boolean)}.
  *
  * If {@code useCaches} is true, the package parser might return a cached
  * result from a previous parse of the same {@code packageFile} with the same
  * {@code flags}. Note that this method does not check whether {@code packageFile}
  * has changed since the last parse, it's up to callers to do so.
  *
  * @see #parsePackageLite(File, int)
  */
 @UnsupportedAppUsage
 public Package parsePackage(File packageFile, int flags, boolean useCaches)
         throws PackageParserException {
  if (packageFile.isDirectory()) {    
    return parseClusterPackage(packageFile, flags);
  } else {
    return parseMonolithicPackage(packageFile, flags);
  }
 }

/**
 * Equivalent to {@link #parsePackage(File, int, boolean)} with {@code useCaches == false}.
 */
@UnsupportedAppUsage
public Package parsePackage(File packageFile, int flags) throw PackageParseException {
  return parsePackage(packageFile, flags, false);
}
```

通过注释我们可以知道，这个方法可以解析安装包，同时也具有解析` split apk`的能力。那么做法已经很明显了，肯定是直接反射获取` PackageParser`对象，然后调用其` parsePackage`方法，之后我们就可以获得一个` Package`对象，我们来看一下` Package`到底是什么：

``` java
public final static class Package implements Parcelable {
  public String packageName;
  public String manifestPackageName;
  public String[] splitNames;
  // 略。。。
  public ApplicationInfo applicationInfo = new ApplicationInfo();
  public final ArrayList<Permission> permissions = new ArrayList<Permission>(0);
  public final ArrayList<PermissionGroup> permissionGroups = new ArrayList<PermissionGroup>(0);
  public final ArrayList<Activity> activities = new ArrayList<Activity>(0);
  public final ArrayList<Activity> receivers = new ArrayList<Activity>(0);
  public final ArrayList<Provider> providers = new ArrayList<Provider>(0);
  public final ArrayList<Service> services = new ArrayList<Service>(0);
  public final ArrayList<Instrumentation> instrumentation = new ArrayList<Instrumentation>(0);
  // 略。。。
}
```

可见这个` Package`保存着apk被解析后的所有信息，那我们需要做的事情也相当清楚了，那就是把这些信息给缓存下来且让其常驻内存，毕竟应用程序之间做交互的时候，我们并不希望程序在这个时候进行数据加载或解析。既然数据需要缓存，那么` Package`对象就需要是可序列化的，这个问题Android的工程师早就考虑到了，所以其实现了` Parcelable`接口。

我们都知道` Android`应用程序的世界里，其组件是相当重要的部分，其中`Activity、Service、Receiver和Provider`四大组件最为重要，那么我们对` Package`再进行深一层的代码查看，发现` Activity`等都在其中，打开` Activity`类看一下：

``` java
public final static class Activity extends Component<ActivityIntentInfo> implements Parcelable {
  public final ActivityInfo info;
  // 略。。。
}
```

里面有一个` ActivityInfo info`的成员变量，我们打开` ActivityInfo`的源码看看：

``` java
public class ActivityInfo extends ComponentInfo implements Parcelable {
  // 略。。。
  public int theme;
  // 略。。。
}
```

基本上就是保存着` Activity`组件的相关信息，由于其为` ComponentInfo`的子类，所以我当然也是要看一个` ComponentInfo`的源码咯：

``` java
public class ComponentInfo extends PackageItemInfo {
  public ApplicationInfo applicationInfo;
  public String processName;
  // 略。。。
}
```

再看一下` PackageItemInfo`：

``` java
public class PackageItemInfo {
  // 略。。。
  public String name;
  public String packageName;
  // 略。。。
}
```

终于到底了，嗯，也是一些基本信息，那么基本上就是确定了每个组件都会有一个` xxxInfo`的成员，而这个成员都是像上面所描述的实现方式，我们大概看看其他的组件确认一下：

``` java
// provider组件
public final static class Provider extends Component<ProviderIntentInfo> implements Parcelable {  
  public final ProviderInfo info;
  // 略。。。
}
public static class ProviderInfo extends ComponentInfo {}

// service组件
public final static class Service extends Component<ServiceIntentInfo> implements Parcelable {
  public final ServiceInfo info;
  // 略。。。
}
public static class ServiceInfo extends ComponentInfo {}

// 略。。。
```

嗯，没问题，就是这么实现的。来到这里，我们还有一个事情没搞清楚，那就是每个组件都会继承一个叫` Component`的类，我们来看看：

``` java
public static abstract class Component<II extends IntentInfo> {
  public final ArrayList<II> intents;
  public final String className;
  public Bundle metaData;
  public Package owner;
  public int order;
   // 略。。。
}
```

馁了，` Component`是一个抽像类，里面记录了类名、哪个` package`拥有此组件、排序值和意图列表等信息，这么看来，保存这些信息是为了组件的查找等操作了，我们来看看这个` intents`，顾名思义，这里面铁定存放着各种意图信息，` II extends IntentInfo`，存储的这些类都是来自于` IntentInfo`，那么必然就是` Intent`的各种属性呗，我们来看一下：

``` java
public static abstract class IntentInfo extends IntentFilter {
  public boolean hasDefault;
  public int labelRes;
  public CharSequence nonLocalizedLabel;
  // 略。。。
}
```

有点意外，竟然都是一些资源信息，再看看`IntentFilter`：

``` java
public class IntentFilter implements Parcelabel {
  // 略。。。
  private final ArrayList<String> mActions;
  private ArrayList<String> mCategories = null;
  private ArrayList<String> mDataSchemes = null;
  // 略。。。
}
```

噢，看到熟悉的变量了，很显然这里面保存着的信息是跟我们在`AndroidManifest.xml`是相互对应的。下面我们来捋一下他们上述所描述的类的关系：

``` mermaid
classDiagram

IntentFilter <|-- IntentInfo
IntentInfo <|-- ActivityIntentInfo
ActivityIntentInfo --o Component
Component <|-- Activity
Component : +ArrayList~ActivityIntentInfo~ intents
PackageItemInfo <|-- ComponentInfo
ComponentInfo <|-- ActivityInfo
Activity : + ActivityInfo info
ActivityInfo --o Activity

```

现在我们对包解析器已经有一定了解了，那么我们解析出来的这些东西干嘛用的呢？前面也有提过为了各应用程序之间做交互用的，想像一下，如果我们需要启动一个这样的` Activity`：

``` XM
<activity android:name=".MainActivity" android:exported="true">
		<intent-filter>
			<action android:name="android.intent.action.MAIN" />
			<category android:name="android.intent.category.LAUNCHER" />
		</intent-filter>
</activity>
```

的时候，我们一般会这么做：

``` java
Intent intent = new Intent();
intent.setAction("android.intent.action.MAIN");
intent.addCategory("android.intent.category.LAUNCHER");
startActivity(intent);
```

然后相应的` Activity`可能就打开了，那么我们的系统是怎么找到这个组件的呢？省略一大段的调用过程（具体会在` ActivityProxy`的实现过程再做详细分析），我们直接来到：

``` java
// 略。。。
public class ActivityStart {
  // 略。。。
  int execute() {
  	// 略。。。
    if (mRequest.activityInfo == null) {
      // !!!
    	mRequest.resolveActivity(mSupervisor);
    }
    // 略。。。
    res = executeRequest(mRequest);	// 执行打开的逻辑
    // 略。。。
  }
  // 略。。。
}
```

在上面的 ` !!!`处会去检索` activityInfo`，跟进去看看：

``` java
static class Request {
  // 略。。。
  void resolveActivity(ActivityTaskSupervisor supervisor) {
    // 略。。。
    resolveInfo = supervisor.resolveIntent(intent, resolvedType, userId, 
                                          0 /* matchFlags */,
                                          computeResolveFilterUid(callingUid, realCallingUid, filterCallingUid));
    // 略。。。
  }
  // 略。。。
}
```

接着往下看：

``` java
public class ActivityTaskSupervisor implements RecentTasks.Callbacks {
  ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId, int flags, int filterCallingUid) {
    // 略。。。
    return mService.getPackageManagerInternalLocked().resolveIntent(
                        intent, resolvedType, modifiedFlags, privateResolveFlags, userId, true,
                        filterCallingUid);
    // 略。。。
  }
}
```

再继续看：

``` java
public class PackageManagerService extends IPackageManager.Stub implements PackageSender, TestUtilityService {
  // 略。。。
  private class PackageManagerInternalImpl extends PackageManagerInternal {
    // 略。。。
    public ResolveInfo resolveIntent(Intent intent, String resolvedType, 
            int flags, int privateResolveFlags, int userId, boolean resolveForStart,
            int filterCallingUid) {
      return resolveIntentInternal(intent, resolvedType, flags, privateResolveFlags, userId, resolveForStart,
                    filterCallingUid);
    }
    // 略。。。
    private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType, int flags,
            @PrivateResolveFlags int privateResolveFlags, int userId, boolean resolveForStart,
            int filterCallingUid) {
      // 略。。。
      final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
                    flags, privateResolveFlags, filterCallingUid, userId, resolveForStart,
                    true /*allowDynamicSplits*/);
      // 略。。。
      // 这一步在上面的query选择一个最适合的
      final ResolveInfo bestChoice =
                    chooseBestActivity(
                            intent, resolvedType, flags, privateResolveFlags, query, userId,
                            queryMayBeFiltered);
      // 略。。。
    }
  }
  // 略。。。
}
```

我们再看看` queryIntentActivitiesInternal`方法：

``` java
public final @NonNull List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
                String resolvedType, int flags, @PrivateResolveFlags int privateResolveFlags,
                int filterCallingUid, int userId, boolean resolveForStart,
                boolean allowDynamicSplits) {
  // 略。。。
  if (comp != null) {
    final List<ResolveInfo> list = new ArrayList<>(1);
    final ActivityInfo ai = getActivityInfo(comp, flags, userId);
    if (ai != null) {
      // 一堆判断之后
      final ResolveInfo ri = new ResolveInfo();
      ri.activityInfo = ai;
      list.add(ri);
    }
    // 调用了下面的方法就返回了
    List<ResolveInfo> result = applyPostResolutionFilter(
                        list, instantAppPkgName, allowDynamicSplits, filterCallingUid,
                        resolveForStart,
                        userId, intent);    
    return result;
  }
  // 不过我们的例子comp肯定是个null，毕竟Intent没传可以让其不为null的参数，我们要看下面的代码
  // 略。。。
  // 下面的代码在Android R就被封装成这样了，但是在Android Q以下会有所区别，原理差不多  
  QueryIntentActivitiesResult lockedResult =
                    queryIntentActivitiesInternalBody(
                        intent, resolvedType, flags, filterCallingUid, userId, resolveForStart,
                        allowDynamicSplits, pkgName, instantAppPkgName);
  /*
  	Android Q大概是长这样的
  	result = filterIfNotSystemUser(mComponentResolver.queryActivities(
                        intent, resolvedType, flags, userId), userId);

  */
  // 略。。。
  //然后一堆逻辑之后就返回了List<ResolveInfo> result
}
```

代码量挺多的，我们再看看` queryIntentActivitiesInternalBody`这个方法：

``` java
public @NonNull QueryIntentActivitiesResult queryIntentActivitiesInternalBody(...) {
  // 略。。。
  // 来到了上面讲的Android Q那馁代码了，一模一样，也迎来了我们的重点了
  result = filterIfNotSystemUser(mComponentResolver.queryActivities(
                        intent, resolvedType, flags, userId), userId);
  // 略。。。
}
```

经过一大馁的逻辑之后，我们需要关注的类终于现身了那就是` mComponentResolver`这玩意了，上面的` filterIfNotSystemUser`这个方法干了啥，我们就不看了，字面意思也比较清楚，咱们直接看看` mComponentResolver`到底是啥：

``` java
/** Resolves all Android component types [activities, services, providers and receivers]. */
public class ComponentResolver extends WatchableImpl implements Snappable {
  // 略。。。
  List<ResolveInfo> queryActivities(Intent intent, String resolvedType, int flags, int userId) {
    synchronized (mLock) {
      return mActivities.queryIntent(intent, resolvedType, flags, userId);
    }
  }
  // 略。。。
}
```

看到官方的注释，可以得知这里可以检索` Android`的四大组件，上面的` mActivities`是个` ActivityIntentResolver`，那我们再看看它是啥：

``` java
// 继承了MimeGroupsAwareIntentResolver并且会有一定的约束, 追踪下去也还是IntentResolve的子类
private static class ActivityIntentResolver
            extends MimeGroupsAwareIntentResolver<Pair<ParsedActivity, ParsedIntentInfo>, ResolveInfo> {
  // 略。。。
  List<ResolveInfo> queryIntent(Intent intent, String resolvedType, int flags,
                int userId) {
    if (!sUserManager.exists(userId)) {
      return null;
    }
    mFlags = flags;
    return super.queryIntent(intent, resolvedType,
                             (flags & PackageManager.MATCH_DEFAULT_ONLY) != 0,
                             userId);
  }
  // 略。。。
}
```

` Android Q`以下的代码如下：

``` java
private static final class ActivityIntentResolver
            extends IntentResolver<PackageParser.ActivityIntentInfo, ResolveInfo> {
  // 略。。。
  // queryIntent的实现跟上面的一样
  // 略。。。
}
```

既然最终都是` IntentResolve`的子类，那么我们直接看一下` IntentResolve`的代码：

``` java
public abstract class IntentResolver<F, R extends Object> {
  // 略。。。
  // 这里的R是个Object类型，但是再往子类去看，在我们这里它就是ResolveInfo
  public List<R> queryIntent(Intent intent, String resolveType, boolean defaultOnly, int userId) {
  	// 略。。。
    // 逻辑大概就是处理一下intent，经过一轮的清洗，我们的例子大概会跑到下面几句
    if (resolvedType == null && scheme == null && intent.getAction() != null) {
      firstTypeCut = mActionToFilter.get(intent.getAction());	// 因为我们有传action，所以这里肯定会有值
      if (debug) Slog.v(TAG, "Action list: " + Arrays.toString(firstTypeCut));
    }
    FastImmutableArraySet<String> categories = getFastIntentCategories(intent);	// 也传了这个，所以这里同样会有值
    if (firstTypeCut != null) {
      // 那么就会跑进来这里
      buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                       scheme, firstTypeCut, finalList, userId);
    }
    // 略。。。
  }
  // 略。。。
}
```

这里需要解析一下这个`mActionToFilter`里面存的是啥了：

``` java
/**
  * All of the actions that have been registered, but only those that did
  * not specify data.
	*/
private final ArrayMap<String, F[]> mActionToFilter = new ArrayMap<String, F[]>();
```

看注释就是保存着所有的` actions`，同样往子类回溯，它其实就是` ActivityIntentInfo`。好了，到这里我们已经回到了开头讲的那些类了，我们再继续往下看：

``` java
private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
            boolean debug, boolean defaultOnly, String resolvedType, String scheme,
            F[] src, List<R> dest, int userId) {
  // 略。。。
  final R oneResult = newResult(filter, match, userId);
  // 略。。。
}

@SuppressWarnings("unchecked")
protected R newResult(F filter, int match, int userId) {
  return (R)filter;
}
```

里面的逻辑认真看一下还是比较好懂的，大概就是通过匹配到的` actionsFilter`(这里是` ActivityIntentInfo`列表)进行一些条件帅选之后，将符合要求的通过调用` newResult`对` ResolveInfo`进行实例化，很显然上面的`newResult`必然会被其子类重写，有兴趣可以看看`ActivityIntentResolver.newResult`的实现，大概就是对`ResolveInfo`的实例化并且对其属性进行填充。

### 实现过程

