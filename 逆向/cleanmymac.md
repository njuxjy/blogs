> 注：本文`2015.09`发表于简书，内容可能已过时，此处仅做归档

使用工具: `class-dump`, `Hopper Disassembler`。

`CleanMyMac`试用版的限制是：只能清理最多500MB的垃圾。这里要做的是在试用版下也能突破这个限制。
首先用class-dump导出所有头文件备参考，用Hopper打开可执行文件，应该就是`CleanMyMac 3`：

![](http://cdn.iosre.com/uploads/default/original/2X/0/0bbe2ec730174278d72f3e4212acc4e361fbabe8.png)

当超过500MB以后，每次点清理按钮会弹出下面的窗口：

![](http://cdn.iosre.com/uploads/default/optimized/2X/e/e5d6ff481b8be90d2189d58e7e19406e51ddd032_2_690x345.png)

在Hopper中搜索窗口中字符串"I already have a license"：

![](http://cdn.iosre.com/uploads/default/optimized/2X/c/ce65b1535480dd6b2cd4b611c14ed2b5d204f330_2_690x98.png)

得到变量名`cfstring_I_already_have_a_license`，然后继续搜索该变量：

![](http://cdn.iosre.com/uploads/default/optimized/2X/1/15cc753d9f8dcb60c1590bcb97c740e17d5f967a_2_690x81.png)

可以看到`[CMPurchaseViewController loadView]`中引用到了它，从名字判断这个`controller`应该对应了弹出来的窗口。应该是点了按钮后什么东西初始化了这个`controller`，之前的探索方向可能有点问题，应该去找`button`的处理函数。于是在`Hopper`搜索框里猜测`buttonHandler`，`buttonTapped`，`buttonClick`等等之类，最后发现有个叫`ubiquitousButtonPurchaseClicked`的函数很可疑：

![](http://cdn.iosre.com/uploads/default/optimized/2X/8/88f0e4f1e6c8e92a625d29e9fb5d56bcb3568c7f_2_312x500.png)

![](http://cdn.iosre.com/uploads/default/original/2X/b/b6d362653c88dd52b531c6813c412aadc0e419d4.png)

在`0x1006dec58`的位置引用到了它，于是顺藤摸瓜到这个地址：

![](http://cdn.iosre.com/uploads/default/original/2X/3/37318c2a538fc64f0151c3c678593b3604e1f2df.png)

是被这个函数给引用了：`-[CMUbiquitousButtonView performActionOnDelegate]`
打开它的头文件，发现里面有个mouseDown:函数，应该就是按钮处理函数，但是是不是清理按钮现在还不知道，总之先来看看`mouseDown:`的实现。 用Hopper找到它，查看伪代码：

![](http://cdn.iosre.com/uploads/default/original/2X/6/6f4eac217d1f3ba73bdf75350d93d9c3d15271c2.png)

可以看到最后还是调用了`performActionOnDelegate`，看来是在这个函数里决定怎么处理接下来的逻辑，那么就继续看看`performActionOnDelegate`的实现：

![](http://cdn.iosre.com/uploads/default/original/2X/1/16fdf8a8549fbcc767b24a8f75163b365fc6c987.png)

哈哈，在这个函数里很惊喜的看到了疑似开始清理过程的`ubiquitousButtonStartCleanProcess:`函数，迫不及待的打开看看：

![](http://cdn.iosre.com/uploads/default/original/2X/0/05f5419fad3ca9faf48ca533985a67f727e3bb37.png)

有两个类有这个函数，这里可以通过打断点判断得出是`CMModuleViewController`调用了。看到实现：

![](http://cdn.iosre.com/uploads/default/optimized/2X/c/cf10d37718c475ebb3653f6585e0c028b6213c78_2_690x164.png)

于是痛苦开始了，关键部分做了代码混淆， 函数名变成了：`JwUMdSW7rENEEIgVEGUtnns7cx3JNc9UTOuabo1ThwTJyDSydBKrFyvIIyTL6IgTvp9KwMqg1pF1VqvBEF8tC8YjbebfGbCBzIiRHQiX1wgasjtB0yneXyLo8vUGJhOmWNxu6FDurz8vOBkOSCpGfyGpMC8S1eJS8VWY9JRKfv7dahJuH0MAth7SwKv48LilHi63doAFcf1WDN2c7aJErpPKXKh3n08CPwiOcQxI888pDSR6K4XcjiWsYV3zHreX`，
姑且称为`加密函数A`，是属于CMMainWindowController的某个敏感函数，看看它的实现如下：

![](http://cdn.iosre.com/uploads/default/optimized/2X/d/de11b29b6ea88c3b188355ff3387eb905c5f6f2b_2_690x202.png)

忍不住吐血，里面又调了一个`加密函数B`。不管了，继续看B的实现，由于太长了就不贴出来了，总之里面又调了一堆加密函数。这段时间比较痛苦，一度想放弃，尝试着去理清程序执行的顺序，但又没有get到Hopper其实可以单步调试的技能，一度都是用下面的手段让程序挂掉来得知程序的执行流程：

```
MOV   AX,  4C00H

INT   21H
```

注意到伪代码里经常出现一个类似下面的代码片段：

![](http://cdn.iosre.com/uploads/default/original/2X/d/dda9c100ca607b9357281c475d86a8b8332ebb0e.png)

猜测是`block`调用，于是写了段代码用`Hopper`反汇编一下验证了果然是。

对于`加密函数B`，通过修改汇编的判断条件，让程序避过了弹出警告框的逻辑，最后发现这个函数其实没做什么实质的清理工作，只不过是在各种判断用户有没有权限进行清理。最实质性的调用是这一句：

![](http://cdn.iosre.com/uploads/default/original/2X/e/e8f52144f9a30eaeb86aadf2d671599990bfda1e.png)

这其实是一句`block`调用，参数是1，`block`是外面传进来的。接下来就在函数开始合适的地方加上它的汇编代码，汇编代码如下：

```
mov        qword [ss:rbp + var_D8], r15

mov        esi, 0x1

mov        rdi, qword [ss:rbp + var_D8]
call       qword [ds:rdi+0x10]

jmp        0x100207266
```

这里有个插曲，直接加上如上的代码会让Hopper挂掉。查了下原因是`Hopper`这个版本还不支持编辑的时候引用`var_`开头的变量，尝试换了种办法，`D8 == 十进制的216`，所以上面的汇编代码等价于：

```
mov        qword [ss:rbp - 216], r15

mov        esi, 0x1

mov        rdi, qword [ss:rbp - 216]

call       qword [ds:rdi+0x10]

jmp        0x100207266
```
这里又要看到之前的`ubiquitousButtonStartCleanProcess:`函数，然后结合`加密函数A和B`可以知道，`ubiquitousButtonStartCleanProcess`里有个`block`，然后把`block`丢给`加密函数A`，`加密函数A`又把`block`丢给`加密函数B`，由B执行到最后再调用了这个`block`。这种执行流程很像是这个`block`就叫`onAuthorizeSuccess`，两个加密函数做了点能否执行清理的判断，如果成功的话执行`onAuthorizeSuccess block`。 那么这里就应该看到`ubiquitousButtonStartCleanProcess:`里的`block`执行体，也就是`sub_1000dd914`函数：

![](http://cdn.iosre.com/uploads/default/optimized/2X/4/49d393fd83f410e32d1340c2e828cf5e3b614347_2_526x500.png)

接下来一路顺藤摸瓜，从`-[CMModuleViewController startClean]:`到`-[CMGroupScanner startClean]:`到`sub_100090e3a`到`-[CMGroupScanner cleanWithSession]`到`-[CMScanner cleanWithSession]`到`-[CMScanner cleanThreadWithSession]`到`-[CMScanner recursivelyCleanNode:parentNode:session:]`都比较顺利，最后在`-[CMScanner recursivelyCleanNode:parentNode:session:]`里发现有很多`-[shouldScanner:pauseCleaningWithNextNodeToClean:]`函数，这个函数有好几个类里都有，一一把它们直接返回`false`。
以为大功告成了，跑一下程序，发现挂了。幸好`Hopper`的`debugger`给出了`exception`的位置，发现是`加密函数B`中由于改变了程序执行流程，导致最后某个不需要`release`的变量被`release`了，于是把这局操作置空就行。

以为接下来肯定大功告成了，结果发现清理系统垃圾的时候需要管理员权限，而被`patch`过的程序始终无法成功。这里牵涉到了`SMJobBless`和`privileged helper tool`等mac上的获取系统权限接口，搞了两天没搞定。 最后只能简单粗暴的让`-[CMAgentController install]`返回false来跳过所有需要系统权限的垃圾的清理。

有大神知道怎么搞定`SMJobBless`的欢迎补充。
