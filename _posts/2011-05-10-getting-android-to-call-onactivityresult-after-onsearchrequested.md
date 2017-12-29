---
layout: post
title: Getting Android to call onActivityResult() after onSearchRequested()
tags:
- Android
redirect_from:
  - /2011/05/getting-android-to-call-onactivityresult-after-onsearchrequested/
---
Because everything is handled through the `SearchManager` in Android, `onActivityResult()` is not typically called after `onSearchRequested()`.

What I was looking for was a clean way to get search results back to the activity that originally called `onSearchRequested()`.

I first tried the most obvious approach of calling `setResult()` and `finish()` from the search result (searchable) activity, but because the activity isn’t launched using `startActivityForResult()` it doesn’t work. (Complicating things further, the `SearchManager` is the one launching the activity so it really doesn’t even work in theory).

**Solution:** I had to make the original calling activity also the searchable activity, so my entry in the manifest looks like this:

```java
<activity android:name=".MyBaseActivity"
          android:launchMode="singleTop">
 
   <!-- MyBaseActivity is also the searchable activity -->
   <intent-filter>
      <action android:name="android.intent.action.SEARCH" />
   </intent-filter>
   <meta-data android:name="android.app.searchable"
              android:resource="@xml/searchable"/>
 
   <!-- enable the base activity to send searches to itself -->
   <meta-data android:name="android.app.default_searchable"
              android:value=".MyBaseActivity" />
</activity>
```

Then, as described in the [android documentation](http://developer.android.com/guide/topics/search/search-dialog.html#LifeCycle), I created a `handleActivity` method in which I manually start the search activity _**for result**_:

```java
private void handleIntent(Intent intent) {
   // Get the intent, verify the action and get the query
   if (Intent.ACTION_SEARCH.equals(intent.getAction())) {
     String query = intent.getStringExtra(SearchManager.QUERY);
     // manually launch the real search activity
     final Intent searchIntent = new Intent(getApplicationContext(),
           MySearchActivity.class);
     // add query to the Intent Extras
     searchIntent.putExtra(SearchManager.QUERY, query);
     startActivityForResult(searchIntent, ACTIVITY_REQUEST_CODE);
   }
```

From there it is just a matter of having the search activity perform its search as normal and, when a result is clicked, call `setResult` and `finish`. The original activity will then get the result as expected in the `MyBaseActivity.onActivityResult` method.
