### Module 3 - Static Analysis

So far you've learned how to configure your computer and device with the necessary tools to decrypt iOS apps and copy them to your computer. In this module you'll learn how to analyze an iOS application by inspecting all its files, frameworks (dependencies) and lastly the application binary. It's called `static analysis` because you're not going to execute the binary, you'll be reviewing all the files contained in the `.ipa` archive. This is intended to be an interactive module, meaning I'll point you in the _hopefully_ right direction and you are going to find the issues yourself. But don't worry, if you feel lost or cannot find any issues, all the solutions are at the end of the module (along with explanations on _why they are considered issues_ and some _recommended solutions_).

After you decrypt an iOS application you'll end up with a `.ipa` file. This is an application archive, basically a zip archive. It includes the application binary, 3rd-party frameworks, configuration files, media files (like images and videos), UI elements (like [storyboards](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/DesigningwithStoryboards.html) and [nibs](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html)), custom fonts and any other file the developers embed within the application.

To illustrate the most common vulnerabilities in iOS applications I've created a very insecure application called `CoinZa`[^1], I wrote it in `Objective-C` (aka Objc) to make it simpler to explain some reversing steps. Applications written in `Swift` still prove a bit difficult for some tools, though I plan to add support to some modules for `Swift` applications in the future. For now you can download the Objc version form [here](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/tree/master/Files).

#### Extracting the application files
- Extracting the `.ipa` contents is as simple as changing its extension to `.zip` and unzipping it.
```bash
mv CoinZa.ipa CoinZa.zip
unzip CoinZa.zip
```
- After unzipping the contents you'll have a folder named `Payload` and inside you'll find the application bundle named `CoinZa.app`. _Note: On an application downloaded from the App Store you'll find 2 more files along with the `Payload` folder, a `iTunesArtwork` file which is the app icon and a `iTunesMetadata.plist` file that contains information like the developer's name and ID, the bundle identifier, copyrights, the name of the application, your email and the date you purchased it, among other information._
- Right-click (or Control ⌃ + Left-click) the `CoinZa.app` and select `Show Package Contents`.
- Finally, move all the files within the `.app` bundle to a new folder. This is to have an easier access to them, instead of right-clicking it and selecting `Show Package Contents` all the time.
```bash
mkdir CoinZaFiles
mv CoinZa.app/* CoinZaFiles/
```

#### Analyzing embedded files
Your end goal is to understand as much as possible what the developers are shipping with every application. It's a good idea to start by looking for _low-hanging fruit_ kind of issues. In iOS reversing these come as configuration files, example data files, database connection files or embedded private keys for SSH connections. Yes, as I've said before, I've seen all of these cases in real applications.
- The two most common configuration files I've encountered in iOS applications are `.plist` and `.json`. Start your research by reading through all the files you can find with these extensions and see if you can find some information that **should not be there**.
- A very important file is the `Info.plist` in the root directory of an iOS application. This file contains a lot of configuration data like if the application _enables_ weak TLS settings on some domains (search for the `NSAppTransportSecurity` key), or if the application accepts custom [`Scheme URLs`](https://developer.apple.com/documentation/uikit/core_app/allowing_apps_and_websites_to_link_to_your_content/defining_a_custom_url_scheme_for_your_app) (search for the `CFBundleURLTypes` key).

#### Analyzing 3rd party frameworks
Almost every single iOS application uses at least one 3rd party framework. As a security researcher this is very important because this increases the attack surface and more often than not the developers forget to update their dependencies and the bigger the list of dependencies, the harder it is to keep track of updated versions. This means that as long as an application "still works" there's no incentive to update these 3rd party frameworks. This leaves users with outdated, and potentially vulnerable, code on their devices. All the 3rd party framework within an iOS bundle live in a folder called `Frameworks`.
- Open the `Frameworks` folder, take a look at which frameworks `CoinZa` is using and pay attention to the frameworks' versions.
- **Tip 1:** The framework's version is disclosed in its `info.plist` file.
- **Tip 2:** Google those framework versions and search for known vulnerabilities.

#### Dumping the application classes
An essential part of the static analysis of any application is to gather information about what methods and classes are contained in the application. This step gives you very important information because, as many developers know, declaring very descriptive methods help the development of good products. Thus the names of some of the methods will give an insight of what the application features are. I'll show you how to use `class-dump-z` to dump the application's classes and methods. There's not going to be an exercise for this section, but you can then spend some time reading through the output and taking notes on interesting classes or methods.
- Dumping the classes is extremely easy with `class-dump-z`, navigate to the folder where you extracted the `CoinZa.app` files and run `class-dump-z` with the binary name as its first parameter and save the ouput on a `dump.txt` file:
    ```bash
    cd ~/Downloads/Payload/CoinZaFiles
    class-dump-z CoinZa > dump.txt
    ```
- If you open the `dump.txt` file, you have now all the classes, methods and some instance variable names of the application binary. As you can see there are some interesting classes like `Wallet`, `KeyPair`, `AddFundsViewController`, `CreateWalletViewController`. Even without installing the application we can see that this _probably_ is a cryptocurrency application.
- Finally, if you run `class-dump-z` with no parameters it will show you all the options it has for dumping classes.

#### Disassembling and decompiling the binary - Hopper
After the initial reconnaissance work, you've reached (IMO) the most exciting part of this module, understanding the actual behaviour of the application methods. After searching through the classes and methods in the `class-dump-z` output, you could see that this application is very small; but most of the applications are significantly bigger and have far more classes and methods. Because of this, it's important that you can prioritize your work and focus on the more interesting cases.
- To disassemble and decompile the binary open Hopper and drag-n-drop the CoinZa binary in Hopper's active window. _Note: You'll see that this binary is a [`FAT` binary](https://en.wikipedia.org/wiki/Fat_binary), which means that it contains code for more than one architecture. In this case it contains code for the `ARMv7` and `ARM64` architectures because this application targets a minimum version of iOS 10 and the minimum supported devices on iOS 10 are the iPhone 5, iPod Touch 6th Gen and iPad 4th Gen, which are `ARMv7` devices._

![Hopper](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-3/hopper1.png?raw=true)
- Hopper will ask you which architecture you want to disassemble. You can choose which ever you want though I'd recommend the `ARMv7` since it has a smaller and simpler instruction set but Hopper has some trouble disassembling some parts of the application on `ARMv7` and to explain better I'll be using the `ARM64` disassembled code.
- After selecting the architecture, it will ask you to set some options for the [Mach-o file](https://en.wikipedia.org/wiki/Mach-O). The defaults should suffice.
- Hopper will then begin to disassemble the binary. It shouldn't take too long since, again, this is a small app. But I've had some instances where it took about 45min to finish, and this was on a MacPro 6-core Xeon with 64GB RAM.
- Once Hopper finishes disassembling, select the `Procedures` tab on the left panel and you'll be able to see the list of method names that hopper was able to find.

![Hopper Procedures](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-3/hopper2.png?raw=true)
- If you select the `Str` tab (next to the `Procedures` one) as you probably guessed is the list of all the String-looking or printable characters within the binary. This is another favourite of mine since you can start searching for words like `secret`, `private`, `test` or `debug` and trace their usage. More often than not, developers leave test classes that provide a good insight. Sometimes there are even developer modes that we can enable to get extra functionality out of the application.
- To trace the usage of a string:
    - Search for a string, for example search for `isProVersion`.
    - On the main window, select the string and right-click on it.
    - On the menu select `References to aIsproversion`.
    - Hopper will take you to a the `cfstring` section, which is where the c-string literals are listed.
    - Select the `cfstring_isProVersion` and right-click on it and select `References to cfstring_isProVersion`.
    - Hopper will now show you a window with a list of methods. As you probably guessed, this is the list of methods that use the `isProVersion` string.
    - Select the first instance of `[AddFundsViewController viewDidAppear:]` and click Go.
    - On the main window you'll now see the assembly code of the `viewDidAppear` method of the `AddFundsViewController` class. If this is a bit confusing for you, Hopper has also a decompiler function.
    - In the middle of the top options bar select the `Pseudo-code Mode` tab (the one with the `if(b)` text).

    ![Hopper Pseudo-code Mode](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-3/hopper3.png?raw=true)
    - You'll be able to see that the string `isProVersion` is actually a key of an object stored in the [`NSUserDefaults`](https://developer.apple.com/documentation/foundation/nsuserdefaults) shared settings. I'll explain more on this in a bit.
- With the string search exercise you found evidence of features potentially guarded by a `ProVersion` state. You also saw in the previous exercise that you can load the `pseudo-code` of a method.
- To analyze a method with its `pseudo-code`: _Note: use the `ARM64` disassembly for this exercise._
    - Click on the `Procedures` tab and search for `WalletDetailViewController` and select the `didUpdateWalletBalance:` method.
    - Uncheck the `Remove potentially dead code` checkbox. Sometimes Hopper tries to optimize the decompiled code or just gets it wrong and the `pseudo-code` has some missing information. I usually uncheck this checkbox in case that happened.
    - I want to bring your attention to this section of the `pseudo-code`:
    ```c
    r2 = @"isProVersion";
    if (objc_msgSend(r0, @selector(boolForKey:)) != 0x0) {
            r8 = 0x1001f0000;
            r2 = @"isProVersion";
            r1 = @selector(stringWithFormat:);
            var_60 = d8 * 0x1001ad2e0;
            r2 = @"Since you are a pro user we added an extra 20%% and it's on us!\nYour balance will actually increase by US$%f.";
            r0 = objc_msgSend(@class(NSString), r1);
            r29 = r29;
    } else {
            r8 = 0x1001f0000;
            r2 = @"isProVersion";
            var_60 = d8;
            r2 = @"Funds purchased successfully, your balance will increase by US$ %f.";
            r0 = objc_msgSend(@class(NSString), @selector(stringWithFormat:));
            r29 = r29;
    }
    ```
    - What you can see is that enabling the `ProVersion` state will be beneficial to an attacker since it will grant them an extra 20% of _something_. You don't know what that _something_ is yet, but looks like you should take a note about this finding. 😉 Specially since it looks like the check is done on the client side.
    - I'll leave it to you to keep digging around and take notes of interesting methods and classes. Analyze as many classes and methods as you can because they will help on the next module.
- **Tip 1:** Ignore all classes with the `FIR` prefix, they are part of the Firebase framework and are outside of the scope of this analysis.
- **Tip 2:** If you are using the trial version of `Hopper` take into account that it will self-close every 30min.

#### Disassembling and decompiling the binary - Ghidra
On March 5th, 2019 the [NSA released](https://ghidra-sre.org/) a free and open source reversing tool called [`Ghidra`](https://en.wikipedia.org/wiki/Ghidra). `Ghidra` supports Windows, Linux and macOS. Even though it's a very new tool and I haven't been using it as long as Hopper, I wanted to add it to the course so that we all could learn from it. _Note: Like I said, I haven't used `Ghidra` much so please bear with me while I show you how to use it._
- You can launch Ghidra by running the `ghidraRun` bash script at the root of the `ghidra_9.0.1/` directory. _Note: `Ghidra` requires the Java JDK, if you don't have it on your machine you can download it from [here](https://www.oracle.com/technetwork/java/javase/downloads/jdk11-downloads-5066655.html)._
```bash
./ghidraRun
```
- If this is the first time you're running `Ghidra` you'll have to create a project. Click on `File` and then `New Project...` (or `⌘ + N`).
- Choose if you want a `Shared` or `Non-Shared` project. `Shared` project can be accessed by other users.
- Select a directory to save your project and give it a name.
- Drag-n-drop the `CoinZa` binary into `Ghidra`.
- `Ghidra` will display a dialog saying that the file contains nested files.This is the same as Hopper telling you that it was a `FAT` binary and you need to choose an architecture. Select the `Batch` option.
- You'll be presented with a window showing you the two architectures in the  binary. `AARCH64:LE:64:v8A` is `ARM64` and `ARM:LE:32:v8` is `ARMv7`. You can keep both selected, but since they are the same I'd suggest to just keep one selected.
- `Ghidra` will show a toast saying the file was _imported_. Super fast eh? Not so fast, _imported_ doesn't mean disassembled.
- Expand your project folder and the `CoinZa` folder and you'll see a file called either `ARM-32-cpu0x9` or `AARCH64-64-cpu0x0` depending on the file you previously selected.
- Drag-n-drop the `ARM-32-cpu0x9`/`AARCH64-64-cpu0x0` file on top of the `CodeBrowser` button (the one with the dragon icon).
- `Ghidra` will tell you that the file hasn't been analyzed and if you want to do it. Click `Yes`.
- Leave the default selected Analyzers selected and click `Analyze`. _Note: To be honest I haven't played around too much with `Ghidra` to know the different analyzers, that's why I suggested to leave the defaults._
- In my computer `Ghidra` took significantly longer than `Hopper`.
- On the `Symbol Tree` window (on the far left) select `Classes` and scroll down to `Wallet` and select the `WalletDetailViewController` class.
- Within the `WalletDetailViewController` functions search, again, for `didUpdateWalletBalance`.
- On the `Decompiler` window (on right side of the split windows) you'll see the decompiled code of the method. _Note: If you don't see the `Decompiler` window press `⌘ + E`._
```c
_objc_msgSend(&OBJC_CLASS__NSUserDefaults,"standardUserDefaults");
uVar2 = objc_retainAutoreleasedReturnValue();
iVar1 = objc_msgSend(uVar2,"boolForKey:",&cf_isProVersion);
if (iVar1 == 0) {
  _objc_msgSend(&OBJC_CLASS__NSString,"stringWithFormat:",
                &cf_Fundspurchasedsuccessfully,yourbalancewillincreasebyUS$%f.);
} else {
  _objc_msgSend(&OBJC_CLASS__NSString,"stringWithFormat:",
                &
                cf_Sinceyouareaprouserweaddedanextra20%%andit'sonus!YourbalancewillactuallyincreasebyUS$%f.
               );
}
```
- As you can see this looks very similar to what `Hopper` showed on its `pseudo-code` mode.
- A huge advantage is that `Ghidra` is free!

![Hopper Pseudo-code Mode](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-3/ghidra1.png?raw=true)

#### Conclusions
- A static analysis on an iOS application can take you as little or as long as you want. You can go as deep as you can. Specially because the same techniques used to inspect the main application binary can be used to reverse engineer the 3rd party frameworks' binaries. I personally spend many days, and sometimes even many weeks, performing static analysis on iOS applications. _Note: The first mobile bug I was ever rewarded for on [HackerOne](https://hackerone.com) was a weak encryption vulnerability, specifically an insecure encryption key generation, basically I was able to predict past and future encryption keys. This was possible because I spent a lot of time understanding their key generation algorithm and was finally able to understand its behaviour without even running the application, all via static analysis._
- Many developers don't realize that any file they embed in their application will be very easy to extract and analyze.
- As researchers is very good idea to check the 3rd party frameworks bundled with the application.
- Gather as much information as you can on this step because you'll use it in the dynamic analysis step.

#### Solutions
- `coinza-c7e97-firebase-adminsdk-ok3f8-df3457e3e8.json`: This is a configuration file for a [Firebase](https://firebase.google.com/) project and it **includes a private key** that the developers were using to connect with the Firebase backend services.
    - **The problem:** Sometimes developers don't realize that any file they embed with the app will be _easily_ extracted by anyone. In this case this configuration file gives attackers all the information they need to impersonate a legitimate access to the company's Firebase backend services and data.
    - **Recommended fix:** This configuration file is supposed to be stored in a backend server, not in a client application. What the developers need to do is to add an authentication flow in the client side and that lets the app authenticate with their own backend server and then have that backend server connect to Firebase.
    - _Note: I found a similar configuration file on a popular VPN application and when I reported it to the company they replied saying their Firebase project was not being used anymore and that's why they didn't feel the need to remove the configuration file, I did **not** use the private key to connect to their Firebase backend and verify if it was valid._

    ![Firebase private key suggestion](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-3/firebase-private-key-warning.png?raw=true)

- `SQLCIPHER_KEY`: A hard-coded secret key in the application's [info.plist](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Introduction/Introduction.html) file, this is the secret key used for the application's [SQLCipher](https://www.zetetic.net/sqlcipher/) to encrypt its database contents.
    - **The problem:** Even though the developers were thinking about protecting their users' generated content by encrypting the database, they did so by hard-coding a secret key that **all** installations of their app would use. This is a little bit harder to exploit but if an attacker is able to, for example, get a hold of users' _not encrypted_ iTunes backups, they will be able to decrypt the databases because **all** databases are using the same secret key.
    - **Recommended fix:** Generate a secret key per installation and store it in the [iOS keychain](https://developer.apple.com/documentation/security/keychain_services), this way attackers would have to compromise the devices of every victim and that's a considerable harder attack.

- `NSAllowsArbitraryLoads`: Disables [App Transport Security](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33) (aka ATS), allowing weak TLS configurations.
    - **The problem:** Apple introduced ATS to protect users from weak and vulnerable TLS configurations, disabling this feature means that this application puts the security and privacy of the end user at risk.
    - **Recommended fix:** Remove this key from the `Info.plist` and work with the backend engineers to update the servers' TLS configuration.

- `CFBundleURLTypes`: Having custom `Scheme URLs` is not an issue, but it means this app can be launched by other applications using the `coinza://` scheme URL and it probably takes some parameters with it. Just take a note about this feature, it will be helpful on the next module's exercises.

- `AFNetworking 2.5.1`: This version of the popular framework `AFNetworking` had a serious vulnerability that allowed attackers to perform [Man-in-the-Middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) (aka MitM) attacks on any application using any version <= 2.5.1 of this framework that was _not_ using SSL pinning.
    - **The problem:** Even though 3rd party frameworks help developers by saving time and resources, because they don't need to build that functionality themselves, they can introduce vulnerabilities and the developer has to wait until a patch is made available. In this case the authors of the frameworks acted quickly and released a patched version (though it was not a [complete fix](https://github.com/AFNetworking/AFNetworking/blob/2.5.2/AFNetworking/AFSecurityPolicy.m#L257-L265)) but all the applications had to be updated and re-submitted to the App Store and then users had to update their apps on their devices. This process takes a lot of time from patch to updating the end user. Every time you are about to add a 3rd party framework, think about this and consider if the framework is _really_ needed. Even if you don't introduce a vulnerability yourself, the moment you include a 3rd party framework on your application it becomes your responsibility.
    - **Recommended fix:** The fix is relatively simple, just update the framework. But the real recommendation is to use the minimum amount of 3rd party frameworks on your applications. I try to use the built-in frameworks that Apple provides before exploring a 3rd party option.

[^1] Since I **love** pizza, most of the things I do involve pizza. Hence the Pizza-based cryptocurrency `CoinZa`.
