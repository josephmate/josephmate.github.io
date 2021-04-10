---
layout: post
title: Creating a Continuously Appendable List - Result
date: 2010-09-10 04:23
author: matejoseph
comments: true
categories: [Android]
---
Result ( The left is mine. ):

![Continuously Appendable List](/assets/20100910_myresult.png)
![Continuously Appendable List](/assets/20100910_contacts_ui.png)

Big Picture of What's Going On:
I have a TableLayout that I add rows to when the "+" button is clicked. Each row has a "-" button that knows which table and row it belongs to. That way it has enough information to remove itself from the table.

Code:
```java
public class AppendToAList extends Activity {
    
	TableLayout list;
	int rowsSoFar = 0;
	
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        
        Button addButton = (Button) findViewById( R.id.add );
        // Every time the '+' button is clicked,
        // add a new row to the table.
        addButton.setOnClickListener( new OnClickListener() {
			public void onClick(View view) { addButton(); }
		});
        
        list = (TableLayout) findViewById( R.id.list );
        
        // Start with one row.
        addButton();
    }
    
    /***
     * Gets all the information necessary to delete itself from the constructor.
     * Deletes itself when the button is pressed.
     */
    private static class RowRemover implements OnClickListener {
    	private TableLayout list;
    	private TableRow rowToBeRemoved;
    	
    	/***
    	 * @param list	The list that the button belongs to
    	 * @param row	The row that the button belongs to
    	 */
    	public RowRemover( TableLayout list, TableRow row ) {
    		this.list = list;
    		this.rowToBeRemoved = row;
    	}
    	
    	public void onClick( View view ) {
    		list.removeView( rowToBeRemoved );
    	}
    }
    
    public void addButton() {
    	TableRow newRow = new TableRow( list.getContext() );
    	Button actionButton = new Button( newRow.getContext() );
    	actionButton.setText( 'Action: ' + ++rowsSoFar );
    	Button removeSelfButton = new Button( newRow.getContext() );
    	removeSelfButton.setText( '-' );
    	// pass on all the information necessary for deletion
    	removeSelfButton.setOnClickListener( new RowRemover( list, newRow ));
    	newRow.addView( actionButton );
    	newRow.addView( removeSelfButton );
    	list.addView( newRow );
    }
}
```

Layout:
```xml
<?xml version='1.0' encoding='utf-8'?>
<ScrollView xmlns:android='http://schemas.android.com/apk/res/android'
    android:layout_width='fill_parent'
    android:layout_height='fill_parent' >
	<LinearLayout xmlns:android='http://schemas.android.com/apk/res/android'
	    android:orientation='vertical'
	    android:layout_width='fill_parent'
	    android:layout_height='fill_parent' >
		<TableLayout xmlns:android='http://schemas.android.com/apk/res/android'
		    android:orientation='vertical'
		    android:layout_width='fill_parent'
		    android:layout_height='fill_parent'
		    android:stretchColumns='0' >
		    <TableRow
		    	android:layout_width='fill_parent' >
		    	<TextView
		    		android:text='Heading' />
		    	<Button
		    		android:id='@+id/add'
		    		android:text='+' />
		    </TableRow>
		</TableLayout>
		<TableLayout xmlns:android='http://schemas.android.com/apk/res/android'
			android:id='@+id/list'
		    android:orientation='vertical'
		    android:layout_width='fill_parent'
		    android:layout_height='fill_parent' 
    		android:stretchColumns='0' />

	</LinearLayout>
</ScrollView>
```

I'm off to apply this to my android application!
Cheers,
Joseph

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="26"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>