LayoutInflater 是一个抽象类，在加载ComtextImpl类时，会调用SystemServiceRegistry的方法，同时加载SystemServiceRegistry类，在SystemServiceRegistry的static方法中注册一个叫做 

```
//Context.java
public static final String LAYOUT_INFLATER_SERVICE = "layout_inflater";
```

的service。

```

// Service registry information.
// This information is never changed once static initialization has completed.
private static final HashMap<Class<?>, String> SYSTEM_SERVICE_NAMES =
            new HashMap<Class<?>, String>();
private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new HashMap<String, ServiceFetcher<?>>();

registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
            
            
private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```

而平时所用的

```
LayoutInflater inflater = LayoutInflater.from(context);


public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
}
```

通过源码可以发现，实际实现的类是 **PhoneLayoutInflater**

```
//PhoneLayoutInflater.java
private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
};

 @Override 
 protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
                // In this case we want to let the base class take a crack
                // at it.
            }
        }

        return super.onCreateView(name, attrs);
}
```

实际上，createView会通过反射创建Android提供的View或用户自定义View（这是为什么我们的自定义view在xml中必须包含包名，因为createView会验证xml中的tag是否包含`.`）
