### 前言

经过上一篇文章的讲解，我们已经知道怎么利用系统的`PackageParser`类来进行`apk`文件的解析和大概了解到系统是如何进行组件查找的，那么此篇文章我们将会介绍如何来设计和实现沙盒的安装器。

## 设计过程

### 入口设计

首先我们会设计一个接口，此接口可以接收任意的路径（如：`apk`安装包的路径，需要考虑`split apk`会是`apks`、`xapk`或者文件夹），而应用层只需要调用此接口即可完成应用程序的安装，当然了，应用层需要知道安装到底是成功了还是失败了，若是失败了，到底什么原因失败的，故代码如下：

``` java
// 结果是可以通过binder进行传输的，故需要可序列化的
public class RhInstallResMod implements Parcelable {
  private int resCode;	// 返回码
  private String resDesc;	// 返回描述
}

public class RhPackageInstaller {  
  public RhInstallResMod installPackage(String packageFilePath) {
    // TODO: 进行安装
  }
}
```

在进行安装文件解析之前，我们需要对安装文件进行清洗操作，例如到底是不是我们所支持的文件类型，传进来的如果是`split apk`，那么我们需要对其进行解压缩，并且对不相关的文件进行清除处理，这里就不贴代码了，只需要做到安装接口拿到的路径要么是`xxx.apk`这样的单文件，要么就是一个文件夹路径且此文件夹仅仅存在`.apk`这一种类型的文件即可。

### 解析器

对于解析器如何解析`apk`或者`split apk`的，我们不需要知道有兴趣的兄弟可以查看`Framework`的相关代码，这里只需要通过反射机制对系统的解析器进行调用即可。通过查看`PackageParser.java`的源码可知，若想调用其`parsePackage`方法，我们得拿到`PackageParser`的实例才行：

``` java
public class PackageParserRef {
  public static Class<?> REF = RhClass.init(PackageParserRef.class, "android.content.pm.PackageParser");
  
  public static RhConstructor<Object> constructor;	// 反射其构造方法
  @RhMethodParams({String.class, int.class})
  public static RhMethod<Object> parsePackage;	// 反射解析方法
}
```

OK，有了其反射类，那么调用就变得相当简单了：

``` java
public class RhPackageParser {
  public static RhPackage parsePackage(File packageFile) {
    Object packageParser = PackageParserRef.constructor.newInstance();	// 实例化packageParser
    Object basePackage = PackageParserRef.parsePackage.call(packageParser, packageFile, 0);	// 调用parsePackage方法
  }
}
```

从安装器的准备篇我们得知调用`PackageParser.parsePackage`方法之后，我们可以得到一个叫`PackageParser.Package`的类，而这个类里面存储着解析后的应用程序的所有信息，所以我们需要编写一个`PackageParser.Package`类所对应的反射类，为了与`FrameWork`的源码结构保持一致，所以我们直接将其编写在`PackageParserRef`类的内部：

``` java
public class PackageRef {
  public static Class<?> REF = RhClass.init(PackageRef.class, "android.content.pm.PackageParser$Package");
  public static RhField<String> packageName;
  public static RhField<String[]> splitNames;
  // 略。。。
}
```

有了`PackageParser.Package`的实例以及其对应的反射类，那么我们就可以轻松地访问其成员变量了。可是`Package`类的内容太多了，对于沙盒来说，把所有的信息都持久化或者常驻内存是没有太大的必要的，况且，我们在读取或修改其成员的值的时候，每次都是`PackageRef.packageName.get(basePackage)`这样，说真的，我很懒，也觉得这样的调用不简洁，所以，我们要再进行一次类的封装。

我们先来回想一下在准备篇里谈到的组件的构造、组件与组件之间的关系和系统是如何对相应的组件进行查找的，很自然地表明，我们需要把组件们都封装起来，我们首先看看`Package`要怎么封装，其实很简单，就是照葫芦画瓢，对着`PackageParser.Package`抄即可，当然了，我们需要精简一下，并不是所有的成员我们都需要：

``` java
public class RhPackage implements Parcelable {
  public String packageName;
  public String[] splitNames;
  public String codePath;
  // 略。。。
}
```

然后我们的`RhPackageParser`就可以这么写了：

``` java
public class RhPackageParser {
  public static RhPackage parsePackage(File packageFile) {
    // 略。。。
    Object basePackage = PackageParserRef.parsePackage.call(packageParser, packageFile, 0);
    // 实例化一个RhPackage，并且对其进行成员赋值
    RhPackage rhPackage = new RhPackage();
    rhPackage.packageName = PackageRef.packageName.get(basePackage);
    // 略。。。
    return rhPackage;
  }
}
```

接着我们看看组件怎么封装，我们以`Activity`为类，