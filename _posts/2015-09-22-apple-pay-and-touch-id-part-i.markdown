Apple Pay & Touch ID - Part I: Introduction to Keychain ACLs and LA Framework
=====================
This serie of articles was written few months ago, but never had the chance to publish it. Its content is based on a light research performed to the Apple Touch ID component, its integration with mobile applications, and Apple Pay.

The series will be initially composed by three articles:

* **Part I** - Introduction to Keychain ACLs and LA Framework.
* **Part II** - [Analyzing its implementation on mobile applications.](http://retarded.ninja/2015/apple-pay-and-touch-id-part-ii/)
* **Part III** - A quick look to Apple Pay.

------------------------------------------------------------------------------------
Introduction
=========
Touch ID is Apple’s biometric fingerprint authentication technology, introduced with the iPhone 5S. There are currently two ways to use Touch ID as an authentication mechanism in an iOS 8 application: *Keychain ACLs* and *Local Authentication framework*.

This article explains the changes introduced to the Keychain and Touch ID in iOS 8, reviews the security involved in its integration with third party applications, and the potential for security bypass in some implementations of the biometric verification

------------------------------------------------------------------------------------
 
Review of the Keychain & Secure Enclave
============================
The Keychain is a specialized database optimized for secrets, i.e., (*passwords*, *internet passwords*, *keys*, *certificates*, and *identities*). Applications have access to their own keychain items, but may not access keychain items of other applications. This operation is enforced by the *securityd* daemon. Inside the Keychain, each row is known as a Keychain Item, and an encrypted Keychain Attribute describes each item.

Keychain access control may include a combination of a user passcode and/or unique device secret. The Secure Enclave is a coprocessor that initially was first included within the A7 chip, and utilizes its own secure boot process completely independent from the application processor.

The Keychain does not decrypt Keychain items by itself, rather, the Secure Ecnalve 'unwraps' content keys, and the actual decryption process probably happens in securityd daemon. It also provides all cryptographic operations. Furthermore, the Secure Enclave is responsible for determining a successful match from fingerprint data analyzed from the TouchID sensor against a specific registered print.

![imagefoot](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_image1.png?raw=true)
<p align="center" style="font-size:11px">**Figure 1** - Application interacting with Keychain - *Credits to Xamarin for this graphic*</p>

------------------------------------------------------------------------------------

Touch ID
======
According to Wikipedia, Touch ID is a fingerprint recognition feature, designed by Apple and available on the iPhone 5S, the iPhone 6 and iPhone 6+. Its main purpose is to allow users to unlock their device, as well as make purchases in the various Apple media stores, and to authenticate Apple Pay online or in apps.

------------------------------------------------------------------------------------
Keychain ACLs and Touch ID
====================
Access Control List (*ACL*) is a Keychain item attribute introduced in iOS 8 that describes the information required for a particular operation to occur, e.g., (*a passcode*). What makes the ACLs interesting is the ability to set accesibility and authentication properties for a Keychain item to which they are applied:

* **Accessibility** - Represents when a particular item should be readable (*for example when a device is unlocked*). This is similar to what the '*kSecAttrAccessible*' attribute does.
* **Authentication** - Is a concept that is represented by a policy specifying what authentication requirements must be satisfied before the system will decrypt the Keychain item.
</br></br>
![imagespecial](https://raw.githubusercontent.com/0xroot/0xroot.github.io/master/_images/ap_image2.png)
<p align="center" style="font-size:11px">**Figure 2** - Diagram of this new attribute with the rest of keychain item - *Credits to Xamarin for this graphic*</p>

Furthermore, with iOS 8, there is a user policy, '*kSecAccessControlUserPresence*', enforced by the *securityd* daemon. This means that Touch ID can be used for authentication to authorize the Keychain item decryption operation. However, the device configuration can influence the policy evaluation, the table below this paragraph is for one particular policy (*kSecAccessControlUserPresence*):

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{font-family:Arial, sans-serif;font-size:14px;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg th{font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 5px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;}
.tg .tg-s6z2{text-align:center}
.tg .tg-6ead{background-color:#eaeaea;text-align:center}
</style>
<table class="tg">
  <tr>
    <th class="tg-6ead">Device Configuration</th>
    <th class="tg-6ead">Policy Evaluation</th>
    <th class="tg-6ead">Backup Mechanism</th>
  </tr>
  <tr>
    <td class="tg-s6z2">Device without passcode</td>
    <td class="tg-s6z2">No access</td>
    <td class="tg-s6z2">No backup</td>
  </tr>
  <tr>
    <td class="tg-s6z2">Device with passcode</td>
    <td class="tg-s6z2">Requires passcode</td>
    <td class="tg-s6z2">No backup</td>
  </tr>
  <tr>
    <td class="tg-s6z2">Device with Touch ID</td>
    <td class="tg-s6z2">Prefers Touch ID</td>
    <td class="tg-s6z2">Allows passcode</td>
  </tr>
</table>

* **Device without passcode set** - The request to retrieve the Keychain item will always fail.
* **Device with passcode set** - The user is presented with UI to enter the device passcode.
* **Device with Touch ID** -  The user is prompted to unlock the item using Touch ID. Passcode can be used as a fall-back.


</br>
Currently, there are two ways to use Touch ID as an authentication mechanism.

------------------------------------------------------------------------------------

Using Touch ID through Local Authentication Framework
=======================================
The simple implementation involves an API that returns the success or failure of a finger recognition. In case of error, an error code is returned, explaining the reason. However, there are some caveats with using this implementation:

* The application must be in foreground.
* The policy evaluation may fail, and the devleoper is responsible for handling all errors and ensuring an application has a fall-back mechanism.
</br></br>

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_image3.png?raw=true)
<p align="center" style="font-size:11px">**Figure 3** - Application communicating through Local Authentication*</p>

Any application can call Policy Evaluation inside Local Authentication, which will start an operation inside Secure Enclave. It’s possible in this way to start the Touch ID operation and get access granted to the application the credentials without being in touch with Secure Enclave.

A couple of examples where this framework can be used instead of Keychain are:

* Verifying that user is enrolled, in order to unlock specific features of application, or perform some kind of parental control because as a developer you may want to be sure that the user is present.

* As an extension to the current application’s authentication mechanisms, by using Touch ID as first factor instead of a password or PIN, (*though this is usually a task better suited to Keychain*). As a second factor authentication, which in this case, Local Authentication is the best choice for your application.

It’s important to highlight how the security is implemented in both mechanisms because in Keychain keys the SE is the trusted component, and OS trusts on it but SE doesn’t have to trust OS. But with Local Authentication the app trust the OS, but the OS is the one that enforces that bad-behaving apps don’t do what they are not suppose to do. It also means that there is no direct access to SE, and the application will only get the results.

Last but not least, with Local Authentication, there is no access to the registered fingerprints, nor fingerprint images, which are secured by the SE. The SE is the sole owner of such information and no other system component can get access to this data.

From the API perspective, it’s possible to use Local Authentication framework and use From the API perspective, it’s possible to use Local Authentication framework and use of  the Keychain. For this purpose, two new functions are available:

* **canEvaluatePolicy** - Checks if the policy can be ever evaluated on the device. It will determine if the device is Touch ID enabled and contains any fingerprints.
* **evaluatePolicy** - Starts the authentication operation and shows the necessary user interface, and the result that will be returned by this evaluatePolicy function will be a simple boolean value.

</br>
The only policy available at the moment to show the Touch ID prompt is ‘*LAPolicyDeviceOwnerAuthenticationWithBiometrics*'

------------------------------------------------------------------------------------

Integrating Local Authentication
==
First step to integrate Local Authentication within your application is to create an ‘LAContext’ object that will handle the task of asking the user to authenticate using Touch ID.

```
LAContext *myContext = [[LAContext alloc] init]; __autoreleasing NSError *authenticationError;

NSString *localizedReasonString = NSLocalizedString(@“Title Box", @“Reason why Touch ID is required");
```
The nil auto-released NSError object will be passed by reference to the ``-canEvaluatePolicy:error:`` method in the example below, in order to determine if an error occurred.

Next thing is to create the localized reason string, since all Touch ID operations require this string. This string provides the reason the application is requesting the fingerprint.

```
[myContext evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
                 localizedReason:myLocalizedReasonString
                           reply:^(BOOL success, NSError *error) {
                               if (success) {
                                   NSLog(@"User is authenticated successfully");
                               } else {
                                   switch (error.code) {
                                       case LAErrorAuthenticationFailed:
                                           NSLog(@"Authentication Failed");
                                           break;
                                       case LAErrorUserCancel:
                                           NSLog(@"User pressed Cancel button");
                                           break;
                                       case LAErrorUserFallback:
                                           NSLog(@"User pressed \"Enter Password\"");
                                           break;        
                                       default:
                                           NSLog(@"Touch ID is not configured");
                                           break;
                                   }
                                   NSLog(@"Authentication Fails");
                               }
                           }];
   } else {
       NSLog(@"Can not evaluate Touch ID");
   }
```

First thing is to check if the device can use Touch ID by triggering the ‘-canEvaluatePolicy:error:’ method. If the response is a ‘true’ statement, then a call to ‘-evaluatePolicy:localizedReason:reply:’ method is performed, passing as argument the localized reason string defined previously. If the response is a false, then it will be necessary to handle the response obtained in the error object, as below:

```
typedef NS_ENUM(NSInteger, LAError)
{
    /// Authentication was not successful, because user failed to provide valid credentials.
    LAErrorAuthenticationFailed = kLAErrorAuthenticationFailed,
     
    /// Authentication was canceled by user (e.g. tapped Cancel button).
    LAErrorUserCancel       	= kLAErrorUserCancel,
     
    /// Authentication was canceled, because the user tapped the fallback button (Enter Password).
    LAErrorUserFallback     	= kLAErrorUserFallback,
     
    /// Authentication was canceled by system (e.g. another application went to foreground).
    LAErrorSystemCancel     	= kLAErrorSystemCancel,
     
    /// Authentication could not start, because passcode is not set on the device.
    LAErrorPasscodeNotSet   	= kLAErrorPasscodeNotSet,
 
    /// Authentication could not start, because Touch ID is not available on the device.
    LAErrorTouchIDNotAvailable  = kLAErrorTouchIDNotAvailable,
     
    /// Authentication could not start, because Touch ID has no enrolled fingers.
    LAErrorTouchIDNotEnrolled   = kLAErrorTouchIDNotEnrolled,
} NS_ENUM_AVAILABLE(10_10, 8_0);
```

------------------------------------------------------------------------------------

Using Touch ID through Keychain keys
==========================
When creating items in the Keychain, multiple attributes need to be set. Such attributes specify two things:

* What the item represents.
* The access level that it should have.

</br>
A full list of such attributes can be seen [here](https://developer.apple.com/library/ios/documentation/Security/Reference/keychainservices/#//apple_ref/doc/constant_group/Attribute_Item_Keys), however, there is an special attribute that needs to be added to the Keychain if the use of Touch ID is required:

* **kSecAttrAccessControl** - This allows to the developer to define a fine-grained access control. The value associated with this is of type ‘*SecAccessControl*’, which represents the ACL and has to be created using the ‘*SecAccessControlCreateWithFlags*’ function. Currently, there is only one value ‘*UserPresence*’. To be able to create this object, you have to define which with authentication and which accessibility you require for this item.

</br>
The following snippet code shows how to store a secret:

```
SecAccessControlRef sacObject =
SecAccessControlCreateWithFlags(kCFAllocatorDefault, kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly, kSecAccessControlUserPresence, &error); !
						
NSData* secret = [@"top secret" dataWithEncoding:NSUTF8StringEncoding];
NSDictionary *query = @{			
	(id)kSecClass: (id)kSecClassGenericPassword,
	(id)kSecAttrService: @"myservice",
	(id)kSecAttrAccount: @"account name here",
	(id)kSecValueData: secret,
	(id)kSecAttrAccessControl: (id)sacObject};
						
OSStatus status =  SecItemAdd((CFDictionaryRef)query, nil);
```

For a more detailed example, please check this [resource](https://developer.apple.com/library/ios/samplecode/KeychainTouchID/Listings/KeychainTouchID_AAPLKeychainTestsViewController_m.html#//apple_ref/doc/uid/TP40014530-KeychainTouchID_AAPLKeychainTestsViewController_m-DontLinkElementID_8)

------------------------------------------------------------------------------------

Conclusions
=========
Since its presentation in WWDC 2014 (*you should watch [Session 711 - Keychain and Authentication with Touch ID](https://developer.apple.com/videos/wwdc2014/)*), where Apple specified possible uses for Local Authentication that involve Touch ID as a potential way to replace passwords or PINs, or even using it as second factor authentication, I found it very questionable.

After analyzing its implementation, I came to the conclusion that Local Authentication only seems useful when an unlocked device falls into the wrong hands, basically because it has been designed to ensure that when an application is launched, an additional login screen will be displayed to unlock features of application or to prevent access to the application’s secrets among other things.

However this information is accessible through USB, and because of that, sensitive information is usually encrypted and a key derived from the password introduced when the application is launched is required in order to decrypt the information.

Using Touch ID, no password is introduced and the SE only provides a true/false statement that will grant us direct access to the requested information. This can be translated into a lack of additional encryption as there is no password from which derive a key to encrypt/decrypt sensitive information. That’s why Local Authentication framework, as authentication mechanism being used to protect your applications’ assets, should be called into question. Additionally, the ease of achieving a bypass and getting access to application’s assets has been proven in this article.

It may prevent from promptly access to sensitive data on a first instance, but the Local Authentication provided in the sample code from Apple doesn’t provide security benefit from this new feature added on iOS 8. Instead of Local Authentication, it would be better to implement a mechanism based on Keychain Access Control Lists (ACL), through the ‘*kSecAttrAccessControl*’ attribute (*explained at the beginning of this article*), which specifies the Keychain entries that can be decrypted if the user has been re-authenticated using the device passcode or Touch ID.

References
========
* [Keychain and Authentication with Touch ID](http://devstreaming.apple.com/videos/wwdc/2014/711xx6j5wzufu78/711/711_keychain_and_authentication_with_touch_id.pdf?dl=1)
* [Andreas Kurtz - iOS 8 Touch ID Authentication API](http://www.andreas-kurtz.de/2014/10/ios-8-touch-id-local-authentication-caveats.html)
* [iOS Security Guide](https://www.apple.com/business/docs/iOS_Security_Guide.pdf)
* [Local Authentication Framework Reference](https://developer.apple.com/library/ios/documentation/LocalAuthentication/Reference/LocalAuthentication_Framework/)
