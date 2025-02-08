---
layout: post
title: Creating a Continuously Appendable List - Tinkering
date: 2010-09-10 00:54
author: matejoseph
comments: true
categories: [Android]
---
My goal is to create a list where the user can continuously append buttons. This is similar to the functionality we see when editing a contact on the android.

![Continuously Appendable List](/assets/20100910_contacts_ui.png)

First I started tinkering with a LinearLayout and dynamically added some buttons to it.

Here's the code I came up with:
```java
public class AppendToAList extends Activity {
    
	LinearLayout list;
	
	/** Called when the activity is first created. */
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        
        list = (LinearLayout) findViewById( R.id.list );
        
        Button b1 = new Button( list.getContext() );
        b1.setText( 'This is the first textbox' );
        Button b2 = new Button( list.getContext() );
        b2.setText( 'This is the second textbox' );
        list.addView(b1);
        list.addView(b2);
    }
}
```
And this is the layout:
```xml
<?xml version='1.0' encoding='utf-8'?>
<LinearLayout xmlns:android='http://schemas.android.com/apk/res/android'
	android:id='@+id/list'
    android:orientation='vertical'
    android:layout_width='fill_parent'
    android:layout_height='fill_parent'
    >
</LinearLayout>
```

The result looked like this:

![First try](/assets/20100910_first_try.png)

Sweet, it worked. Now I am going to add an OnclickListener to one of the buttons to add more buttons:
```java
public class AppendToAList extends Activity {
    
	LinearLayout list;
	
	/** Called when the activity is first created. */
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        
        list = (LinearLayout) findViewById( R.id.list );
        
        Button addButtons = new Button( list.getContext() );
        addButtons.setText( 'Add a button.' );
        addButtons.setOnClickListener( new OnClickListener() {
			
        	int buttonsSoFar = 0;
        	
			public void onClick(View arg0) {
				Button newButton = new Button( list.getContext() );
				newButton.setText( 'Button: ' + ++buttonsSoFar );
				list.addView( newButton );
			}
		});
        list.addView(addButtons);
    }
}
```

The result:

![Second Try](/assets/20100910_second_try.png)

There's a a slight problem. You cannot see the new buttons if you try to add more than eight. Here's what is looks like when I added 12 buttons:

![Problem with the Second Try](/assets/20100910_second_try_problem.png)

However, we can just place the LinearLayout inside a ScrollView, to make it scrollable:
```xml
<?xml version='1.0' encoding='utf-8'?>
<ScrollView xmlns:android='http://schemas.android.com/apk/res/android'
    android:layout_width='fill_parent'
    android:layout_height='fill_parent'
	>
	<LinearLayout xmlns:android='http://schemas.android.com/apk/res/android'
		android:id='@+id/list'
	    android:orientation='vertical'
	    android:layout_width='fill_parent'
	    android:layout_height='fill_parent'
	    >
	</LinearLayout>
</ScrollView>
```

The result:

![Third Try](/assets/20100910_third_try.png)

Now I believe that I have all the tools necessary to mimic the google contact's continuously appendable list.

