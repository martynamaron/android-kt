## AndroidManifest.xml

`AndroidManifest.xml` provides information about your app to the Android system that tell it how to run the app, and what Activities are included.

All `Activity` classes need to be defined in the manifest.

```
<activity android:name=".MainActivity">
   <intent-filter>
       <action android:name="android.intent.action.MAIN"/>

       <category android:name="android.intent.category.LAUNCHER"/>
   </intent-filter>
</activity>
```

The `<action>` and `<category>` elements inside `<intent-filter>` tell the system where to start the app when the user clicks the launcher icon.

The `AndroidManifest.xml` also defines permissions the app needs to run.