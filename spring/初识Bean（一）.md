作为一名pythoner，在学习java的过程中，最开始还是有些渐入佳境的感觉的，因为语言之间有太多相似性，java的很多概念在python中都能找到对应的。
比如，python的threading模块的源码第一行就是这样一条注释：

```
"""Thread module emulating a subset of Java's threading model."""
```

所以在阅读java的`java.lang.Thread`的源码时，有种强烈的即视感，这极大减轻了我作为一名pythoner学习java的难度。其他类似的例子还有很多，如`java.util.concurrent.ThreadPoolExecutor`和python的`concurrent.futures.ThreadPoolExecutor`，这两个类的类名、设计思路甚至于一些方法名都是一样的……

所以在遇到了`Bean`这一概念之前，学习java过程中的一切都还是美好的。

在艰难的阅读sping源码的过程中，慢慢地我对`Bean`也开始有了初步的理解，没有了初见时那种挥之不去的陌生感和困惑感。所以这里写篇文章记录一下，既作为源码学习过程中的阶段性总结，也作一点分享。

一开始接触spring相关开发时，头脑中满是疑惑，很多代码对我这个pythoner来说简直不可思议，比如

为什么类名上面加上注解`Component`就是一个`Bean`？（实际上这样表述并不准确）
```java
@Component
class A {

}
```

为什么`@Autowired`就能将其它`Bean`注入进来？
为什么`@Value`能给`value`赋值？
```java
@Service
class B {
    @Autowired
    private A a;

    @Value("${xxx}")
    private String value;
}
```

```java
@Repository
public interface UserDAO {

    int insert(User user);

}

@Component
class C {
    @Autowired
    private UserDAO userDao:

}
```
为什么在使用mybatis时，明明没有写`UserDAO`的具体子类实现，`C`也能注入`UserDAO`的实例？

为什么……
为什么……
为什么……

种种疑问，在经历了最近这一段时间对spring源码的阅读后，终于有了答案。当然，因为spring的源码量实在不小，有很多细节、流程、具体实现等，我并没有深入研究，这些需要后续不断的学习。

实际上，当阅读过`Bean`实例的实例化过程中的相关源码后，`Bean`就已经不那么陌生和神秘了。

先从一段代码说起
```java
// 源码见 AnnotatedBeanDefinitionReader#doRegisterBean
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
        @Nullable BeanDefinitionCustomizer[] customizers) {

    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```
上面是类`AnnotatedBeanDefinitionReader`中方法，在这个方法中，需要特别留意`AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass)`这个实例, `this.scopeMetadataResolver.resolveScopeMetadata(abd)`读取`beanClass`的`@Scope`注解获取`abd`实例的`scope`属性。类似的，在`AnnotationConfigUtils.processCommonDefinitionAnnotations(abd)`方法中读取`@Lazy`获取`lazyInit`属性，读取`@Primary`获取`primary`属性，读取`DependsOn`获取`dependsOn`属性等等。

这些属性都是`BeanDefinition`的属性，`BeanDefinition`是对`Bean`的定义进行描述的实例对象。

简单来说，spring有三大组件，Core(`org.springframework.core`)、Context(`org.springframework.context`) 和 Beans(`org.springframework.beans`)，Context中的类`ClassPathBeanDefinitionScanner`和`AnnotatedBeanDefinitionReader`读取相应包的相应类（如该类有`Component`注解），解析该类的注解（解析实现见Core），生成`BeanDefinition`实例，所有这些实例都注册在Beans里的`DefaultListableBeanFactory`实例中。

所以`Bean`的实例化是在 `DefaultListableBeanFactory` 中发生的。`DefaultListableBeanFactory` 是 `BeanFactory` 子类实现，`BeanFactory` 还有其他实现，一般开发中用到的应该是 `DefaultListableBeanFactory` 。

`DefaultListableBeanFactory` 里有个 `beanDefinitionMap` 属性：
```java
/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

`BeanDefinition`实例对象管理在这个 `Map` 中，当调用 `DefaultListableBeanFactory` 的 `getBean(String name)`时，首先会从这个 `Map` 中获取对应的 `BeanDefinition` 实例对象，再根据该实例对象生成对应的 `Bean` 实例。
但不得不说，这个实例化的过程具体实现非常非常复杂，源码量很庞大，很多具体的细节实现单独拎出来都能又写一篇心得了。不过整个过程的阶段还是相对比较清晰的，这里梳理了这个过程中的一部分阶段（部分阶段的实现还未细看）：

1. 首先解析 `name`，将其转化为真正的 `beanName`，因为 `BeanDefinition`是可以有别名的
```java
// 源码见 AbstractBeanFactory#doGetBean
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name);
    // ...
}
```

2. 检查有没有已经实例化过的 Singleton 实例，如果有，将不会执行下面的实例化过程，这里就将直接返回（实际上还有一小段处理，涉及 `FactoryBean`，这里先不展开说了）
```java
// 源码见 AbstractBeanFactory#doGetBean
Object sharedInstance = getSingleton(beanName)
```

3. 如果没有 `beanName` 对应的 `BeanDefinition` 对象时，会调用父 `BeanFactory` 对象的 `getBean` 方法获取
```java
// 源码见 AbstractBeanFactory#doGetBean
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // ...
}
```

4. 读取 `BeanDefinition` 对象的 `dependsOn` 属性，如果有依赖的 `Bean`，注册依赖关系，并且先实例化这些 `Bean`
```java
// 源码见 AbstractBeanFactory#doGetBean
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dep : dependsOn) {
        if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        }
        registerDependentBean(dep, beanName);
        try {
            getBean(dep);
        }
        catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
        }
    }
}
```

5. 读取 `BeanDefinition` 的 `scope` 属性，如果是 Singleton，则实例化的对象会保存起来，下次再调用 `getBean` 方法时，直接返回该对象。如果是 Prototype，则每次调用 `getBean` 方法生成新的对象。
```java
// 源码见 AbstractBeanFactory#doGetBean
if (mbd.isSingleton()) {
    // 默认 scope 为 Singleton
    // ...
}

else if (mbd.isPrototype()) {
    // ...
}

else {
    // 自定义的scope实现，见 org.springframework.beans.factory.config.Scope
    // ...
}
```

6. 重头戏，根据 `BeanDefinition` 对象里的信息生成实例对象，见 `AbstractAutowireCapableBeanFactory#createBean`

6.1 根据 `BeanDefinition` 获取类信息
```java
// 源码见 AbstractAutowireCapableBeanFactory#createBean
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
```

6.2 有一些非常规的实例化方式，比如会检查 `DefaultListableBeanFactory` 里的 BeanPostProcessors 能不能直接返回对象（`BeanPostProcessor`也是重要概念）这里实例化的对象会在 `createBean`方法中直接返回。不同于其他方式，还会经历后续一系列处理
```java
// 源码见 AbstractAutowireCapableBeanFactory#createBean
try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
        return bean;
    }
}
catch (Throwable ex) {
    throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
}
```

6.3 常规方式获得实例化对象，有以下几种：

6.3.1 如果 `BeanDefinition` 里直接定义了 Supplier，直接调用。（这里的Supplier 有点类似于python里的 Callable）
```java
// 源码见 AbstractAutowireCapableBeanFactory#createBeanInstance
Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
if (instanceSupplier != null) {
    return obtainFromSupplier(instanceSupplier, beanName);
}
```

6.3.2 根据 `BeanDefinition` 里的 beanClass 等信息进行实例化。这里的代码解释了 `BeanDefinition` 的 beanClass 是 `Interface` 的情况下为什么也能生成实例对象
```java
// 源码见 SimpleInstantiationStrategy#instantiate 
if (!bd.hasMethodOverrides()) {
    // 1. 先根据JAVA反射机制获取 Constructor
    Constructor<?> constructorToUse;
    synchronized (bd.constructorArgumentLock) {
        constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
        if (constructorToUse == null) {
            final Class<?> clazz = bd.getBeanClass();
            // Interface 这里无法实例化
            if (clazz.isInterface()) {
                throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
                if (System.getSecurityManager() != null) {
                    constructorToUse = AccessController.doPrivileged(
                            (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
                }
                else {
                    constructorToUse = clazz.getDeclaredConstructor();
                }
                bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
                throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
        }
    }
    // 2. Constructor -> instance，获得实例对象，个人感觉有点类似于python里的 cls.__new__(cls)
    return BeanUtils.instantiateClass(constructorToUse);
}
else {
    // BeanDefinition 的 beanClass 是 Interface 的，只要同时定义了 methodOverrides，利用 CGLIB 可以动态生成 subclass，从而生成实例！
    // CGLIB 生成子类实例的具体实现见 CglibSubclassingInstantiationStrategy
    // Must generate CGLIB subclass.
    return instantiateWithMethodInjection(bd, beanName, owner);
}
``` 

6.3.3 上面的实例化中的 `Constructor` 的参数个数为0，如果是下面的情形实例化会有所不同
```java
@Component
class D {

    private A a;

    // D 的 Constructor 的参数个数为1
    public D(A a) {
        this.a = a;
    }

}
```
```java
// 源码见 AbstractAutowireCapableBeanFactory#autowireConstructor

// 如果 Constructor 的参数个数不为0时，实例化调用该方法
// 简单来说，这个方法会 根据 BeanDefinition 里的 constructorArgumentValues 获取参数对应的对象实例，如果没有
// 则根据参数类型，去匹配该类型的 Bean，实例化后作为参数实例
protected BeanWrapper autowireConstructor(
        String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

    return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

7. 实例化之后的后续处理。常规方式实例化之后，还有一系列后续处理

比如调用 DefaultListableBeanFactory 里的 BeanPostProcessors 对实例化对象进行处理
```java
// 源码见 AbstractAutowireCapableBeanFactory#populateBean
for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse == null) {
            if (filteredPds == null) {
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                return;
            }
        }
        pvs = pvsToUse;
    }
}
```
比如`AutowiredAnnotationBeanPostProcessor` 也是一种 `BeanPostProcessor` ，将 `@Autowired`，`@Value`注解的Field注入相应值，如下列情形：

```java
@Service
class B {
    // AutowiredAnnotationBeanPostProcessor 会找到类是 A 的 Bean，将该 Bean 的实例注入进来
    @Autowired
    private A a;

    // 注入属性值
    @Value("${xxx}")
    private String value;
}
```

`ApplicationContextAwareProcessor` 发现 Bean 实例为 `EnvironmentAware`时调用实例的 `setEnvironment` 方法，实例为 `ApplicationContextAware`时调用实例的 `setApplicationContext` 方法等
```java
// 源码见 ApplicationContextAwareProcessor#invokeAwareInterfaces
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
}
```
所以，以下情形的 Bean 实例将获得 `ApplicationContext` 对象
```java
@Component
class E implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    
}
```
