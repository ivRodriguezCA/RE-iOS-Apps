### Module 5 - Binary Patching

This is the 5th and final module of this course. I hope you've been having as much fun as I had creating this course, actually if you're having 1 unit above zero fun I'll be happy. 😂 If you felt like the last two modules were way too long, I have good news for you. This is going to be a very short module. You've learned most of what I can teach you (or what I can come up with for this course).

`Cycript` and `Frida` can handle pretty much any job you need to modify the behaviour of an iOS appliation. But they have two main problems:

- All the changes to the runtime (not including writing to or modifying files on disk) live in memory. This means that every time you want to modify these methods you have to re-run your scripts.
- As you saw with both tools, you need to run the application on your phone before you can start using them. This means that all the logic that's executed when the application is initialized has already happened and cannot be modified with either of the tools. For example in the `CoinZa` application, the jailbreak detection was done in the `viewDidLoad` method of the first controller ever created by the application, thus you had to built the controller again and present it.

Sometimes these problems become really annoying and that's when the final tool in your toolbox comes to play. Binary patching will allow you to modify the assembly code of any iOS application then repackage it and install it on your device and the modified (patched) version of the application will run natively on your devices without needing to load any scripts to modify its behaviour.

For this exercise I'm going to exclusively use `Hopper` because I don't know how to do this in `Ghidra` yet (or if it's even possible). If I later figure it out I'll update the course with both versions.

- Load the `CoinZa` binary in `Hopper`.
- Search for the `Utils` class and select the `isJailbroken` method.
- If you're not there, select the `CFG mode` tab to see the method's assembly code.
- As most of the jailbreak detection methods, this is a long list of checks with early returns if any of this checks is successful. In the flow graph we can see that most of the checks have a jump to a label `loc_100009cc4` (this might be different on your end), but the important part is that the instruction that's executed when jumping to this label:
    ```assembly
    orr        w20, wzr, #0x1
    ```
    - This is the `or` operation between the `wzr` register and `0x1`, this means that the result in the register `w20` will always be `1` (or `true`) because `x or 1 = 1`. This basically setting the value 1 to the register `w20`.
- On the other side of the flow you can see the following two instructions being executed before jumping:
    ```assembly
    cmp        x8, #0x0
    cset       w20, eq
    ```
    - The first instructions compares the value in the register `x8` to `0x0` and then the second instruction is set based on that comparison. (Based on the ARM [documentation](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0802b/CSET_CSINC.html), the register is technically increased if the comparison is `true`).
- These are the two exit points for the `isJailbroken` method. All you have to do is set the value of the register `w20` to `0x0` (or `false`) and the function will always return `false`.
- To modify the assembly code:
    - Select the `orr` instruction by clicking on it.
    - Press `⌥ + A` or from the top menu select `Modify` and then `Assemble instruction...`
    - Hopper will show a window with a text field enter:
      ```assembly
      movz w20, #0x0
      ```
      - This will replace the existing instruction with this one. What this instruction is doing is loading the value `0x0`onto the register `w20`.
    - Follow the same steps for the `cset` instruction.

![Hopper](https://github.com/ivRodriguezCA/RE-iOS-Apps-Extras-Github/blob/master/Module-5/binary-patching.png?raw=true)
- Once you've replaced the two instructions, press `⇧ + ⌘ + E` or from the top menu select `File` and then `Produce New Executable...`.
- Name the new binary `CoinZa` and click `Save`.
- Replace the existing `CoinZa` binary in `CoinZa.app`.
- Repackage the application:
    ```bash
    zip -qry CoinZa-Frida.ipa Payload
    ```
- Now you can follow the same `Cydia Impactor` steps, just like you've done so far, and install the application on your device.
- Run the application by tapping on its icon.
- The application doesn't show the "Sorry not sorry" popup and you can use the application as normal.

#### Conclusions
- Patching the binary is a very powerful technique that will allow you to modify applications from its source and re-install them on your devices as if they were applications downloaded from the App Store.
- This technically not part of reverse engineering a binary since by now you probably already finished the reversing and are using that knowledge to your advantage.
- This step is a little bit more difficult than most of the previous steps because it requires knowledge of ARM assembly.
