---
title: Butterknife 源码浅析
date: 2020-01-21 18:07:28
categories: Android
tags:
---
# Butterknife 源码浅析

## 一、前言

Butterknife 依赖注入框架，简化手动书写 `findViewById` 等等，这里我们将深入源码来看看 Butterknife 内部的实现原理。

<!--more-->

## 二、基本用法

首先，再复习一下基本的用法。

```java
class MainActivity : AppCompatActivity() {

    @BindView(R.id.btn_jetpack) //2
    lateinit var btnJetPack: Button

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        ButterKnife.bind(this) //1
    }
}
```

可以看到，用 `@BindView` 注解替代了 `findViewById`，当然还有其他的用法，比如 `@OnClick` 注解替代 `setOnClickListener()` 等等，但其内部原理基本一致，所以这里我们仅对 `@BindView`  的内部实现来解析 Butterknife 的实现。

看到这里，你可能会有疑问，注解是什么？为什么通过一个简单的注解就能替代简化代码呢？

在进入源码之前，我们得先了解一下注解。

首先，得先知道 APT(Annotation Processing Tool) 注解处理工具，网上其实有很多教程之类的，我们这里便不再深入，简单的了解一下 APT 的实现步骤以及其作用时机。

* 先声明一个注解 `@interface`，定义注解的生命周期 `@Retention` 和作用区域 `@Target`；
* 然后定义一个注解处理器，继承并实现 `AbstractProcessor` 中的方法；
* 最后，代码在编译时，编译器会扫描所有带注解的类，通过定义的注解处理器来生成相应的模板 Java 类。

下面，我们将进入 Butterknife 的源码世界。

## 三、源码浅析

### 3.1 Butterknife.bind()

我们先来看看 `ButterKnife.bind(this)` 这里面到底做了什么处理。

```java
//ButterKnife.java
public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return bind(target, sourceView);
}

public static Unbinder bind(@NonNull Object target, @NonNull View source) {
  Class<?> targetClass = target.getClass();

  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

  if (constructor == null) {
    return Unbinder.EMPTY;
  }

  //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
  try {
    return constructor.newInstance(target, source);
  }
  //...
}
```

代码清晰简单，主要是找到目标类的构造器 `constructor` ，然后通过反射的方式实例化。

```java
//ButterKnife.java
static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
  Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
  if (bindingCtor != null || BINDINGS.containsKey(cls)) {
    return bindingCtor;
  }
  String clsName = cls.getName();
  if (clsName.startsWith("android.") || clsName.startsWith("java.")
      || clsName.startsWith("androidx.")) {
    return null;
  }
  try {
    Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
    //noinspection unchecked
    bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
  } catch (ClassNotFoundException e) {
    bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
  } catch (NoSuchMethodException e) {
    throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
  }
  BINDINGS.put(cls, bindingCtor);
  return bindingCtor;
}
```

首先，使用了 `LinkedHashMap` 作为缓存，避免每次重新加载。然后过滤了系统相关类，如果是 `android.`、 `java.` 、`androidx.` 开头的不作处理。最后通过类加载器加载对应的 `_ViewBinding` 类，比如这边的例子中应该是 `MainActivity_ViewBinding` 类。

到这里，我们就会产生一些疑问，我们在例子里都没写过 `MainActivity_ViewBinding` 这个类，那它从哪里生成的呢？又是做什么的？

在 AS 中搜索一下，可以发现在 `/app/build/generated/source/kapt/debug/包名/` 目录下找到了此类。

```java
public final class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.btnJetPack = Utils.findRequiredViewAsType(source, R.id.btn_jetpack, "field 'btnJetPack'", Button.class);
  }

  @Override
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.btnJetPack = null;
  }
}
```

可以发现代码非常简单，在构造方法中寻找对应的控件。这边有一点稍微注意一下，我们在使用 Butterknife 的时候，控件的修饰符不能为 `private` 的原因就在这里，是直接赋值而不是通过 set 方式。

```java
//Utils.java
public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
    Class<T> cls) {
  View view = findRequiredView(source, id, who);
  return castView(view, id, who, cls);
}

public static View findRequiredView(View source, @IdRes int id, String who) {
  View view = source.findViewById(id);
  if (view != null) {
    return view;
  }
  //...
}

public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
  try {
    return cls.cast(view);
  }
  //...
}
```

最终也是通过 `findViewById` 来寻找控件。

到这里，我们可以知道生成的以 `_ViewBinding` 结尾的类主要是用来寻找被 `@BindView` 注解标识的控件。那么该类是在哪里生成的呢？

### 3.2 ButterKnifeProcessor

在前面，我们说过 APT 需要声明一个注解以及一个注解处理器，以便编译器在编译的时候作相应的处理。

```java
@Retention(RUNTIME) @Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```

声明了一个 `BindView` 的注解。

```java
//ButterKnifeProcessor.java
@Override
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

  for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
    TypeElement typeElement = entry.getKey();
    BindingSet binding = entry.getValue();

    JavaFile javaFile = binding.brewJava(sdk, debuggable);
    try {
      javaFile.writeTo(filer);
    }
    //...
  }
  return false;
}

private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
  Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
  Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

  //...

  // Process each @BindView element.
  for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
    // we don't SuperficialValidation.validateElement(element)
    // so that an unresolved View type can be generated by later processing rounds
    try {
      parseBindView(element, builderMap, erasedTargetNames);
    } catch (Exception e) {
      logParsingError(element, BindView.class, e);
    }
  }

  //...

  return bindingMap;
}

private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
    Set<TypeElement> erasedTargetNames) {
  TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

  //...

  // Assemble information on the field.
  int id = element.getAnnotation(BindView.class).value();
  BindingSet.Builder builder = builderMap.get(enclosingElement);
  Id resourceId = elementToId(element, BindView.class, id);
  if (builder != null) {
    String existingBindingName = builder.findExistingBindingName(resourceId);
    //...
  } else {
    builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
  }

  String name = simpleName.toString();
  TypeName type = TypeName.get(elementType);
  boolean required = isFieldRequired(element);

  builder.addField(resourceId, new FieldViewBinding(name, type, required));

  // Add the type-erased version to the valid binding targets set.
  erasedTargetNames.add(enclosingElement);
}

private BindingSet.Builder getOrCreateBindingBuilder(
    Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
  BindingSet.Builder builder = builderMap.get(enclosingElement);
  if (builder == null) {
    builder = BindingSet.newBuilder(enclosingElement);
    builderMap.put(enclosingElement, builder);
  }
  return builder;
}
```

`ButterKnifeProcessor` 的代码这里精简了很多，抓主流程就好，更多的细节如果有兴趣可以自己跟下源码。在 `process()` 方法中，主要是寻找被  `BindView` 注解标识的类存到集合中，最后循环取出通过 `javapoet` 生成 `_ViewBinding`  模板代码类。

## 四、总结

通过上面的简单的源码解析，我们大概清楚了 Butterknife 的实现原理。当然 Butterknife 并不仅仅这么简单，还有其他的功能，但原理是一样的。

最后，我们简单做个总结：

1. 首先，在编译期间扫描声明的注解（如 `BindView` ），通过 `ButterKnifeProcessor` 注解处理器相应的解析以及调用 `Javapoet` 库生成 `_ViewBinding` 模板代码。
2. 然后，在我们调用 `ButterKnife.bind(this)` 方法的时候，通过类加载器加载指定的类（如 `MainActivity_ViewBinding` ），并通过反射方式实例化该模板类，从而实现了对标识注解的控件赋值。

## 五、参考与推荐

1. [ButterKnife](https://github.com/JakeWharton/butterknife/)
