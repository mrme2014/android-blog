---
title: Kotlin实用技巧
---
<!--more-->

## 目录

- 使用 Kotlin 安卓扩展，向findViewById说拜拜
- 字符串的空判断，向TextUtils.isEmpty说拜拜
- 使用@JvmOverloads告别繁琐的构造函数重载


## 使用 Kotlin 安卓扩展，向findViewById说拜拜

在进行Android 编码时我们避免不了的需要使用`findViewById()`来获取指定控件的对象，下面是摘自我们的底部导航组件`HiTabBottom.java`中的一段代码：

```java
tabImageView = findViewById(R.id.iv_image);
tabIconView = findViewById(R.id.tv_icon);
tabNameView = findViewById(R.id.tv_name);
groupView = findViewById(R.id.rl_root);
```


从上面代码中不难看出我们应用了大量的`findViewById(R.id.xx)`的模板代码，那是否能有一种方式能帮我们减少这些重复劳动呢？

你能会想到JakeWharton大神的"黄油刀"：

```java
@BindView(R.id.iv_image) ImageView tabImageView;
```

虽然黄油刀能帮我们省去`findViewById(R.id.xx)`的模板代码，但它还是需要我们使用@BindView(R.id.xxx)的这样的模块代码，那有没有一种方式能够使我们直接获取
xml中的控件而不用先获取在使用呢？

答案是有的。

在Kotlin中这些看似离谱的要求，也是可以实现的，Kotlin为Android开发者提供了安卓扩展，安卓扩展是 IntelliJ IDEA 与 Android Studio 的 Kotlin 插件的组成之一，因此不需要再单独安装额外插件。

我们仅需要在模块的 `build.gradle` 文件中启用 Gradle 安卓扩展插件即可：

```
apply plugin: 'kotlin-android-extensions'
```

### 如何使用呢？

仅需要一行即可非常方便导入指定布局文件中所有控件属性：

```java
import kotlinx.android.synthetic.main.＜布局＞.*
```

因此，如果布局文件名是 `hi_tab_bottom.xml`，我们将导入 `kotlinx.android.synthetic.main.hi_tab_bottom.*`。

若需要调用 View 的合成属性，同时还应该导入 `kotlinx.android.synthetic.main.hi_tab_bottom.view.*`。

导入完成后即可调用在xml文件中以视图控件具名属性的对应扩展，比如下例:

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/rl_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent">


    <ImageView
        android:id="@+id/iv_image"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_above="@id/tv_name"
        android:layout_centerHorizontal="true"
        android:layout_marginBottom="4dp"
        android:scaleType="centerInside"
        android:visibility="gone" />


    <TextView
        android:id="@+id/tv_icon"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_above="@id/tv_name"
        android:layout_centerHorizontal="true"
        android:layout_marginBottom="4dp"
        android:textColor="#666666"
        android:textSize="25dp"
        android:textStyle="bold"
        android:visibility="gone" />

    <TextView
        android:id="@+id/tv_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true"
        android:layout_marginBottom="7dp"
        android:includeFontPadding="false"
        android:textColor="#999999"
        android:textSize="10dp" />

</RelativeLayout>
```

接下来我们就可以通过空间id来访问这些控件的实例了：

```java
iv_image.visibility = View.GONE
tv_icon.visibility = View.VISIBLE
```

## 字符串的空判断，向TextUtils.isEmpty说拜拜

在我们日常开发中，经常会对字符串进行空判断，相信大家对`TextUtils.isEmpty`一定都不陌生，它可以帮我们判断字符串是否为空而且不用担心npe问题。

那么在Kotlin中，有一个叫：

```kotlin
public inline fun CharSequence?.isNullOrEmpty(): Boolean = this == null || this.length == 0
```

的扩展函数能帮我们省去对`TextUtils.isEmpty`的使用：

>Java

```java
if (!TextUtils.isEmpty(tabInfo.name)) {
    tabNameView.setText(tabInfo.name);
}
```

>Kotlin

```kotlin
if (!tabInfo.name.isNullOrEmpty()) {
    tabNameView.text = tabInfo.name;
}
```

除了`isNullOrEmpty`扩展之外，CharSequence还有个名叫：

```kotlin
public inline fun CharSequence?.isNullOrBlank(): Boolean = this == null || this.isBlank()
```


如果 name 都是空格，则 TextUtils.isEmpty 不满足使用。那 isNullOrBlank 可用。


## 使用@JvmOverloads告别繁琐的构造函数重载

在Kotlin中@JvmOverloads注解的作用就是：在有默认参数值的方法中使用@JvmOverloads注解，则Kotlin就会暴露多个重载方法。
这对我们自定义控件时特别有用，不信你看：

>Java

```java
public class HiTabBottomLayout extends FrameLayout {
   ...
    public HiTabBottomLayout(@NonNull Context context) {
        this(context, null);
    }

    public HiTabBottomLayout(@NonNull Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public HiTabBottomLayout(@NonNull Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
   ...
```

>Kotlin

```kotlin
class HiTabBottom @JvmOverloads constructor(
    context: Context?,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : RelativeLayout(context, attrs, defStyleAttr)
```

上面是摘自我们的底部导航组件`HiTabBottom.java`中的一段代码，在用kotlin重写有，我们只需要在我们的主构造方法中使用`@JvmOverloads`，便可以实现一个顶三个的效果是不是很不可思议呢。





