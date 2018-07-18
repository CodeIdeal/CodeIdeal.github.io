title: 修改View Background Color导致的坑
date: 2016-10-23 23:03:52
---
之前有遇到一个十分神奇的bug，不知道是什么原因突然整个项目所有引用某个颜色资源Id的地方突然就变成了透明色。而这个bug复现的时机又不是很清楚，重启应用又会恢复正常，用着用着又会出现，着实让我摸不着头脑。

原来是前不久做了一个标题栏的属性动画，动画中一个View的Alpha值会从透明到纯白或从纯白到透明，而这个纯白色#FFFFFF的资源id有很多地方都有用到，也就是引用了这个颜色资源id的地方会突然变成透明色。

这个属性动画是随着页面滚动标题栏从透明过渡到白色背景色的实现，类似于 Design 库中 AppBar 的那种效果。实现方法是这样的：

```
// 滚动监听器 伪代码
onscroll(){
	percent = ....
  	view.getBackground().setAlpha(percent);
}
```

当我意识到跟这个资源ID有关时，便Google了一下，果然这里面别有玄机。

StackOverflow中高赞回答提到: view.getBackground() 获得的是一个 ColorDrawable，如果给这个 ColorDrawable 设置 Alpha 值的话，会影响所有设置 background 为这个颜色资源ID的背景色的 Alpha 值。

再到Android文档中去看了看Drawable的相关文档，验证了之前的说法，从同一资源加载的 drawable 确实会共享状态，不过有一个 mutate 方法来禁用这一特性。文档如下：

> [Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html#mutate()) mutate ()
>
> Make this drawable mutable. This operation cannot be reversed. A mutable drawable is guaranteed to not share its state with any other drawable. This is especially useful when you need to modify properties of drawables loaded from resources. By default, all drawables instances loaded from the same resource share a common state; if you modify the state of one instance, all the other instances will receive the same modification. Calling this method on a mutable Drawable will have no effect.
>
> **翻译：**
>
> Drawable mutate ()
>
> 让一个 Drawable 变为 mutable 的。这个操作是不可逆的。一个 mutable 的 drawable 可以保证不会分享自己的状态给其他 drawable。当一个 drawable 是从 resource 加载的，在需要更改它状态时这个方法特别有用。在默认情况下，所有从相同 resource 的 drawable 的实例是共享一个通用状态的；如果你修改了其中一个的状态，所有其他的实例也会收到相同的改动。在一个已经是可变的 drawable 上调用该方法没有效果。

所以，上述代码只要在 drawable 获取之后，调用一下 mutate() 方法即可(如下)。

```java
// 滚动监听器 伪代码
onscroll(){
	percent = ....
  	view.getBackground().mutate().setAlpha(percent);
}
```

