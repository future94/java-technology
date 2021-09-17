
## 1. 代理概念

代理是一种常用的设计模式，其目的就是为其他对象提供一个代理以控制对某个对象的访问。`代理类可以对委托类调用前做特殊处理，调用委托类的实现，以及委托类调用后做特殊处理`。

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/java/images/20200811203201180.png)

好处：可以在不改变原有类的基础上对其增强或控制他的执行等，如Spring Aop。

## 2. 使用步骤

### 2.1 定义业务接口

```java
public interface IUserService {

    void add(String name);
}
```

### 2.2 定义业务接口实现

```java
public class IUserServiceImpl implements IUserService {

    @Override
    public void add(String name) {
        System.out.println("name:" + name);
    }
}
```

### 2.3 实现InvocationHandler接口

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * @author weilai
 */
public class UserInvocationHandler implements InvocationHandler {

	// 被代理的对象
    private Object target;

    public UserInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before invoke");
        Object result = method.invoke(target, args);
        System.out.println("after invoke");
        return result;
    }
}
```

### 2.4 调用代理方案

```java
import java.lang.reflect.Proxy;

/**
 * @author weilai
 */
public class TestUserProxy {

    public static void main(String[] args) {
        IUserService iUserService = new IUserServiceImpl();
        UserInvocationHandler userInvocationHandler = new UserInvocationHandler(iUserService);
        IUserService userService = (IUserService) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), iUserService.getClass().getInterfaces(), userInvocationHandler);
        userService.add("weilai");
    }
}
```

运行输出：
```txt
before invoke
name:weilai
after invoke
```

## 3. 原理

我们从 `Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException` 入手。

```java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }

    /*
     * Look up or generate the designated proxy class.
     * 查询（在缓存中已经有）或生成指定的代理类的class对象
     */
    Class<?> cl = getProxyClass0(loader, intfs);

    /*
     * Invoke its constructor with the designated invocation handler.
     */
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

getProxyClass0实际上就是从proxyClassCache中获取代理对象，而proxyClassCache是一个WeakCache。

> private static final WeakCache<ClassLoader, Class<?>[], Class<?>> proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }

    // If the proxy class defined by the given loader implementing
    // the given interfaces exists, this will simply return the cached copy;
    // otherwise, it will create the proxy class via the ProxyClassFactory
    return proxyClassCache.get(loader, interfaces);
}
```

`WeakCache` 是JDK提供的弱缓存，是一个二级缓存。结构是 `(key, subKey) -> value` 。Proxy中使用了WeakCache缓存了代理对象。key就是代理对象的类加载器，subKey就是代理对象实现的接口由KeyFactory()生成，value就是生成的代理对象由ProxyClassFactory()生成。

```java
// K是Key类型
// P是参数类型
// V是值的类型
final class WeakCache<K, P, V> {
 
    private final ReferenceQueue<K> refQueue
        = new ReferenceQueue<>();
    // 代理缓存对象
    private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
    // reverseMap是用来实现缓存的有效性
    private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
        = new ConcurrentHashMap<>();
    private final BiFunction<K, P, ?> subKeyFactory;
    private final BiFunction<K, P, V> valueFactory;
 
   
    public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                     BiFunction<K, P, V> valueFactory) {
        this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
        this.valueFactory = Objects.requireNonNull(valueFactory);
    }
}
```

回到上面的 `proxyClassCache.get(loader, interfaces);` 方法

```java
// 实际上调用的就是WeakCache的get方法
// K为类加载器
// P为要代理类实现的接口
public V get(K key, P parameter) {
    Objects.requireNonNull(parameter);

    expungeStaleEntries();
    // 获取一级Key
    Object cacheKey = CacheKey.valueOf(key, refQueue);

    // lazily install the 2nd level valuesMap for the particular cacheKey
    // 获取一级key中的内容
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    if (valuesMap == null) {
    	// 没有就初始化
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }

    // 这部分就是调用生成sub-key的代码
    // private static final class KeyFactory
    //     implements BiFunction<ClassLoader, Class<?>[], Object>
    // {
    //    @Override
    //    public Object apply(ClassLoader classLoader, Class<?>[] interfaces) {
    //        switch (interfaces.length) {
    //            case 1: return new Key1(interfaces[0]); // the most frequent
    //            case 2: return new Key2(interfaces[0], interfaces[1]);
    //            case 0: return key0;
    //            default: return new KeyX(interfaces);
    //        }
    //    }
    // }
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    // 得到sub的Supplier
    Supplier<V> supplier = valuesMap.get(subKey);
    // supplier实际上就是这个factory
    Factory factory = null;

    while (true) {
    	// 如果已经缓存过Factory了，调用supplier.get获取这个对象
        if (supplier != null) {
            // supplier might be a Factory or a CacheValue<V> instance
            // 这个是核心方法，获取到V返回，而V就是生成的代理对象，下面分析
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // else no supplier in cache
        // or a supplier that returned null (could be a cleared CacheValue
        // or a Factory that wasn't successful in installing the CacheValue)
        // 下面的所有代码目的就是：如果缓存中没有supplier，则创建一个Factory对象，把factory对象在多线程的环境下安全的赋给supplier。
        // 因为是在while（true）中，赋值成功后又回到上面去调get方法，返回才结束。

        // lazily construct a Factory
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }

        if (supplier == null) {
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // successfully installed Factory
                supplier = factory;
            }
            // else retry with winning supplier
        } else {
            if (valuesMap.replace(subKey, supplier, factory)) {
                // successfully replaced
                // cleared CacheEntry / unsuccessful Factory
                // with our Factory
                supplier = factory;
            } else {
                // retry with current supplier
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

上面我们看到了关键代码 `V value = supplier.get();` 获取出来代理对象，而supplier是个Factory，下面我们看下这个get方法。关键代码是调用`valueFactory.apply()`方法。

```java
private final class Factory implements Supplier<V> {

    private final K key;
    private final P parameter;
    private final Object subKey;
    private final ConcurrentMap<Object, Supplier<V>> valuesMap;

    Factory(K key, P parameter, Object subKey,
            ConcurrentMap<Object, Supplier<V>> valuesMap) {
        this.key = key;
        this.parameter = parameter;
        this.subKey = subKey;
        this.valuesMap = valuesMap;
    }

    @Override
    public synchronized V get() { // serialize access
        // re-check
        Supplier<V> supplier = valuesMap.get(subKey);
        if (supplier != this) {
            // something changed while we were waiting:
            // might be that we were replaced by a CacheValue
            // or were removed because of failure ->
            // return null to signal WeakCache.get() to retry
            // the loop
            return null;
        }
        // else still us (supplier == this)

        // create new value
        V value = null;
        try {
        	// 关键代码是调用valueFactory.apply方法
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        } finally {
            if (value == null) { // remove us on failure
                valuesMap.remove(subKey, this);
            }
        }
        // the only path to reach here is with non-null value
        assert value != null;

        // wrap value with CacheValue (WeakReference)
        CacheValue<V> cacheValue = new CacheValue<>(value);

        // put into reverseMap
        reverseMap.put(cacheValue, Boolean.TRUE);

        // try replacing us with CacheValue (this should always succeed)
        if (!valuesMap.replace(subKey, this, cacheValue)) {
            throw new AssertionError("Should not reach here");
        }

        // successfully replaced us with new CacheValue -> return the value
        // wrapped by it
        return value;
    }
}
```

最终会调用到 `ProxyClassFactory.apply()` 方法，核心是 `ProxyGenerator.generateProxyClass(proxyName, interfaces, accessFlags);` 生成代理类的字节码，调用`native defineClass0()` 将class加载到jvm中。

```java
// 代理名字如$Proxy0、$Proxy1、$Proxy2等等
private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    // prefix for all proxy class names
    // 所有代理类名字的前缀
    private static final String proxyClassNamePrefix = "$Proxy";

    // next number to use for generation of unique proxy class names
    // 用于生成代理类名字的计数器
    private static final AtomicLong nextUniqueNumber = new AtomicLong();

    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        // 校验
        for (Class<?> intf : interfaces) {
            /*
             * Verify that the class loader resolves the name of this
             * interface to the same Class object.
             */
            Class<?> interfaceClass = null;
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            /*
             * Verify that the Class object actually represents an
             * interface.
             */
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            /*
             * Verify that this interface is not a duplicate.
             */
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }

        String proxyPkg = null;     // package to define proxy class in
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        /*
         * Record the package of a non-public proxy interface so that the
         * proxy class will be defined in the same package.  Verify that
         * all non-public proxy interfaces are in the same package.
         */
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }

        if (proxyPkg == null) {
            // if no non-public proxy interfaces, use com.sun.proxy package
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }

        /*
         * Choose a name for the proxy class to generate.
         */
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        /*
         * Generate the specified proxy class.
         */
        // 核心部分，生成代理类的字节码
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
        	// 把代理类加载到JVM中，至此动态代理过程基本结束了
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            /*
             * A ClassFormatError here means that (barring bugs in the
             * proxy class generation code) there was some other
             * invalid aspect of the arguments supplied to the proxy
             * class creation (such as virtual machine limitations
             * exceeded).
             */
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

编写看一下生成的字节码是啥
```java
import sun.misc.ProxyGenerator;

import java.io.FileOutputStream;
import java.io.IOException;
import java.lang.reflect.Modifier;

/**
 * @author weilai
 */
public class TestUserProxy {

    public static void main(String[] args) throws IOException {
        IUserService iUserService = new IUserServiceImpl();
        byte[] proxyClassByte = ProxyGenerator.generateProxyClass("$Proxy0", iUserService.getClass().getInterfaces(), Modifier.PUBLIC | Modifier.FINAL);
        FileOutputStream fileOutputStream = new FileOutputStream("$Proxy0.class");
        fileOutputStream.write(proxyClassByte);
        fileOutputStream.flush();
        fileOutputStream.close();
    }
}
```

下面我们看下这个字节码，$Proxy0继承了Proxy，实现了代理类接口。创建
```java
import com.weilai.springdemo.proxy.IUserService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements IUserService {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

	// 创建的时候传了一个InvocationHandler，就是我们的实现的InvocationHandler接口类
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

	// 这个就是我们要代理的方法
    public final void add(String var1) throws  {
        try {
        	// 我们可以看到实际是通过Proxy（super）获取了InvocationHandler对象（h）调用了我们实现的invoke方法
        	// m3从下面static代码块我们可以看到，实际上就是代理对象之前编写的add方法
        	// m3 = Class.forName("com.weilai.springdemo.proxy.IUserService").getMethod("add", Class.forName("java.lang.String"));
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.weilai.springdemo.proxy.IUserService").getMethod("add", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

**代理类为何要继承Proxy类？**
// TODO 目前不知道，希望大佬指点。

- 有人说因为性能，上面的比下面的生成要少。（我觉得是扯淡）
```java
public $Proxy0(InvocationHandler var1) throws  {
    super(var1);
}
```
```java
private InvocationHandler h;

public proxy(InvocationHandler h){
    this.h = h;
}
```
- 如果想知道这个类是一个经过代理产生的类，遍历所有类看是否有InvocationHandler对象，如果继承了Proxy就可以看是否是Proxy的子类。（感觉也有点扯）

