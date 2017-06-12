Conversant Advertiser SDK - Android
==================

The Conversant Advertiser SDK provides a simple way to gather information on how customers use your application, and allows Conversant to subsequently tailor relevant and meaningful messages to your users based on their interaction and purchase history. The following documentation provides a high-level overview of integrating the Conversant Advertiser SDK - your Conversant Client Integration Engineer will provide a customized integration document outlining events and data for your app and unique purpose.

## Getting Started

Getting started with the Conversant SDK is easy, just follow these simple steps:
* Grab the latest release of the SDK here:  [https://github.com/conversant/Advertiser-SDK-Android/releases](https://github.com/conversant/Advertiser-SDK-Android/releases)
* Update your App Manifest
* Initialize the SDK
* Use the SDK in your App

### Updating Your App Manifest

Add the following entries to your AndroidManifest.xml (if they are not already present).	

Permissions (As a child of <manifest> element, at the same level as the <application> element)
```
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
```

Required Service (As a child of the <application> element)
```
<service android:name="conversant.tagmanager.sdk.CNVRTagService" android:exported="false"/>
```

Required MetaData (As a child of the <application> element)
```
<meta-data android:name="conversantSiteId" android:value="CTM Site ID"/>
<meta-data android:name="com.google.android.gms.version" 
android:value="@integer/google_play_services_version"/>
```

###### Notes
* Be sure to replace ‘CTM Site ID’ with your the Conversant Site ID provided by your Client Integration Engineer
* If the inclusion of '@integer/google_play_services_version' gives you an error, the Google Play Services library may not be installed correctly in your development environment

### Initializing the SDK

Initialize the Tag Manager once, as soon as is possible in your application flow (either in your main activity or application activity, preferably Application class).  If you have an existing onCreate() method, you can simply cut and paste the following block of code
```
@Override 

public void onCreate() { 
  super.onCreate(); 
  CNVRTagManager.initialize(this); 
} 
```

In order ensure that the data being captured is accurate, the Conversant Advertiser SDK requires that Google Play Services be up-to-date. To ensure that, add the following code in your onResume() method(s)
```
@Override 

protected void onResume() { 
  super.onResume(); 

  CNVRTagManager.updateGooglePlayServices(new GooglePlayServicesTask.Listener() { 

    @Override 

    public void onGooglePlayServicesResult(int errorCode) { 
      if (GooglePlayServicesUtil.isUserRecoverableError(errorCode)) { 
        GooglePlayServicesUtil.getErrorDialog(errorCode, MainActivity.this, 0).show(); 
      } 
    } 
  }); 
} 
```

###### Note
The provided code will initiate the standard Google Play Services error dialog in order for your users to upgrade and/or install Google Play Services as needed. If you are already managing Google Play Service updates in your app, you may omit the following line from the block above
```
GooglePlayServicesUtil.getErrorDialog(errorCode, MainActivity.this, 0).show(); 
```

### Using the SDK in your application

The Conversant Advertiser SDK is event based. When a notable event occurs within your application (such as a customer viewing a product or making a purchase) the SDK allows you to 'sync' that event with Conversant.

To construct an event, use the CNVRTagSyncEvent.Builder
```
// Construct an event request 

final CNVRTagSyncEvent event = new CNVRTagSyncEvent.Builder( 
new String[] {"event-1", "event-2"}, "group1/group2") 
.withExtra("key1", "value1") // optional 
.withExtra("key2", "value2") // optional 
.withExtra("key3", "value3") // optional 
.withExtra("key4", "value4") // optional 
.withExtra("key5", "value5") 
.build();
```

###### Notes
Each event includes 3 types of data
* **Event Names** (Required) Used to identify specific events that occur in the application. Each event object may contain multiple event names.
* **Event Group** (Required) Used to hierarchically organize events within the application. Multiple groups may be delimited with forward slash '/'
* **Extra Data** (Optional) Used to pass any arbitrary data that may be necessary or useful to track along with the event(s) and Group(s)

It is recommended that you work with your business unit to determine which events should be tracked, and how best to organize the application in to groups. When in doubt, liberally adding tracking for events is always recommended, as the Conversant Advertiser SDK is designed to only communicate events that have tracking assigned to them by your Client Integration Engineer.

Once an event has been constructed, use the 'fireEvent' method to communicate that event to Conversant
```
// Obtain the sdk singleton 
CNVRTagManager sdk = CNVRTagManager.getSdk(); 

// Fire an event to Tag server 
sdk.fireEvent(event);
```

**Example Events**

A log-in event example which is sending the users MD5 email hash as a variable:
```
final CNVRTagSyncEvent event = new CNVRTagSyncEvent.Builder( 
new String[] {"login","login-success") 
.withExtra("email-hash", emailhash) 
.build(); 
```

A retail event example for view of current product:
```
final CNVRTagSyncEvent event = new CNVRTagSyncEvent.Builder( 
new String[] {"product-view"}, "category/subcategory/product-page") 
.withExtra("category", “Women”) 
.withExtra("subcategory", "Shoes") 
.withExtra("productSKU", "AB1234") 
.build(); 
```

### Capturing the Google INSTALL_REFERRER value

Google provides a [mechanism for capturing install referrer information](https://developers.google.com/analytics/devguides/collection/android/v4/campaigns) by setting a referrer parameter value in the Google Play Store’s app URL. This referrer value can be used for example, to measure campaign performance or help determine the vendor that drove the install. Google Play Store broadcasts this value, if available, to the app after an install completes. 

It is important to note, though, that due to a limitation with Google's implementation, the referrer value can only be passed on installs initiated from the app-based version of Google Play Store, NOT from the web-based version.

The Conversant Advertiser SDK can be configured to automatically capture this value, if available, and include it with your event data. Doing so requires two steps.

First, make the following addition to your AndroidManifest.xml
```
<receiver android:name="conversant.tagmanager.sdk.CNVRInstallReferrerReceiver"
android:exported="true">

<intent-filter> 
<action android:name="com.android.vending.INSTALL_REFERRER" />
</intent-filter>
</receiver>
```

Second, use the 'fireEventWithGoogleInstallReferrer' method in place of 'fireEvent' for the first event that occurs within your application. 
```
// Your application stores a preference value to determine if the install event has already been sent
//
int installRunOnce = Preferences.getInstallRunOnce();
if (installRunOnce == 0) {


  // Fire install event
  //
  CNVRTagSyncEvent event = new CNVRTagSyncEvent.Builder(new String[]{"install"}, "freecell/install")


  .withExtra("installTime", DATE_FORMATTER.format(new Date()))
  .build(); 


// Use the proper fire method that waits for the Play Store to broadcast the INSTALL_REFERRER    intent
//
CNVRTagManager.getSdk().fireEventWithGoogleInstallReferrer(event); 


// Record in your applications preferences that we have successfully fired
//
Preferences.setInstallRunOnce(); 
}
```

###### Notes
It is important that the 'fireEventWithGoogleInstallReferrer' function should only be used for firing an event that requires the Google INSTALL_REFERRER, and should only be used once. The method was specifically designed to delay communication with Conversant in order to wait on Google Play Services to return the referrer information (which can, in some instances, take several seconds). Once the referrer information is returned by the Google Play Services it is stored by the Conversant Advertiser SDK and made available to all subsequent events.

If you are already using Google Analytics in your application to capture the INSTALL_REFERRER, and want to pass it to the Conversant Advertiser SDK as well, you will need to write a custom receiver as outlined in [Google's documentation](https://developers.google.com/analytics/solutions/testing-play-campaigns#troubleshooting) and include the following block of code
```
CNVRInstallReferrerReceiver receiver = new CNVRInstallReferrerReceiver();
receiver.onReceive(context, intent);
``` 

## RELEASE NOTES

For the latest, up-to-date information on Android SDK features and bug fixes, visit our Android SDK Release Notes [https://github.com/conversant/Advertiser-SDK-Android/releases](https://github.com/conversant/Advertiser-SDK-Android/releases)

## LICENSE

The Conversant Advertiser SDK is licensed under the Apache License, Version 2.0 [http://www.apache.org/licenses/LICENSE-2.0.html](http://www.apache.org/licenses/LICENSE-2.0.html)
