# Reverse Engineering iOS Applications

Welcome to my course `Reverse Engineering iOS Applications`. If you're here it means that you share my interest for application security and exploitation on iOS. _Or maybe you just clicked the wrong link_ 😂

All the vulnerabilities that I'll show you here are real, they've been found in production applications by security researchers, including myself, as part of bug bounty programs or just regular research. One of the reasons why you don't often see writeups with these types of vulnerabilities is because most of the companies prohibit the publication of such content. We've helped these companies by reporting them these issues and we've been rewarded with bounties for that, but no one other than the researcher(s) and the company's engineering team will learn from those experiences. This is part of the reason I decided to create this course, by creating a fake iOS application that contains _all_ the vulnerabilities I've encountered in my own research or in the very few publications from other researchers. Even though there are already some projects[^1] aimed to teach you common issues on iOS applications, I felt like we needed one that showed the kind of vulnerabilities we've seen on applications downloaded from the App Store.

This course is divided in 5 modules that will take you from zero to reversing production applications on the Apple App Store. Every module is intended to explain a single part of the process in a series of step-by-step instructions that should guide you all the way to success.

This is my first attempt to creating an online course so bear with me if it's not the best. I love feedback and even if you absolutely hate it, let me know; but hopefully you'll enjoy this ride and you'll get to learn something new. Yes, I'm a n00b!

If you find typos, mistakes or plain wrong concepts please be kind and tell me so that I can fix them and we all get to learn!

### Modules

- [Prerequisites](Prerequisites.md)
- [Introduction](Introduction.md)
- [Module 1 - Environment Setup](/Module-1/README.md)
- [Module 2 - Decrypting iOS Applications](/Module-2/README.md)
- [Module 3 - Static Analysis](/Module-3/README.md)
- [Module 4 - Dynamic Analysis and Hacking](/Module-4/README.md)
- [Module 5 - Binary Patching](/Module-5/README.md)
- [Final Thoughts](Final-Thoughts.md)
- [Resources](Resources.md)

### License

Copyright 2019 Ivan Rodriguez `<ios [at] ivrodriguez.com>`

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### Donations
I don't really accept donations because I do this to share what I learn with the community. If you want to support me just re-share this content and help reach more people. I also have an online store ([nullswag.com](https://nullswag.com)) with cool clothing thingies if you want to get something there.

### Disclaimer
I created this course on my own and it doesn't reflect the views of my employer, all the comments and opinions are my own.

### Disclaimer of Damages
Use of this course or material is, at all times, "at your own risk." If you are dissatisfied with any aspect of the course, any of these terms and conditions or any other policies, your only remedy is to discontinue the use of the course. In no event shall I, the course, or its suppliers, be liable to any user or third party, for any damages whatsoever resulting from the use or inability to use this course or the material upon this site, whether based on warranty, contract, tort, or any other legal theory, and whether or not the website is advised of the possibility of such damages. Use any software and techniques described in this course, at all times, "at your own risk", I'm not responsible for any losses, damages, or liabilities arising out of or related to this course. In no event will I be liable for any indirect, special, punitive, exemplary, incidental or consequential damages. this limitation will apply regardless of whether or not the other party has been advised of the possibility of such damages.

### Privacy
I'm not personally collecting any information. Since this entire course is hosted on Github, that's the [privacy policy](https://help.github.com/en/articles/github-privacy-statement) you want to read.

[^1] I love the work [@prateekg147](https://twitter.com/prateekg147) did with [DIVA](http://damnvulnerableiosapp.com/) and OWASP did with [iGoat](https://www.owasp.org/index.php/OWASP_iGoat_Tool_Project). They are great tools to start learning the internals of an iOS application and some of the bugs developers have introduced in the past, but I think many of the issues shown there are just theoretical or impractical and can be compared to a "_self-hack_". It's like looking at the source code of a webpage in a web browser, you get to understand the static code (HTML/Javascript) of the website but any modifications you make won't affect other users. I wanted to show vulnerabilities that can harm the company who created the application or its end users.
