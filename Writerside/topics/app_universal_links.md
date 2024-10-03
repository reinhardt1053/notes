# App Links and Universal Links

During the development of My Azur, an application for Android and iOS, I implemented what are called **App Links** in 
Android and **Universal Links** in iOS.

The user receives a link via SMS/WhatsApp with the following format:

<code-block lang="console">
https://myazur.app/link/?action_id=1&status_id=2
</code-block>

Following a link with such format will take the user directly to the app instead of 
opening the default internet browser. The app opens immediately if it's installed
and I can handle the open link event by parsing the URL query args and bring the user
to a specific screen of the app. 

## Android App Links

For Android, it's important to clarify right away the difference between Deep Links, Web Links, and App Links. 
They might seem like different names for the same thing, but there are some important differences.

<img src="link-types-capabilities.svg" alt="Capabilities of deep links, web links, and Android App Links." width="200"/>

A Deep Link is the most general form, it is a URI that can have an arbitrary scheme chosen by the developer. 
For example, in my case, I could have chosen the scheme "myazur" and created a URI in the form:

<code-block lang="console">
myazur://app/link?action_id=1&status_id=2
</code-block>

A **Deep Link** is called a **Web Link** if it uses http or https as the scheme. So this is a web link:

<code-block lang="console">
https://myazur.app/link?action_id=1&status_id=2
</code-block>

A Web Link becomes an **App Link** if it uses as host a verified domain. We will see later that it is 
necessary to prove to Google that we own the domain we use as the host in our App Links.

The advantage of App Links is that they take the user directly into our application. On the other hand, 
in the case of a Deep Link or Web Link, Android displays a dialog where the user can choose which application
should open the link. Since the browser can open any Web Link, it will always be presented as an option, 
introducing an additional step that worsens the user experience.

<note>
Android App Links are available on Android 6.0 (API level 23) and higher.
</note>

### Create an Android App Link

Let's start creating an Android App Link by editing the ``AndroidManifest.xml``

<code-block lang="xml">
<![CDATA[
   <activity android:name="lu.sitasoftware.myazur.root.MainActivity"
             android:windowSoftInputMode="adjustResize"
             android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
        <intent-filter android:autoVerify="true">
            <data android:scheme="https" android:host="myazur.app"/>
            <category android:name="android.intent.category.DEFAULT"/>
            <category android:name="android.intent.category.BROWSABLE"/>
            <action android:name="android.intent.action.VIEW" />
        </intent-filter>
   </activity>
]]>
</code-block>

Please note the attribute ``android:exported="true"`` which indicates that the activity can be 
accessed by other apps on the device (Whatsapp, SMS, Browser, etc.)

The most important attribute above is ``android:autoVerify="true"``. This attribute allows your app to designate 
itself as the default handler of a given type of link. So when the user clicks on an Android App Link,
your app opens immediately if it's installed—the disambiguation dialog does not appear.

<img src="app-disambiguation_2x.png" alt="The disambiguation dialog" width="200"/>

### Verify your domain
On Android Studio you can navigate to the menu Tools->App Links Assistant

<img src="app_links_assitant.png" alt="Android Studio App Links Assistant" width="300"/>

select ``Open Digital Asset Links File Generator`` and from here you can generate the Digital Asset Link file 
that needs ot be hosted on your website under: 

```bash
https://myazur.app/.well-known/assetlinks.json
```

<img src="website_association.png" alt="Android Digital Asset Link file generation" width="400"/>


<code-block lang="json">
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "lu.sitasoftware.myazur",
        "sha256_cert_fingerprints":
        ["09:91:83:AC:C7:E5:A0:E2:BB:2D:C2:5B:5F:52:AB:D0:3F:1D:91:42:63:1E:78:18:2B:8E:60:5A:43:97:D0:C3"]
    }
}]
</code-block>

You can include multiple sha256 fingerprints for example your prod release an your debug keystore fingerprint.

<note>
If you're using Play App Signing for your app, then the certificate fingerprint produced by running keytool locally will usually not match the one on users' devices. You can verify whether you're using Play App Signing for your app in your Play Console developer account under Release > Setup > App signing; if you do, then you'll also find the correct Digital Asset Links JSON snippet for your app on the same page.
</note>

When publishing the JSON verification file, be sure of the following:

* The assetlinks.json file is served with content-type application/json.
* The assetlinks.json file must be accessible over an HTTPS connection, regardless of whether your app's intent filters declare HTTPS as the data scheme.
* The assetlinks.json file must be accessible without any redirects (no 301 or 302 redirects).
* If your app links support multiple host domains, then you must publish the assetlinks.json file on each domain.
* Do not publish your app with dev/test URLs in the manifest file that may not be accessible to the public (such as any that are accessible only with a VPN). A work-around in such cases is to configure build variants to generate a different manifest file for dev builds.

When android:autoVerify="true" is present in at least one of your app's intent filters, installing your app on a device that runs 
Android 6.0 (API level 23) or higher causes the system to automatically verify the hosts associated with the URLs 
in your app's intent filters. On Android 12 and higher, you can also invoke the verification process manually to test the verification logic.

### Test your Android App Links
Use the following command to check whether the system verified your app and set the correct link handling policies:

<code-block lang="bash">
adb shell am start -a android.intent.action.VIEW \
    -c android.intent.category.BROWSABLE \
    -d "https://myazur.app/link/?action_id=1&status_id=2"
</code-block>

