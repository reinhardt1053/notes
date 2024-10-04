# Deep linking in Android and iOS

During the development of My Azur, an application for Android and iOS, I implemented deep linking using 
 **App Links** in Android and **Universal Links** in iOS.

The user receives a link via SMS/WhatsApp with the following format:

<code-block lang="console">
https://myazur.app/link/?action_id=1&status_id=2
</code-block>

Clicking a link in this format will take the user directly to the app, bypassing the default browser. 
If the app is installed, it opens immediately, allowing me to handle the link by parsing the URL's 
query parameters and navigating the user to a specific screen within the app.

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

A Web Link becomes an **App Link** if it uses as host a verified domain. You will see later that it is 
necessary to prove to Google that you own the domain you use as the host in our App Links (auto-verification process).

### Why should I use an App Link instead of a Web Link?

When the user clicks on an Android App Link, your app opens immediately if it's installed and the disambiguation dialog does not appear.
for simple web links Android will always show the disambiguation dialog. Since the browser can open any Web Link, 
it will always be presented as an option, introducing an additional step that worsens the user experience.
So if you own the domain you should always use app links instead of web links. 

<img src="app-disambiguation_2x.png" alt="The disambiguation dialog" width="200"/>

<tip>
Android App Links are available on Android 6.0 (API level 23) and higher.
</tip>

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
itself as the default handler of a given type of link. 

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
        ["09:91:83:AC:C7:E5:A0:E2:BB:2D:C2:5B:5F:52:AB:D0:3F:..."]
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

<code-block lang="bash">
adb shell pm get-app-links --user cur lu.sitasoftware.myazur
</code-block>

Run the following command to query the verification state of the domain:

<code-block lang="bash">
adb shell pm get-app-links --user cur lu.sitasoftware.myazur
</code-block>

<code-block lang="bash">
lu.sitasoftware.myazur:
ID: 674bf8f9-ec26-4090-8cee-d4cefb9a7fe4
Signatures: [09:91:83:AC:C7:E5:A0:E2:BB:2D:C2:5B:5F:52:AB:...]
Domain verification state:
    myazur.app: verified
    User 0:
        Verification link handling allowed: true
        Selection state:
            Disabled:
                myazur.app
</code-block>

Domains that pass the verification process will have a status of ``verified``

If the domain has any other status, it means the verification could not be completed.
Specifically, if the status is ``none`` it suggests that the verification process may still be ongoing and 
hasn't finished yet, you may need to wait about 20 seconds to get a verification result.

<warning>
Please note that the domain myazur.app is listed as Disabled. Odd enough that's the expected state for a verified domain. 
Android lists a domain as `Enabled` only when the domain is not verified (auto-verification failed) and the user has manually 
allowed the app to open links. I find it misleading the way my domain is reported as Disabled but apparently the selection 
state makes sense only for domains for which the auto-verification has failed. 
</warning>


### Incorrect assetlinks.json Signature

Double-check that your signature in assetlinks.json is accurate and matches the one used to sign your app. 

Common errors include:

- Signing the app with a debug certificate but having only the release signature in assetlinks.json.
- Using a lowercase signature, when it should be uppercase.
- If you're using Play App Signing, ensure you're using the correct signature that Google uses to sign your releases. 
Follow the instructions for declaring website associations for detailed steps, including an example JSON snippet.


## References
- <a href="https://developer.android.com/training/app-links">Official Android Documentation: Handling Android App Links</a>
- <a href="https://zarah.dev/2022/02/08/android12-deeplinks.html">Zarah Dominguez - Debugging App Links in Android 12</a>
