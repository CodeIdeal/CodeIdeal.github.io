---
title: '从View#setTranslationX/Y到自定义属性动画'
date: 2017-07-16 11:28:38
tags:
---

## 属性动画简单源码解析
在Android Reference中的定义
>Sets the vertical/horizontal location of this view relative to its top/left position. This effectively positions the object post-layout, in addition to wherever the object's layout placed it.

设置此视图相对于其左侧位置的水平位置。 除了View自身的Layout之外，这有效地定位了在Layout后的View。

那这个方法有什么用处呢？我们都知道在属性动画中也有一对transationX/Y,在属性动画中做X/Y方向上的平移动画时是这样的：

```java
ObjectAnimator objectAnimator=ObjectAnimator.ofFloat(btn,"translationX/Y",0f,200f);
objectAnimator.setDuration(1000);
objectAnimator.start();
```

我们从Android源码来看看，动画的每一帧的位置是如何确定的：

首先我们在ObjectAnimation中找到这样一个方法：

```java
    /**
     * This method is called with the elapsed fraction of the animation during every
     * animation frame. This function turns the elapsed fraction into an interpolated fraction
     * and then into an animated value (from the evaluator. The function is called mostly during
     * animation updates, but it is also called when the <code>end()</code>
     * function is called, to set the final value on the property.
     *
     * <p>Overrides of this method must call the superclass to perform the calculation
     * of the animated value.</p>
     *
     * @param fraction The elapsed fraction of the animation.
     */
    @CallSuper
    @Override
    void animateValue(float fraction) {
        final Object target = getTarget();
        if (mTarget != null && target == null) {
            // We lost the target reference, cancel and clean up. Note: we allow null target if the
            /// target has never been set.
            cancel();
            return;
        }

        super.animateValue(fraction);
        int numValues = mValues.length;
        for (int i = 0; i < numValues; ++i) {
            mValues[i].setAnimatedValue(target);
        }
    }
```

通过注释我们可以知道,这个方法在动画的每一帧被调用，将进度分数通过差值器转化为插值分数，最后再将分数转化为对应的属性动画的数值,再设置给对应的属性。

再通过`mValues[i].setAnimatedValue(target);`跳转到下面的函数：

```java
/**
     * Internal function to set the value on the target object, using the setter set up
     * earlier on this PropertyValuesHolder object. This function is called by ObjectAnimator
     * to handle turning the value calculated by ValueAnimator into a value set on the object
     * according to the name of the property.
     * @param target The target object on which the value is set
     */
    void setAnimatedValue(Object target) {
        if (mProperty != null) {
            mProperty.set(target, getAnimatedValue());
        }
        if (mSetter != null) {
            try {
                mTmpValueArray[0] = getAnimatedValue();
                mSetter.invoke(target, mTmpValueArray);
            } catch (InvocationTargetException e) {
                Log.e("PropertyValuesHolder", e.toString());
            } catch (IllegalAccessException e) {
                Log.e("PropertyValuesHolder", e.toString());
            }
        }
    }
```
这是PropertyValuesHolder源码中一段核心的代码，他的主要目的即是，将计算出的当前帧的属性数值赋值到做动画的View中。
我们可以看到这里在拿到当前帧的数值后，通过反射调用mSetter将数值赋值给对应的方法。

```java
    /**
     * Utility function to get the setter from targetClass
     * @param targetClass The Class on which the requested method should exist.
     */
    void setupSetter(Class targetClass) {
        Class<?> propertyType = mConverter == null ? mValueType : mConverter.getTargetType();
        mSetter = setupSetterOrGetter(targetClass, sSetterPropertyMap, "set", propertyType);
    }
```

那么mSetter方法是怎么来的呢？我继续往下找,在一个`PropertyValuesHolder#setupSetter()`方法中找到了mSetter的赋值逻辑，而`setupSetter()`方法又是在`PropertyValuesHolder#setupSetterAndGetter()`中被调用的，`setupSetterAndGetter()`方法是在`ObjectAnimator#initAnimation()`中被调用的。

那么也即是在属性动画的初始化方法中，我们就通过一系列的调用，最终在`setupSetter()`方法中拿到了设置对相应的属性值的方法对象.

我们再看`setupSetter()`方法中，mSetter是通过一个`setupSetterOrGetter()`函数得到的,而且这里的入参`prefix`传入了一个"set"。

我们先不管这个"set"是干嘛的，继续往下看：

```java
    /**
     * Returns the setter or getter requested. This utility function checks whether the
     * requested method exists in the propertyMapMap cache. If not, it calls another
     * utility function to request the Method from the targetClass directly.
     * @param targetClass The Class on which the requested method should exist.
     * @param propertyMapMap The cache of setters/getters derived so far.
     * @param prefix "set" or "get", for the setter or getter.
     * @param valueType The type of parameter passed into the method (null for getter).
     * @return Method the method associated with mPropertyName.
     */
    private Method setupSetterOrGetter(Class targetClass,
            HashMap<Class, HashMap<String, Method>> propertyMapMap,
            String prefix, Class valueType) {
        Method setterOrGetter = null;
        synchronized(propertyMapMap) {
            // Have to lock property map prior to reading it, to guard against
            // another thread putting something in there after we've checked it
            // but before we've added an entry to it
            HashMap<String, Method> propertyMap = propertyMapMap.get(targetClass);
            boolean wasInMap = false;
            if (propertyMap != null) {
                wasInMap = propertyMap.containsKey(mPropertyName);
                if (wasInMap) {
                    setterOrGetter = propertyMap.get(mPropertyName);
                }
            }
            if (!wasInMap) {
                setterOrGetter = getPropertyFunction(targetClass, prefix, valueType);
                if (propertyMap == null) {
                    propertyMap = new HashMap<String, Method>();
                    propertyMapMap.put(targetClass, propertyMap);
                }
                propertyMap.put(mPropertyName, setterOrGetter);
            }
        }
        return setterOrGetter;
    }

```

这个方法中，首先在`propertyMap`中去找之前的缓存，如果没有就通过`getPropertyFunction()`方法得到Method对象，并将其放入`propertyMap`缓存中。

得，继续看`getPropertyFunction()`这个方法：

```java
    /**
     * Determine the setter or getter function using the JavaBeans convention of setFoo or
     * getFoo for a property named 'foo'. This function figures out what the name of the
     * function should be and uses reflection to find the Method with that name on the
     * target object.
     *
     * @param targetClass The class to search for the method
     * @param prefix "set" or "get", depending on whether we need a setter or getter.
     * @param valueType The type of the parameter (in the case of a setter). This type
     * is derived from the values set on this PropertyValuesHolder. This type is used as
     * a first guess at the parameter type, but we check for methods with several different
     * types to avoid problems with slight mis-matches between supplied values and actual
     * value types used on the setter.
     * @return Method the method associated with mPropertyName.
     */
    private Method getPropertyFunction(Class targetClass, String prefix, Class valueType) {
        // TODO: faster implementation...
        Method returnVal = null;
        String methodName = getMethodName(prefix, mPropertyName);
        Class args[] = null;
        if (valueType == null) {
            try {
                returnVal = targetClass.getMethod(methodName, args);
            } catch (NoSuchMethodException e) {
                // Swallow the error, log it later
            }
        } else {
            args = new Class[1];
            Class typeVariants[];
            if (valueType.equals(Float.class)) {
                typeVariants = FLOAT_VARIANTS;
            } else if (valueType.equals(Integer.class)) {
                typeVariants = INTEGER_VARIANTS;
            } else if (valueType.equals(Double.class)) {
                typeVariants = DOUBLE_VARIANTS;
            } else {
                typeVariants = new Class[1];
                typeVariants[0] = valueType;
            }
            for (Class typeVariant : typeVariants) {
                args[0] = typeVariant;
                try {
                    returnVal = targetClass.getMethod(methodName, args);
                    if (mConverter == null) {
                        // change the value type to suit
                        mValueType = typeVariant;
                    }
                    return returnVal;
                } catch (NoSuchMethodException e) {
                    // Swallow the error and keep trying other variants
                }
            }
            // If we got here, then no appropriate function was found
        }

        if (returnVal == null) {
            Log.w("PropertyValuesHolder", "Method " +
                    getMethodName(prefix, mPropertyName) + "() with type " + valueType +
                    " not found on target class " + targetClass);
        }

        return returnVal;
    }
```

我们先看看注释：

>JavaBeans约定：名为foo的属性对应setFoo或getFoo这样两个设置和获取属性的方法，用这样的约定来确定setter或getter函数。
>此函数指出函数的名称应该是什么，并使用反射在目标对象上查找具有该名称的Method。

再看其中的逻辑

其中的`getMethodName()`方法,将属性名的第一位字母大写，然后在前面拼上`prefix`前缀，得到最后的符合JavaBeans约定的驼峰方法名，即方法名一定是setXxx()。
```java
    static String getMethodName(String prefix, String propertyName) {
        if (propertyName == null || propertyName.length() == 0) {
            // shouldn't get here
            return prefix;
        }
        char firstLetter = Character.toUpperCase(propertyName.charAt(0));
        String theRest = propertyName.substring(1);
        return prefix + firstLetter + theRest;
    }
```

而真正的获取方法对象的逻辑中，通过顺序匹配数值类型来解决了传入的数值类型不匹配的问题。

最后通过`Class.getMethod(methodName, args)`方法找到对应的方法对象。其中args是只有一个元素的class数组，所以我们知道ObjectAnimator支持的set方法只能有一个参数。

## 自定义属性动画

首先新建一个继承自TextView的自定义View, 并提添加一个setXxx的方法，这里我们设置一个给文字设置颜色的方法：
```java
public void setColor(@ColorInt int value){
    setTextColor(value);
    postInvalidate();
}
```

在MainActivity中设置自定义的属性动画：
```java
...
    private int[] colors;

    {
        int 赤 = Color.rgb(255, 0, 0);

        int 橙 = Color.rgb(255, 165, 0);

        int 黄 = Color.rgb(255, 255, 0);

        int 绿 = Color.rgb(0, 255, 0);

        int 青 = Color.rgb(0, 127, 255);

        int 蓝 = Color.rgb(0, 0, 255);

        int 紫 = Color.rgb(139, 0,255);

        colors = new int[]{赤, 橙, 黄, 绿, 青, 蓝, 紫};
    }
...

    ObjectAnimator objectAnimator = ObjectAnimator.ofInt(text,"color",colors);
    objectAnimator.setRepeatCount(ValueAnimator.INFINITE);
    objectAnimator.setRepeatMode(ValueAnimator.REVERSE);
    objectAnimator.setDuration(1000);
    objectAnimator.setEvaluator(new ArgbEvaluator());
    objectAnimator.start();
```
这里我们用了一个`ArgbEvaluator`估值器，他是`TypeEvaluator`接口的实现类，专为ARGB颜色计算过渡值。

实现的效果如下:

![](https://i.ooxx.ooo/2018/07/17/1e5ccc40b38d25c14e9106e4a2f23ddd.gif)
