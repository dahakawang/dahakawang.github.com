---
layout: post
title: "关于Visual C++增量链接以及.textbss"
date: 2011-08-01 22:08
comments: true
categories: [原理探求]
sharing: false
published: true
---
好的，文接上回，本文我就来讲讲微软link.exe连接器的Incremental Liking这个特性。当然这个其实不是微软linker独有的特性，很多链接器都有这个特性，这个特性实际上是为了提高链接速度的。

想象一下这个场景，我写了两个函数foo()和bar()，其中foo()在0x400100处而bar()紧接着保存在0x400200处。现在我将foo()改写了一下，添加了一些perfect的功能，然后编译了新的代码。不过现在的麻烦是foo()不可避免的变大了，他现在需要200h字节来保存了。那么链接器该怎么办？

<!-- more -->

一般的思考是——重新洗牌，将现有的编译好的exe删除了，然后重新布局所有的函数，也即是说bar()函数向后挪动0x100h字节的位置，给foo()腾出空间来。然后之后所有的函数都需要重新定位……对于大型软件来说这个处理时间开销是痛苦的，但作为程序员我们却不能避免需要不断的调试改代码，不断地重复这个耗时的工作。

不过我们现在并不需要给客户最终的发行代码，我们只是想要尽快地将程序的bug改掉然后去休息而已！于是，Incremental Linking出现了！它的原理如下：

![](/images/blogs/2012/increamental_linking.gif)

现在连接器不会将所有函数紧挨着放在一块儿了，他们会在函数之间加上padding，这个时候函数要想添几句指令就有余地了。只要我们的改动不大，没有超过padding的范围连接器就不需要重新洗牌，这大大提高了链接的速度。

先别高兴，加入我们的改动很大，以至于超过padding能够搞定的范围怎么办？如上图，我们还会在整个section末尾设置一个较大的padding（当然具体在哪里要看实现，比如我这图是从GCC那里搞得，说的就是ld.exe的行为方式），这时候就可以将这个函数搬到这里来了。但有个毁灭性的问题——所有调用我这个函数的函数都必须重定位他们的call指令啊！

为了解决这个问题，我们引入了一个ILT表（Incremental Linking Table），这个表是放在.text区域中的（我在IDA中观察得知）。它的原理是什么呢？我们来看：

``` nasm
;之前我们都是直接调用函数
   call foo

;现在我们来点小把戏
   call foo_stub

foo_stub:
   jmp foo
```

我们现在不直接调用函数，而是call到一个包含jmp指令的地方，然后由这个指令将我们的程度带往foo()函数的实现去。现在如果我们将foo()的实现改动过大后，linker直接将foo()移动了，然后只需要修改这个jmp指令就行了。可以看到，这种实现方式开销是O(1)。然后当很多个函数都用这种方式时，就形成了一个有jmp指令构成的表——这就是ILT表啦。

有兴趣的童鞋可以做下实验，在VS2010编译一次代码，然后用IDA或者W32Dasm之类的软件可以看到两个函数之间间隔了不少距离，而这些间隔就是我们所谓padding。padding被填充以**0xCCh**的数据。熟悉win32汇编的朋友这时候该笑而不语了，是的，这个值就是指令**INT 3**。在WIndows下，执行这个指令会引发一个异常，然后程序会被终止或是回到调试器去，这当然是出于安全性考虑的。这之后如果你在前一个函数加几句话，编译后可以看到两个函数位置不变，但函数间的padding变小了。

## 和.textbss的关系

之前有篇我讨论了PE常见的section，里面提到了这个节，下面我就详细介绍一下它的作用。

首先Incremental Linking作用不仅仅是在于减少我们重新连接程序所需要的时间，他还是我们调试时能够动态改动代码的前提。不知你还记得不，在那个炎热的夏天，你正汗流浃背地在没有空调的部屋里调试C代码（咳，说远了……）你直接修改了代码，然后VS直接在调试的时候将你的改动反映到程序里去了。这就是VS在Debug模式时动态编译代码的功能。

实际上这个功能是基于Incremental Linking机制的，而且是使用的Incremental Linking的第二套方案——直接找个大的地儿把修改的函数挪过去。

但是和.textbss有啥关系？

首先我们看到，.textbss有关键字bss，这就说明实际上这个节没有占据实际的硬盘空间。然后text关键字告诉我们这里段是包含代码的，另外用工具查知这个段有可执行属性更是印证了这个观点。没有代码，那要这个节有啥用呢？

你想到了么？是的，在VS动态编译的时候，他直接将被修改的函数放到了.textbss节里，然后修改了对应的ILT表项，是他指向这个位置。

说是简单，但实际上这个过程还个细节需要注意——你把我的函数挪地儿了，要是我正在执行这个函数怎么办？实际上，在改了ILT之后立刻会做的，就是检查当前程序所有线程的TIB，如果他们的EIP指向老的函数（它们正在执行老版本函数），我们就修改EIP使其指向新版本函数的对应位置。当然，这实际上暗示了，这个工作非要在调试程序的帮助下不可了。注意动态编译的功能只在Debug版本程序下有效,Release版本是不行的，因为Release版本默认禁用Incremental Linking。

下面是实验时间，以验证我的观点：

我就用手头上的程序来测试。有函数CheckValidPE()，它的RVA是0x12490h（你可理解为内存地址）。我的程序地址空间部分如下：

 Section　　　|　　　  RVA　　　|　　　Size
 .textbss　　　|　　　1000h　　　|　　10000h
 .text　　　　　|　　 11000h　　|　　　7000h
 
 可以看到这两个函数实际上是位于.text节内的。我在CheckValidPE()上下个断，可以看到：

``` nasm
 bool PEAnalyser::checkValidPE()
{
00D92490  push        ebp
00D92491  mov         ebp,esp
00D92493  sub         esp,0FCh
00D92499  push        ebx
00D9249A  push        esi
00D9249B  push        edi
00D9249C  push        ecx
......
```

从这里我发觉了VS貌似从来都是随机装载PE映像的ImageBase位置的，搞得我之前一次实验满心欢喜用常用的0x400000h为基址换算了所有的RVA~~T_T。。。

我之前说了CheckValidPE()的RVA是0x12490h，这里的绝对地址是0x00D92490h，我们用0x00D92490h-0x12490h = 0xD80000h，得到两个的差值。哈哈，看来这次的映像基址被射到了0xD80000h。

我回到VS源代码视图上，对代码稍作粉饰，嗯，然后它发生了！！！

``` nasm
bool PEAnalyser::checkValidPE()
{
00D81000  push        ebp
00D81001  mov         ebp,esp
00D81003  sub         esp,0FCh
00D81009  push        ebx
00D8100A  push        esi
00D8100B  push        edi 
00D8100C  push        ecx
00D8100D  lea         edi,[ebp-0FCh]
00D81013  mov         ecx,3Fh
00D81018  mov         eax,0CCCCCCCCh
00D8101D  rep stos    dword ptr es:[edi]
.....
```

注意看函数变到这个地址来了！！而且VS的调试程序指针也确实说明EIP被更改了。

我们来算一下，看看这个地址在哪里？用这里新的函数起始地址减去我们之前计算的的基址得到RVA = 0xD81000h - 0xD80000h = 0x1000h

回去查一下我的内存地址空间，RVA为0x1000h正是位于.textbss节的起始位置。看来我的猜测是正确的。

## 小结

小结一下，关于Incremental Linking，由于他的机制所致，势必带来程序体积的臃肿以及执行的低效，但是由于我们只是在Debug程序是使用，所以问题不大。另外VS默认是在Debug是开启Incremental Linking而Release模式关闭这个特性的。这说明在Release时，我们不能够动态的改动代码了。

另外注意Incremental Linking是和/LTCG 选项不兼容的，你不能同时开启Incremental Linking和Link Time Code Generation，从这个角度讲，使用Incremental Linking进一步会造成程序执行效率下降。所以，我们应该在发布程序时，注意避免带上这个特性。
