### 前言
在制作Android沙盒的时候，调用系统的内部方法可以有效且快速地达到相应的目的，然而系统方法很多时候都是隐藏的，所以我们需要使用反射机制来对这些隐藏的方法进行调用。一次完整的系统隐藏方法的调用，首先是加载相应的类，再通过反射找到相应的方法，然后就是通过相应的类实例来进行方法调用，在这个过程还需要处理各种异常，显然对于每个系统调用我们都要做大量的重复工作，那么封装一个易用的系统方法调用工具就变得十分重要了。

### 设计过程

我们先从需要被反射的类开始入手，以`android.app.Application`为类，首先，为了能清晰地看出反射类与被反射类的关系，故反射类的包名和类名必须得存在着被反射类的特征：

```java
// 后面的android.app与Application的android.app对应
package com.xxh.rosehong.refection.android.app;

// 以Application加上Ref，代表着此类为Application的反射类
public class ApplicationRef {}
```

经过上面的代码设计，我们通过查看反射类就能轻松地知道被反射的类到底是什么。当然，我们反射的目的是要访问被反射的类的成员或者调用其方法，我们先来看看`Application`类到底长什么样的：

``` java
package android.app;
// 略...
public class Application extends ContextWrapper implements ComponentCallbacks2 {
  // 略...
  public LoadedApk mLoadedApk;
  // 略...
  final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
  }
  // 略...
}
```

很显然，我们要访问`Application`的`mLoadedApk`成员和`attach`方法，自然就想到这么这么写：

```java
public class ApplicationRef {
  // 定义静态变量，对应mLoadedApk成员
  public static Field mLoadedApk;
  // 定义静态变量，对应attach方法
  public static Method attach;
  static {
    try {
      // 直接通过反射拿到mLoadedApk成员
	    mLoadedApk = Application.class.getDeclaredField("mLoadedApk");
      mLoadedApk.setAccessible(true);
  	  // 直接通过反射拿到attach方法
    	attach = Application.class.getDeclareMethod("attach", Context.class);
      attach.setAccessible(true);
    } catch(Exception e) {} 
  }
}
```

这写起来太烦锁了，而且每次编写的时候，都要对异常进行处理，我很懒，我希望一行代码就能把这些事情都做完，所以我们需要对上面的代码进行优化。首先我们得封装几个工具类，这些类需要做的就是完成成员或方法的反射、设置权限和处理异常等，由于静态与非静态、`Field`与`Method`、构造方法与成员方法和方法的重载等有着不一样的特性，故我们需要分别进行封装处理，由于原理大致相同，所以这里会以`Field`和`Method`为例子进行讲解。

所有反射相关的工具类都统一放在` com.xxh.rosehong.utils.ref`包里面：

``` java
package com.xxh.rosehong.utils.ref;

// 处理Field
public class RhField {
  private Field mRefField;	// 被反射的成员变量，如android.app.Applicaiton中的mLoadedApk的Field对象
  
  /*
  	srcClass: 被反射类的Class对象，如android.app.Application
  	fieldName: 需要反射的成员变量名，如ApplicationRef中的mLoadedApk
  */
  public RhField(Class srcClass, String fieldName) {
    try {
      // 我们只需要能够反射到其声明的成员变量即可
      mRefField = srcClass.getDeclaredField(fieldName);
      mRefField.setAccessible(true);
    } catch (NoSuchFieldException e) {
      e.printStackTrace();
    }
  }
}
```

同理`Method`可得：

```java
package com.xxh.rosehong.utils.ref;

// 处理Method
public class RhMethod {
  private Method mRefMethod;	// 被反射的成员方法，如android.app.Application中的attach的Method对象
  
  /*
  	srcClass: 被反射类的Class对象，如android.app.Application
  	methodName: 需要反射的方法名，如ApplicationRef中的attach
  */
  public RhMethod(Class srcClass, String methodName) {
    // 同样，我们只需要反射其声明的成员方法即可，不过方法是分为：有参和无参的，这里处理无参，有参的待会再作处理
    mRefMethod = srcClass.getDeclaredMethod(methodName);
    mRefMethod.setAccessible(true);
  } catch (MoSuchMethodException e) {
    e.printStackTrace();
  }
}
```

有了这两个工具类之后，我们的`ApplicationRef`就变得精简了许多：

```java
import android.app.Application;

public class ApplicationRef {
  public static RhField mLoadedApk = new RhField(Application.class, "mLoadedApk");
  public static RhMethod attach = new RhMethod(Application.class, "attach");
}
```

观察上面的`ApplicationRef`代码，我们发现每次定义一个反射关系的成员变量，都需要做一次`new`操作，并且首个都是`Application.class`和第二个参数就是对应的成员变量的名称。显示这里可以再抽象出来，况且`ApplicationRef`站在计算机的角度也无法直接得知其被反射的对象到底是谁。

想要解决直接得知被反射对象是谁的话，很简单，我们使用一个变量来记录就行了：

```java
import android.app.Application;

public class ApplicationRef {
  // 直接使用REF来记录被反射的对象
  public static Class REF = Application.class;
  
  // 略...
}
```

不过我们好像可以将`REF = Application.class`作出更进一步的的修改，大胆设想一下，如果我们能把`REF`的初始化变成`ApplicationRef`的初始化，这样的话，我们就可以把上面的设计全部封装到初始化里面做，就可以真正地做到一行代码搞定所有了，说干就干，既然是反射类的初始化，那么就把类反射相关的操作先封装好：

```java
package com.xxh.rosehong.utils.ref;

public class RhClass {
  
  // 处理无法直接获取得到类对象的情况
  public static Class<?> init(Class refClass, String srcClassName) {
    try {
      // 获取被反射的类对象
      Class srcClass = Class.forName(srcClassName);
      return init(refClass, srcClass);
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    }
  }
  
  /*
  	refClass: ApplicationRef
  	srcClass: Application
  */
  public static Class<?> init(Class refClass, Class srcClass) {
    return srcClass;
  }
}
```

这一步完之后，我们的`ApplicationRef`就可以写成这样：

```java
public class ApplicationRef {
  // 直接通过调用RhClass.init进行初始化，由于init的返回值就是Application.class，所以达到了REF记录的值也是对的
  public Class<?> REF = RhClass.init(ApplicationRef.class, Application.class);
  
  public static RhField mLoadedApk = new RhField(Application.class, "mLoadedApk");
  public static RhMethod attach = new RhMethod(Application.class, "attach");
}
```

好了，接下来要处理的事情也很明显了，那就是把`mLoadedApk`和`attach`的初始化放到`RhClass.init`里面实现：

```java
public class RhClass {
  public static Class<?> init(Class refClass, Class srcClass) {
    // 由于refClass就是ApplicationRef的类对象，那么我们可以通过反射获取其所有的成员变量
    for (Field field : refClass.getFields()) {
      // 遍历出来，准备对其进行初始化操作，由于我们一定会将其定义为static，所以这里进行一下过滤
      if (Modifier.isStatic(field.getModifiers())) {
        if (field.getType() == RhField.class) {
          // 如果是RhField类型的
          field.set(null, new RhField(srcClass, field.getName()));
        } else if (field.getType() == RhMethod.class) {
          // 如果是RhMethod类型的
          field.set(null, new RhMethod(srcClass, field.getName()));
        }
      }
    }
    return srcClass;
  }
}
```

如此一来，我们的`ApplicationRef`就变成非常简洁了：

```java
public class ApplicationRef {
  public static Class<?> REF = RhClass.init(ApplicaitonRef.class, Application.class);
  public static RhField mLoadedApk;
  public static RhMethod attach;
}
```

来到这里，我们基本上锁定了编写一个反射类是什么样子的了，不过还有一个需要优化的地方，就是我们的`RhClass.init`里对`RhField`和`RhMethod`的判断，我们可以使用一个`HashMap`来存储`RhField`和`RhMethod`的构造方法，并在`init`的时候直接使用反射调用来进行实例化：

```java
public class RhClass {
  
  // 定义一个map容器
  private static Map<Class, Constructor> mInterfaceMap = new HashMap(2);
  static {
    try {
      // 对所有的成员反射类型进行构造方法注册
      mInterfaceMap.put(RhField.class, RhField.class.getConstructor(Class.class, Field.class));
      mInterfaceMap.put(RhMethod.class, RhMethod.class.getConstructor(Class.class, Field.class));
    } catch (NoSuchMethodException e) {
      e.printStackTrace();
    }
  }
  
  public static Class<?> init(Class refClass, Class srcClass) throw Exception {
    for (Field field : refClass.getFields()) {
      // 成员为静态，并且已注册了构造方法
      if (Modifier.isStatic(field.getModifiers()) && mInterfaceMap.containsKey(field.getType())) {
        Constructor constructor = mInterfaceMap.get(field.getType());
        // 通过构造方法对对象进行实例化
        field.set(null, constructor.newInstance(srcClass, field.getName()));
      }
    }
    return srcClass;
  }
}
```

好了，现在基本上其他任意需要反射的类，都可以通过上述的方式来完成了。回到先前的`Method`参数的问题，上面的封装仅支持无参的方法的处理，那么有参的方法该怎么处理呢？我们要怎样才能把参数传递给`RhMethod`去进行处理呢？思来想去，要解决这个问题的话，比较理想的办法就是使用注解，通过注解的方式，我们就能把参数类型列表传递给`RhMethod`了。我们先来定义一个注释：

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RhMethodParams {
  Class<?>[] value();	// 返回Class数组
}
```

定义好注解之后，那么在`Method`进么反射的时候，就可以写成如下这样：
``` java
public class ApplicationRef {
	public static Class<?> REF = RhClass.init(ApplicationRef.class, Application.class);
  
  @RhMethodParams({Context.class})
  public static RhMethod attach;
}
```
当然了，在`RhMethod`里面，我们还需要对注解进行解析：
``` java
public class RhMethod {
	private Method mRefMethod;
  
  // 第二个参数需要更改一下，因为我们需要通过它去获取参数类型列表，所以得把Field对象传递过来
  public RhMethod(Class srcClass, Field refFeild) {
    try {
      Class<?>[] parameterTypes = getParams(refField);
      // 获取方法名，为了支持重载，可选择特写分隔符来进行处理，如attach#1，取出attach即可
      String methodName = refField.getName();
      if (parameterTypes != null) {
        mRefMethod = srcClass.getDeclaredMethod(methodName, parameterTypes);
      } else {
        mRefMethod = srcClass.getDeclaredMethod(methodName);
      }
      mRefMethod.setAccessible(true);
    } catch (NoSuchMethodException e) {
      e.printStackTrace();
    }    
  }
  
  // 通过解析注释来获取参数列表
  public static Class<?>[] getParams(Field field) {
    Class<?>[] res = null;
    // 存在注释
    if (field.isAnnotationPresent(RhMethodParams.class)) {
      RhMethodParams annotation = field.getAnnotation(RhMethodParams.class);
      // 将列表读取出来
      res = annotation.value();
    }
    return res;
  }
}
```
到这里，我们的反射工具类设计基本完成了，当然了，还有构造方法、静态方法和静态变量等的实现，本文就不再讲解了，原理上是一样的，具体的实现可以查看 [项目源码](https://github.com/RoseHong/RoseHong/tree/main/core/src/main/java/com/xxh/rosehong/utils/res)。Enjoy it!