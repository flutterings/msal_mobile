# MSAL Mobile Flutter Plugin

A Flutter plugin for authenticating with Azure AD on Android and iOS using the Microsoft Authentication library (MSAL).

The plugin wraps the Android and iOS MSAL libraries from Microsoft.  MSAL Mobile currently only supports a single account at a time.  The API allows for a user to be signed in or out, retrieve basic information about the signed in user and acquire tokens both interactively and silently.

## Project Requirements

* Flutter version greater than 1.12
* Android minimum SDK Version of 19
* iOS version of at least 11.0

# Installation

Add the following to your pubspec.yaml
```yaml
dependencies:
  msal_mobile: ^0.0.1
```

# Azure Setup

To authenticate with MSAL you will need to setup a new app registration in Azure Active Directory.  You can setup a new app registration by following these steps:

## Create the app registration
1. Login to the Azure portal and navigate to Azure Active Directory.
2. From the main menu in Azure Active Directory, select **App regisrations**.
3. In the App registrations blade, click **New registration**.
4. Give your app registration a name... anything works.
5. Choose an option for **Supported account types** depending on your target user.
6. Leave the selection empty for **Platform configuration (Optional)**. We will work on platform configuration in the following steps.
7. Click **Register** to create the app registration.

## Configure the app registration for Android
1. From the blade of the new app registration, select **Authentication**.
2. In the upper menu, click **Try out the new experience** if you are not already viewing the new experience.
3. Click the **Add a platform** button.
4. Select **Android** from the **Configure platforms** blade.
5. In the package name text box, enter your Android package name.
    * The package name/application id can be found in **android > app > build.gradle**.  It should be something like com.example.myapp
6. Next you will need to provide the SHA1 signature hash of your Android keystore.  Do the following to get this:
    * Open **android > app > build.gradle** in Android Studio.
    * Open the **Gradle** panel in the upper right-hand corner of the window.
    * In the Gradle panel navigate to **app > Tasks > android** and double click **signingReport**.
    * A signingReport panel will open in the bottom half of the screen. 
    * The different build variants and their corresponding keys are outputted.  The "SHA1" value shown for these variants is the hex representation of what needs to be configured in your app registration.
    * Copy the hex SHA1 signature and convert it to base64.  You can do that with an online tool such as this one: https://base64.guru/converter/encode/hex
7. Click the **Configure** button to finish setting up the Android app in Azure.  
8. The **Android** platform should now be listed as a platform in the app registration authentication blade.  Azure also generated a MSAL Configuration JSON object and redirect URI for the platform.  These will be needed to configure MSAL Mobile, so copy them and set aside for now.

## Configure the app registration for iOS
1. From the blade of the new app registration, select **Authentication**.
2. In the upper menu, click **Try out the new experience** if you are not already viewing the new experience.
3. Click the **Add a platform** button.
4. Select **iOS / macOS** from the **Configure platforms** blade.
5. In the bundle ID text box, enter the bundle identifier for your iOS app.
    * The bundle identifier can be found by opening **ios > Runner.xcodeproj > project.pbxproj** and searching for `PRODUCT_BUNDLE_IDENTIFIER`.  It should be something like com.example.myapp
6. Click the **Configure** button to finish setting up the iOS app in Azure.
7. The **iOS / macOS** platform should now be listed as a platform in the app registration authentication blade.  Azure also generated a redirect URI for the platform.  This will be needed to configure MSAL Mobile, so copy it and set it aside for now.

# Android Setup
1. Open **android > app > src > main** > **AndroidManifest.xml**.
2. Add BrowserTabActivity to your AndroidManifest.xml file.
    * `[your-base64-signature-hash]` should be replaced with the base64 signature hash you generated while setting up your Android app in the Azure app registration.
    * `[your-package-name]` should be replaced with the Android package name you entered when setting up the Android app in your Azure app registration.
```xml
<activity android:name="com.microsoft.identity.client.BrowserTabActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:host="[your-package-name]"
            android:path="/[your-base64-signature-hash]"
            android:scheme="msauth" />
    </intent-filter>
</activity>
```
3. Make sure your Android app has internet access by adding the following just above **\<application\>**
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```
4. Create a new JSON file in your Flutter project and populate it with the configuration that was generated for you by Azure when you added the Android platform to your app registration.  The file can be called anything and placed anywhere in your project.  This was added as **assets/auth_config.json** in the example project.  The default configuration provided by Azure should look something like this:
```json
{
  "client_id" : "00000000-0000-0000-0000-000000000000",
  "authorization_user_agent" : "DEFAULT",
  "redirect_uri" : "msauth://[your-package-name]/[my-url-encoded-base64-signature-hash]",
  "authorities" : [
    {
      "type": "AAD",
      "audience": {
       "type": "AzureADMultipleOrgs",
       "tenant_id": "organizations"
      }
    }
  ]
}
```
5. Add `"account_mode": "SINGLE"` to specify single client authentication.
```json
{
  "client_id" : "<app-registration-client-id>",
  "authorization_user_agent" : "DEFAULT",
  "redirect_uri" : "msauth://[your-package-name]/[url-encoded-package-signature-hash]",
  "account_mode": "SINGLE",
  "authorities" : [
    {
      "type": "AAD",
      "audience": {
       "type": "AzureADMyOrg",
       "tenant_id": "organizations"
      }
    }
  ]
}
```
6. Add the JSON asset file to the pubspec.yaml file.
```yaml
assets
    - assets/auth_config.json
```

# iOS Setup
1. Add `"ios_redirect_uri"` value to the auth_config.json file created during the Android setup.
```json
{
  "client_id" : "<app-registration-client-id>",
  "authorization_user_agent" : "DEFAULT",
  "redirect_uri" : "msauth://<your-package-name>/<url-encoded-package-signature-hash>",
  "ios_redirect_uri": "msauth.<your-ios-bundle-identifier>://auth",
  "account_mode": "SINGLE",
  "authorities" : [
    {
      "type": "AAD",
      "audience": {
       "type": "AzureADMyOrg",
       "tenant_id": "organizations"
      }
    }
  ],
  "logging": {
    "pii_enabled": false
  }
}
```
2. Set the iOS platform target version to a version >= 11.0 by opening the properties window of the Runner.xcodeproj file in Xcode.
3. In the **Signing and Capabilities** section, add the **Keychain Sharing** capability.
4. Add `com.microsoft.adalcache` as a keychain group.
5. Add the following to the AppDelegate.swift to handle redirection for the MSAL interactive flow:
```swift
override func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey : Any] = [:]) -> Bool {
    return MSALPublicClientApplication.handleMSALResponse(url, sourceApplication: options[.sourceApplication] as? String)
}
```
6. Add the following to the Info.plist file in **\<dict\>** by right clicking the file and opening as source. Replace `\[your-bundle-identifier\]` with the iOS bundle identifier identified during the iOS platform setup portion of the app registration setup.
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>msauth.[your-bundle-identifier]</string>
        </array>
    </dict>
</array>
```
7. Add the following to the Info.plist in **\<dict\>** to enable the use of Microsoft Authenticator if available:
```xml
<array>
    <string>msauthv2</string>
    <string>msauthv3</string>
</array>
```

# Usage

Import MSAL Mobile
```dart
import 'package:msal_mobile/msal_mobile.dart';
```

Initialize MSAL Mobile
```dart
class _MyAppState extends State<MyApp> {
    MsalMobile msal;

    @override
    void initState() {
        super.initState();
        MsalMobile.create('assets/auth_config.json', "https://login.microsoftonline.com/Organizations").then((client) {
            setState(() {
                msal = client;
            });
        });
    }
}
```

## APIs

Sign in
```dart
await msal.signIn(null, ["api://[app-registration-client-id]/[delegated-permission-name]"]).then((result) {
    print('access token (truncated): ${result.accessToken}');
})
```

Sign out
```dart
await msal.signOut()
```

Get token - attempt silent acquisition and fallback to interactive acquisition
```dart
await msal.acquireToken(["api://[app-registration-client-id]/[delegated-permission-name]"]], "https://login.microsoftonline.com/Organizations").then((result) {
    print('access token (truncated): ${result.accessToken}');
})
```

Get token interactive
```dart
await msal.acquireTokenInteractive(["api://[app-registration-client-id]/[delegated-permission-name]"]]).then((result) {
    print('access token (truncated): ${result.accessToken}');
})
```

Get token silent
```dart
await msal.acquireTokenSilent(["api://[app-registration-client-id]/[delegated-permission-name]"]], "https://login.microsoftonline.com/Organizations").then((result) {
    print('access token (truncated): ${result.accessToken}');
})
```

Get signed in account
```dart
await msal.getAccount().then((result) {
    if (result.currentAccount != null) {
        print('current account id: ${result.currentAccount.id}');
    }
})
```

## Error Handling

MSAL Mobile exposes a special exception class MsalMobileException to help feed back as much information as possible when exceptions occur.  It exposes an error code and a message as well as an inner exception with those same properties.  When possible, the error code and message correspond to the underlying Microsoft MSAL platform implementations.

It is recommended to check if the exception is an MsalMobileException in your error handling logic to get as many error details as possible.

```dart
await msal.getAccount().then((result) {
    if (result.currentAccount != null) {
        print('current account id: ${result.currentAccount.id}');
    }
}).catchError((exception) {
    if (exception is MsalMobileException) {
        print(exception.errorCode);
        print(exception.message);
        if (exception.innerException != null) {
            print(exception.innerException.errorCode);
            print(exception.innerException.message);
        }
    } else {
        print('exception occurred');
    }
});
```