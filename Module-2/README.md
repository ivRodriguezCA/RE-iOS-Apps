### Module 2 - Decrypting iOS Applications

After setting up your device (and computer), you're ready to start downloading applications from the App Store and decrypting them. As a security researcher this is the first step you'll have to perform to start any analysis. This is because iOS encrypts every application downloaded from the App Store using their DRM technology called [FairPlay](https://en.wikipedia.org/wiki/FairPlay).

Since a system that can perform operations on encrypted binaries doesn't exist yet[^1], iOS has to decrypt the application first in order for the OS to run the actual executable and Clutch leverages that in order to "decrypt" the application. In a few words, what Clutch is doing is "asking" the OS to load the application to memory to run it, then dumping the _decrypted_ version of the application from memory and writing it to disk.

_Note: In case you missed it, you'll only need a jailbroken device for this module and the last part of the [Module 4](../Module-4/README.md). For the next modules I'll provide the decrypted version of the iOS application._

#### If your device's iOS version < 11.0
- Download any application from the App Store.
- Run `iTunnel` to forward your SSH traffic via USB:
    ```bash
    itnl --lport 2222 --iport 22
    ```
- SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
- Use Clutch to list the installed applications on your device:
    ```bash
    Clutch -i
    ```
- Use Clutch to decrypt the application you just downloaded by passing its index:
    ```bash
    Clutch -d 1
    ```
- Wait for Clutch to finish, then in the output you'll see Clutch saved the decrypted application in `/private/var/mobile/Documents/Dumped`.
- On your computer, on a different terminal window, copy the dumped application to your machine:
    ```bash
    scp -P 2222 root@localhost:/private/var/mobile/Documents/Dumped/<app-name>.ipa ~/Desktop/
    ```
- Now you have a decrypted version of the app.

#### If your device's iOS version >= 11.0
- Download any application from the App Store.
- Run `iTunnel` to forward your SSH traffic via USB:
    ```bash
    itnl --lport 2222 --iport 22
    ```
- SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
- If you're using a LiberiOS jailbreak, enable the binpack:
    ```bash
    export PATH=$PATH:/jb/usr/bin:/jb/bin:/jb/sbin:/jb/usr/sbin:/jb/usr/local/bin:/sbin:/usr/sbin:/usr/local/bin:
    ```
- Change directories to `/jb/bfinject`
    ```bash
    cd /jb/bfinject
    ```
- Open the application on your phone by tapping on its icon in [Springboard](https://en.wikipedia.org/wiki/SpringBoard). _Note: This step is important, in order for `bfinject` to be able to decrypt applications, they have to be running in the foreground._
- Use bfinject to decrypt the application:
    ```bash
    bash bfinject -P <app-name> -L decrypt
    ```
- Wait for bfinject to finish, then in the output you'll see bfinject saved the decrypted version in the application's `Documents` folder.
- On your computer, on a different terminal window, copy the dumped application to your machine:
    ```bash
    scp -P 2222 root@localhost:/private/var/mobile/Containers/Bundle/Application/{app-uuid}/Documents/decrypted-app.ipa ~/Desktop/
    ```
- Now you have a decrypted version of the app.

#### Extra information

To keep the step-by-step instructions as clean and straightforward as possible, I omitted some details in some steps. Here's some information that hopefully will clarify any doubts you may have:

**What `<app-name>` should I use with bfinject?**
- This is the name of the application folder. In some cases this name might be different from what's shown on your device's springBoard (the text beneath the app's icon).
- To find the correct name for your application:
    - Change directories to the Applications folder, all the user-installed applications are stored here:
    ```bash
    cd /private/var/containers/Bundle/Application
    ```
    - iOS assigns a random UUID to each app when installed from the App Store. Thus you'll need to search for your application on each of the random UUID folders. (_Note: this is why I asked you to download an application instead of using your already installed apps_) But here's a trick I've been using for a while, list the files/folders within this directory and sort them by date:
    ```bash
    ls -lat
    ```
    - Your recently downloaded application should be the very first one. If it's not, you'll have to go one by one until you find yours.
    - The `<app-name>` string you'll use with bfinject is the name of the folder inside the random UUID.

**bfinject throws the `Unknown jailbreak. Aborting.` error, what now?**
- bfinject was created for 2 very specific jailbreak setups, [Electra](https://coolstar.org/electra/) and [LiberiOS](http://newosxbook.com/liberios/). To identify an Electra jailbreak it checks if `/bootstrap/inject_criticald` exists and for LiberiOS it checks if `/jb/usr/local/bin/jtool` exists. The problem is that if you move some files around or your jailbreak doesn't have those files in those locations the script won't work. There is no one-answer-fits-all here. But the good news is that it's super easy to fix, all you need to do is add your setup to the `bfinject` script in [here](https://github.com/BishopFox/bfinject/blob/master/bfinject#L133-L149). At the time of the writing this is how the `bfinject` script check looks like:
```bash
#
# Detect LiberiOS vs Electra
#
if [ -f /bootstrap/inject_criticald ]; then
    # This is Electra
    echo "[+] Electra detected."
    cp jtool.liberios /bootstrap/usr/local/bin/
    chmod +x /bootstrap/usr/local/bin/jtool.liberios
    JTOOL=/bootstrap/usr/local/bin/jtool.liberios
    cp bfinject4realz /bootstrap/usr/local/bin/
    INJECTOR=/bootstrap/usr/local/bin/bfinject4realz
elif [ -f /jb/usr/local/bin/jtool ]; then
    # This is LiberiOS
    echo "[+] Liberios detected"
    JTOOL=jtool
    INJECTOR=`pwd`/bfinject4realz
else
    echo "[!] Unknown jailbreak. Aborting."
    exit 1
fi
```

**The application name has a whitespace and SCP won't copy it, what gives?**
- To copy files with whitespaces in SCP you'll need to add three (3) backslashes:
    ```bash
    scp -P 2222 root@localhost:/private/var/mobile/Documents/Dumped/app\\\ with\\\ space.ipa ~/Desktop/
    ```

**SCP is not working on my device on iOS < 11.0, do you even?**
- This is why we installed `Apple File Conduit 2` (AFC2), download [iExplorer](https://macroplant.com/iexplorer) or [iFunBox](http://www.i-funbox.com/) and transfer the decrypted app to your computer by navigating to the `/private/var/mobile/Documents/Dumped/` folder and drag-n-drop'ing the .ipa onto your computer.

[^1] There is a type of encryption called [`Homomorphic encryption`](https://en.wikipedia.org/wiki/Homomorphic_encryption) that aims to create a scheme where a system can perform operations on ciphertexts (encrypted data) and return results without reviling anything from the plaintext (raw binary).

### Bonus

If you see all these steps and you think "_a lot of these can be automated, no?_", you're absolutely right! I wrote a python script that automates all these steps, from starting iTunnel to copying the decrypted application to your computer and open sourced it under the MIT license so you can use it and modify it to fit your own needs. You can find this script [here](https://github.com/ivRodriguezCA/decrypt-ios-apps-script).

### Conclusions

- As you can see, _decrypting_ iOS applications is a fairly easy task. As long as you have a jailbroken device and the authors of Clutch and bfinject keep updating their tools, you'll be able to quickly decrypt iOS applications, and since both tools are open source you can help the community by contributing to those projects. The more difficult and exciting steps are to come when we get to decompile the binary, browse through the application bundle and analyze its embedded files.
