```java
//with返回一个RequestManager
public static RequestManager with(Activity activity) {
    return getRetriever(activity).get(activity);
}

//无论调用的是哪个with重载方法，最后都会到这里
public RequestManager get(Activity activity) {
    if (Util.isOnBackgroundThread()) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(activity, fm, null);
    }
}

//这里新建了一个没有视图的RequestManagerFragment 
private RequestManager fragmentGet(Context context,
                                   android.app.FragmentManager fm,
                                   android.app.Fragment parentHint) {
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        Glide glide = Glide.get(context);
　　　　 //绑定requestManager和Fragment的Lifecycle
        requestManager =
                factory.build(
                        glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
        current.setRequestManager(requestManager);
    }
    return requestManager;
}
```