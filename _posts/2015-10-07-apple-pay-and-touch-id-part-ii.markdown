Apple Pay & Touch ID - Part II: Local Authentication caveats
====================
This is the second article of this serie about Apple Pay & Touch ID.  For those interested in the first part, please refer to:  [Introduction to Keychain ACLs and LA framework](http://retarded.ninja/2015/apple-pay-and-touch-id-part-i/).

For this second article, we will take a real application which implements the Touch ID authentication via *Local Authentication* framework and after a simple reverse engineering process we will demonstrate how this mechanism can be defeated with a static and dynamic approach.

The series will be initially composed by three articles:

* **Part I** - [Introduction to Keychain ACLs and LA Framework.](http://retarded.ninja/2015/apple-pay-and-touch-id-part-i/)
* **Part II** - Analyzing its implementation on mobile applications.
* **Part III** - A quick look to Apple Pay.

-------------------------------
NOP all the things: A static approach
====================
First thing that can be done in order to determine if the application is using Local Authentication framework is to check the shared libraries that have been using it in the compilation process with *Otool* utility.

```
➜ NS_LocalAuthentication.app  otool -vL NS_LocalAuthentication | grep -Hsin "LocalAuthentication" --color=AUTO
(standard input):2: 	/System/Library/Frameworks/LocalAuthentication.framework/LocalAuthentication (compatibility version 1.0.0, current version 1.0.0)
```

Next step is to determine which approach is more convenient to achieve our goals. In this article, we are going to modify the application’s binary and tamper with its source code. This can be done in multiple ways, via *Cycript* or *GDB* at runtime, or through static analysis by using *IDA Pro* / *Hopper* / *Radare2* or any other disassembler utility. 

This being said, it is necessary to spot the function where the implementation for Local Authentication is contained. A quick tip to go through this is to perform a search for strings like '*objc_cls_ref_LAContext*', '*canEvaluatePolicy*', or '*evaluatePolicy*'.

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing1.png?raw=true)
<p align="center" style="font-size:11px">**Figure 1** - *evaluatePolicy* matches within the binary*</p>

The only thing that we need to remember is that both methods have a different use:

* **canEvaluatePolicy:error:** - Checks if the device is Touch ID available, otherwise it returns an error.

* **evaluatePolicy:localizedreason:reply:** - Initializes the authentication operation, providing the reason for requesting authentication through the '*localizedReason*' parameter and the reply block that is executed when policy evaluation finishes.

Because of this, we are going to focus our efforts in the ‘evaluatePolicy’ one. Furthermore, the developer must not call ‘canEvaluatePolicy:error:’ in the same block, otherwise it could lead to deadlock. 

A quick look into its cross references reveals the following matches:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing2.png?raw=true)
<p align="center" style="font-size:11px">**Figure 2** - Cross-references for *canEvaluatePolicy:error*</p>

It is necessary to take into consideration that the representation shown below may differ from one application to another one, but the concept is the same and should be applicable to any app using Local Authentication framework mechanisms.  

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing3.png?raw=true)
![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing3_2.png?raw=true)
<p align="center" style="font-size:11px">**Figure 3** - Assembly instructions for *sub_36244c* method.</p>

The instructions at offset **0x00362494** (*# add r9, pc*) and **0x0036249c** (*# ldr.w r1, [r9]*), make reference to the '*evaluatePolicy*' method, but there is no track of the block that makes reference to the reply block that is executed when policy finishes. Accordingly, this is not exactly the method that we need to modify. 

According to Hopper’s representation, the instruction that triggers the ‘evaluatePolicy’ method is:

```
 r0 = [r2 evaluatePolicy:0x1 localizedReason:r3 reply:STK32];
```
As you can see, the implementation for the reply block appears as '*STK32*' when the block structure looks similar to the one below:

```
[context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
    	localizedReason:@"To access your photos"
						
              	reply:^(BOOL success, NSError *error) {
                	if (success) {			
                   	    NSLog(@"Authenticated using Touch ID.");		
                   	} else {
                       	    [self reportError];		
                             }
               }		
}];
```
This means that our reverse engineering work is not done yet -- a new search must be performed to determine the block responsible for returning the evaluation value. Once again, there are multiple paths to reach our goals. The easiest ones are:

* Follow the references to other functions that are presented in the disassembly output for the '*sub_36244c*' function.

* Search for the string '*success*' and take a look at its cross references, as it is commonly used in the if-statement reply block.


With the first procedure, the address we are looking for is located in the instruction at offset **0x362468** (*# ldr r2, [r0 #0x14]*) and **0x36246a** (*# add r9, pc*). This can be translated into a reference to offset '*0x3624b1*' (*sub_3624b0 + 0x1*). A quick look into its source code reveals the structure we have been looking for:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing4.png?raw=true)
<p align="center" style="font-size:11px">**Figure 4** - Diagram for different memory jumps.</p>

Let's take a look into the pseudo-code representation provided by Hopper for this method:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing5.png?raw=true)
<p align="center" style="font-size:11px">**Figure 5** - Pseudo-code representation for *sub_3624b0*</p>

Far from being exact, it helps us to provide a better understanding on what is going on under the hood. In this case, the reply code that will be returned is being assigned to '*r8*' register, which will be used to retrieve its respective Boolean representation and assigns it as a value for the key '*success*'.

The way to proceed is to modify the '*r8*' or '*arg1*' register to *0x1*, assuring that the policy will always return '*true*' independently of the evaluation performed.

The other alternative that can be used to identify the policy block is to search for the string '*success*', although this may not be effective in all cases and a significant amount of time is required to spot the structure code that fits with the implementation we’re looking for. For this particular case, the references to the string ‘success’ obtained were:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing6.png?raw=true)
<p align="center" style="font-size:11px">**Figure 6** - Cross-references for the string *success*</p>

The methods that look suspicious for our purposes are the last three; '*sub_362330*', '*sub_3624b0*', and '*sub_362678*'. However, only one will contain the right code structure, which in this case is the second one, '*sub_3624b0*'.

As we stated before, although the Touch ID use has to be invoked through the ‘evaluatePolicy’ method, the method where it is implemented may differ by application. Because of that, it implies that the way to bypass it will also differ. Perhaps the best wat to see this is through another example:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing7.png?raw=true)
<p align="center" style="font-size:11px">**Figure 7** - Block that contains the check for the fingerprint</p>

In the case above, the reply block is referenced by the instruction located at offset '**0x0028e82c**' (*# add r1, pc*), while the call to '*evaluatePolicy*' method is located at offset '**0x028e85e**' (*# add r1, pc*). The block that evaluates the policy can be seen at '[*StartViewController showTouchIDPromptIfApplicable*]*_block_invoke*'. Its code structure is the following:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing8.png?raw=true)
<p align="center" style="font-size:11px">**Figure 8** - Diagram representation for the previous code block</p>

An approximate representation provided by Hopper can be seen here:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing9.png?raw=true)
<p align="center" style="font-size:11px">**Figure 9** - Pseudo-code representation</p>

As you may have noticed, the evaluation is checked in the if-statement. If the condition is met, then the Touch ID component will allow the user to initializes session into the application. Otherwise it will prompt the fallback mechanism, which will be the access through passcode (*the one established within the application, not by the operating system*).

The way to circumvent this is even simpler than the previous example, as we don’t need to add additional instructions and it can be done by adding a nop instruction in the '*beq.w*' instruction that performs the jump.

Basically, we need to pass from this state:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing10.png?raw=true)
<p align="center" style="font-size:11px">**Figure 10** - Authentication branch statement.</p>

To this other:

![title](https://github.com/0xroot/0xroot.github.io/blob/master/_images/ap_reversing11.png?raw=true)
<p align="center" style="font-size:11px">**Figure 11** - Authentication mechanism bypassed.</p>

By doing this, we are ensuring with 100% probability that the policy will always be evaluated as true, and therefore, the Touch ID authentication mechanism implemented within the application will be avoided with anything that makes contact with the Touch ID sensor. 

------------------------------
Cycript: Overriding evaluatePolicy on runtime
==========================
It is necessary to understand how everything works under the hood, but in terms of productivity and efforts, the static approach describe above this section is time consuming.

Another approach totally different is to use Cycript, for those who are not familiar with such utility, Cycript is a Javascript interpreter which also understands Objective-C syntax, meaning that it is possible to write either Objective-C or Javascript or even both from a terminal.

It is used to hook into a running process and provide us the ability to modify the application's behavior at runtime and override its methods, variables, etc.

For the sake of this article, it will be assumed that the user has Cydia installed and running on its device and knows how it works. Our goal is to override the implementation for the method ``evaluatePolicy:localizedReason:reply:`` and independently of the reason evaluated, force it to return ``YES`` and grant access to the user.

Assuming the user is already attached to the process with:

```
iPhone-6:~ root# cycript -p PID_OR_APP_NAME
```

The code block that we need to provide to Cycript is described below this line:

```
cy# @import com.saurik.substrate.MS
cy# var oldm = {};
cy# MS.hookMessage(LAContext, @selector(evaluatePolicy:localizedReason:reply:), function(self, reason, block) { block(YES, nil); }, oldm);
cy#
```

Right after that, -- *and once the user has completed the authentication with a fingerprint or a passcode previously* -- if the application is closed with the ``home`` button, and re-launched right again, the auth prompt wil be visible for a fraction of second but will dismiss itself.

Another alternative out of the scope for this article is to hook ``LAContext`` method using ``CydiaSubstrate`` via ``dylib``.

------------------------------
References
========
* [Cycript tricks](http://iphonedevwiki.net/index.php/Cycript_Tricks)

------------------------------
Thanks
=====
I would like to take a moment to thank some friends who helped me during my research:

* Andrey Belenko [Twitter](https://twitter.com/abelenko)
* Marco Grassi [Twitter](https://twitter.com/marcograss)
* Pancake [Twitter](https://twitter.com/trufae)
* Revskills [Twitter](https://twitter.com/asesyochos)
* Pof [Twitter](https://twitter.com/pof)



