---
layout: post
title: Cryptic Stacktrace from Android
date: 2010-03-29 01:04
author: matejoseph
comments: true
categories: [Android]
---
I was building a new Activity for my Android application and I came across this error as I was running it:

```java
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.tfc/com.tfc.ui.LearningScreen}: java.lang.RuntimeException: Your content must have a ListView whose id attribute is 'android.R.id.list'
     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2401)
     at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2417)
     at android.app.ActivityThread.access$2100(ActivityThread.java:116)
     at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1794)
     at android.os.Handler.dispatchMessage(Handler.java:99)
     at android.os.Looper.loop(Looper.java:123)
     at android.app.ActivityThread.main(ActivityThread.java:4203)
     at java.lang.reflect.Method.invokeNative(Native Method)
     at java.lang.reflect.Method.invoke(Method.java:521)
     at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:791)
     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:549)
     at dalvik.system.NativeStart.main(Native Method)
 Caused by: java.lang.RuntimeException: Your content must have a ListView whose id attribute is 'android.R.id.list'
     at android.app.ListActivity.onContentChanged(ListActivity.java:236)
     at com.android.internal.policy.impl.PhoneWindow.setContentView(PhoneWindow.java:316)
     at android.app.Activity.setContentView(Activity.java:1620)
     at com.tfc.ui.LearningScreen.onCreate(LearningScreen.java:29)
     at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1123)
     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2364)
     ... 11 more
```

However, there was nothing wrong with my layout XML:
```xml
<?xml version='1.0' encoding='utf-8'?>
<LinearLayout xmlns:android='http://schemas.android.com/apk/res/android'
    android:orientation='vertical'
    android:layout_width='fill_parent'
    android:layout_height='fill_parent'
    >

   	<TextView  
	   android:layout_width='fill_parent' 
	   android:layout_height='wrap_content' 
	   android:layout_weight='1.0'
	   />

	<Button
	    android:layout_width='fill_parent' 
	    android:layout_height='wrap_content' 
		android:text='Flip'
		android:id='@+id/btnFlip'
		/>
    
	<LinearLayout xmlns:android='http://schemas.android.com/apk/res/android'
	    android:orientation='horizontal'
	    android:layout_width='fill_parent'
	    android:layout_height='wrap_content'
	    >
		<Button
		    android:layout_width='wrap_content' 
		    android:layout_height='wrap_content' 
	    	android:layout_weight='0.5'
			android:text='Right'
			android:id='@+id/btnRight'
			/>
		<Button
		    android:layout_width='wrap_content' 
		    android:layout_height='wrap_content' 
	    	android:layout_weight='0.5'
			android:text='Wrong'
			android:id='@+id/btnWrong'
			/>
    </LinearLayout>

</LinearLayout>
```

The problem was I accidentally had my Activity extend from ListActivity instead of Activity:
```java
public class LearningScreen extends ListActivity {
     @Override public void onCreate(Bundle icicle) {
          super.onCreate(icicle);
          setContentView(R.layout.learning_screen);
     }
}
```

Here is a link to a problem with the exact same symptom, but turned out to be a completely different cause:
"Your content must have a listview whose id attribute is 'android.R.id.list' "
http://groups.google.com/group/android-developers/browse_thread/thread/d4d1d09dea087a71

Hope this helps you if you're stuck with a similar issue.

Cheers,
Joseph

