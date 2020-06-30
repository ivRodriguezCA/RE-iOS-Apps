### Module 1 - Environment Setup

In this module you'll learn how to setup your device(s) with all the tools you'll need to decrypt apps, transfer them to your computer and perform static and dynamic analysis on them. I'm assuming you already have a jailbroken device. If you don't have a device, you can skip to module 3.

_Note: If you need help jailbreaking your device, there are many resources online. One of my favourite sites is [iDownloadblog](https://www.idownloadblog.com/jailbreak/)._

#### On your computer

- Download the latest version of [iTunnel](https://code.google.com/archive/p/iphonetunnel-usbmuxconnectbyport/downloads): iTunnel will allow you to [SSH over USB](https://iphonedevwiki.net/index.php/SSH_Over_USB).
- Download the latest version of [Cydia Impactor](http://www.cydiaimpactor.com/): Impactor will allow you install iOS applications on your device, signed with a developer account's certificate.
- Download and install [Hopper](https://www.hopperapp.com/): Hopper is a reverse engineering tool that lets you disassemble, decompile and debug ARM applications, it supports other architectures but in this course I'll focus just on ARM-based binaries. The trial version is enough.
- Download the latest version of [Cycript](http://www.cycript.org/): Cycript will allow you to modify the applications' behaviour at runtime via an interactive console.
- Download the latest version of [Frida](https://www.frida.re/docs/ios/): Frida will allow you to write scripts to change the applications' behaviour at runtime.
    - To install `Frida`:
    ```shell
    sudo pip install frida-tools
    ```
- Download the latest version of [Bettercap](https://www.bettercap.org/installation/): Bettercap will allow you to perform MitM attacks remotely to a device.
- Download the latest version of [class-dump-z](https://code.google.com/archive/p/networkpx/downloads): class-dump-z will allow you to dump Objc classes. There's a Swift version but you won't need it since my vulnerable app is written in Objc.
- Download the latest version of [momdec](https://github.com/atomicbird/momdec): momdec will allow you to decompile CoreData models.
- Download the latest version of [Ghidra](https://ghidra-sre.org/): Ghidra is another reverse engineering tool, which will let you do some of the same tasks as Hopper.
  ##### If your device is on iOS 10.x
  - Download the latest version of [Clutch](https://github.com/KJCracks/Clutch/releases): Clutch will allow you to decrypt iOS applications.

  ##### If your device is on >= iOS 11
  - Download the latest version of [bfinject's](https://github.com/BishopFox/bfinject) `bfinject.tar`: bfinject will allow you to use `Cycript` and `Clutch` to decrypt iOS applications.

  ##### If your device is on iOS 12.x
  - Download the latest version of [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump): `frida-ios-dump` will allow you to decrypt iOS applications and transfer them automatically to your computer.
  - Install its dependencies `sudo pip install -r requirements.txt --upgrade`.

#### On your device with iOS version < 11.0

- Open [Cydia](https://cydia.saurik.com/) and search `cycript` and install it.
- Open Cydia and search `Apple File Conduit "2"` and install it.
- Open Cydia and search `frida` and install it:
    - Tap the `Sources` tab.
    - Add a source: `https://build.frida.re`
    - Now you can go to the `Search` tab and search for `frida`.

#### (Optional) On your device with iOS version < 11.0
In some cases a jailbreak tool for iOS < 11.0 might not come with a SSH client, you might have to [install it yourself](https://ivrodriguez.com/installing-dropbear-ssh-on-ios-10-3-3/). To test if your device already has a working SSH:
- Connect your device to your computer.
- On your computer, open a terminal window and run `iTunnel` with the following parameters:
    ```bash
    itnl --lport 2222 --iport 22
    ```
    - `lport`: Stands for `Local port` and it's the port iTunnel will be locally listening. This can be any port you want.
    - `iport`: Stands for `iPhone port` and it's the port iTunnel will use to forward all the packets sent to `lport`. This has to be `22`, since that's the `SSH` default port.
- On a different terminal window SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
    - `p`: Stands for `port`, this is the port iTunnel is listening on.

If your device asks for a `root` password then it _already_ has SSH working, thus you can skip this step.

#### On your device with iOS > 11

- Connect your device to your computer.
- On your computer, open a terminal window and run `iTunnel` with the following parameters:
    ```bash
    itnl --lport 2222 --iport 22
    ```
    - `lport`: Stands for `Local port` and it's the port iTunnel will be locally listening. This can be any port you want.
    - `iport`: Stands for `iPhone port` and it's the port iTunnel will use to forward all the packets sent to `lport`. This has to be `22`, since that's the `SSH` default port.
- On a different terminal window SSH into your device:
    ```bash
    ssh -p 2222 root@localhost
    ```
    - `p`: Stands for `port`, this is the port iTunnel is listening on.
    - Your device will ask you for the `root` password. The default password is `alpine`, but I'd advice you [to change it](https://cydia.saurik.com/password.html).
- Create a `jb` folder on your root directory:
    - _Note: If you use [LiberiOS](http://newosxbook.com/liberios/) there's already a `/jb` folder, just change directories._
    ```bash
    cd / && mkdir jb
    ```
- Create a `bfinject` folder inside `/jb` and change directories:
    ```bash
    mkdir /jb/bfinject && cd /jb/bfinject
    ```
- In a different terminal window, copy the `bfinject.tar` archive to the device:
    ```bash
    scp -P 2222 ~/Downloads/bfinject.tar root@localhost:/jb/bfinject
    ```
    - `P`: Stands for `port` and it should be the same port iTunnel is listening on. _Note: This is a capital `P`_.
    - Your device will ask you for the `root` password.
- Extract the .tar file contents:
    ```bash
    tar xvf bfinject.tar
    ```

#### On your device with iOS 12.x
- Open Cydia and search `frida` and install it:
    - Tap the `Sources` tab.
    - Add a source: `https://build.frida.re`
    - Now you can go to the `Search` tab and search for `frida`.

(*Note: Since I've only used the Unc0ver jailbreaks I don't know if you're jailbroken with [Chimera](https://chimera.sh/) and/or use `Sileo` as your package manager if you can install Frida.*)

### Conclusions

- Now you should have a device ready to start reversing. Gladly you'll need to perform all these steps only once per device, even when you lose your jailbreak state if your device runs out of batter or restarts for whatever reason[^1]. Don't worry if you don't know some of these tools, in the following modules I'll explain what's their purpose and how to use them.


[^1] On tether and semi-tether jailbreaks, every time you restart your device you'll need to re-jailbreak it because the jailbreak exploit is not persisted after reboot.
