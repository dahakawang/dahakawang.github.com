---
layout: post
title: "SRS Audio Sandbox破解纪实"
date: 2012-06-22 22:37
comments: true
categories: 逆向破解
sharing: false
published: true
---
最近挺忙的，本来需要各种为前路做准备，无奈自己天生属于低血压型的人，偏偏就是提不起劲儿来干正事儿，却把大好的一天光阴全交代到SRS的分析工作上了。虽然说这多少算不务正业，不过由于本人有着严重的软件新版本强迫症，故这算是给自己的一个开脱理由吧。

这款软件的破解工作展开比较容易，没有加壳，直接上工具分析之。首先请出OllyDBG来，通过查找字符串引用和查找API MessageBox引用的老法子，很容易就定位了大概的关键代码位置。

<!-- more -->

找到关键位置后，顺手又记录了一下函数调用栈，然后打开了神器IDA。用IDA载入后，跳转到之前找到的关键代码RVA处，着手分析。说起来，我自己喜欢将动态和静态调试方法结合起来用，不如果不愿意太费神分析汇编代码，有时候直接看运行时结果便是最直观和省时的。载入后发觉之前判断的关键点实际上是CWinApp::ShowAppMessageBox函数，继续沿着刚才记录的函数运行时调用栈往上找，依次顺藤摸瓜：AfxMessageBox--->Sub44D240，这才到了SRS程序的代码领空，于是正是着手分析。（IDA这个静态分析功能实在是强大，它可以根据二进制代码特征判断出其是否是库函数代码。这大大节省了我的时间，应为对这类代码我只需要看SDK文档就行了，而不需要去分析其实际的代码行为）

进一步根据线索判断（Sub44D240里包含了SendMessage、AfxMessageBox、GetWindowText等函数，足以说明该函数的关键作用），我锁定在了Sub44D240函数的Sub4430F0调用上，继续追踪到Sub442FF0上，继而是Sub44B9E0过程，最终到了Sub44BC30过程上。在该过程中，我同时追踪到了用户输入的ProductID以及Serial Number等信息，这进一步是我确认了我的判断。

该过程具体代码如下：

``` nasm Sub44BC30代码

  .text:0044BC30
  .text:0044BC30 ; =============== S U B R O U T I N E =======================================
  .text:0044BC30
  .text:0044BC30 ; Attributes: bp-based frame
  .text:0044BC30
  .text:0044BC30 check_PID_SN    proc near               ; CODE XREF: realCheckSN+182p
  .text:0044BC30
  .text:0044BC30 productId_segs_temp= dword ptr -14h
  .text:0044BC30 new_ProdectID   = dword ptr -10h
  .text:0044BC30 sn_lowerDWord   = dword ptr -4
  .text:0044BC30 SN              = dword ptr  8
  .text:0044BC30
  .text:0044BC30                 push    ebp
  .text:0044BC31                 mov     ebp, esp
  .text:0044BC33                 and     esp, 0FFFFFFF8h
  .text:0044BC36                 sub     esp, 14h
  .text:0044BC39                 push    ebx
  .text:0044BC3A                 push    esi
  .text:0044BC3B                 push    edi
  .text:0044BC3C                 push    14h             ; unsigned int
  .text:0044BC3E                 mov     ebx, eax        ; ebx指向ProductID首位置
  .text:0044BC40                 call    j_??2@YAPAXI@Z  ; operator new(uint)
  .text:0044BC45                 mov     esi, eax
  .text:0044BC47                 lea     eax, [esi+6]
  .text:0044BC4A                 push    eax
  .text:0044BC4B                 mov     eax, [ebp+SN]
  .text:0044BC4E                 lea     ecx, [esi+4]
  .text:0044BC51                 push    ecx
  .text:0044BC52                 lea     edx, [esi+2]
  .text:0044BC55                 push    edx
  .text:0044BC56                 push    esi             ; 新申请的14h的空间
  .text:0044BC57                 push    offset a4x4x4x4x ; "%4x-%4x-%4x-%4x"
  .text:0044BC5C                 push    eax             ; SN首地址
  .text:0044BC5D                 call    String2Integer  ; 将字符串SN转换为数组，比如
  .text:0044BC5D                                         ; “E6E9-24CB-2968-09BC”变为
  .text:0044BC5D                                         ; E9 E6 ...
  .text:0044BC62                 mov     ecx, [esi+4]    ; ECX为后半部分SN
  .text:0044BC65                 mov     edi, [esi]      ; EDI为前半部分SN
  .text:0044BC67                 push    esi             ; void *
  .text:0044BC68                 mov     [esp+40h+sn_lowerDWord], ecx
  .text:0044BC6C                 call    j__free
  .text:0044BC71                 lea     edx, [esp+40h+productId_segs_temp]
  .text:0044BC75                 push    edx
  .text:0044BC76                 push    offset a4x      ; "%4x"
  .text:0044BC7B                 push    ebx
  .text:0044BC7C                 call    String2Integer  ; 将ProductID第一段转换为Word
  .text:0044BC81                 mov     esi, [esp+4Ch+productId_segs_temp] ; esi储存productId_1
  .text:0044BC85                 lea     eax, [esp+4Ch+productId_segs_temp]
  .text:0044BC89                 push    eax
  .text:0044BC8A                 lea     ecx, [ebx+0Ah]
  .text:0044BC8D                 push    offset asc_48C804 ; "%x"
  .text:0044BC92                 push    ecx
  .text:0044BC93                 call    String2Integer
  .text:0044BC98                 mov     edx, [esp+58h+productId_segs_temp] ; EDX储存ProductId_2
  .text:0044BC9C                 add     esp, 38h
  .text:0044BC9F                 xor     eax, eax
  .text:0044BCA1                 push    eax
  .text:0044BCA2                 push    esi
  .text:0044BCA3                 push    eax
  .text:0044BCA4                 push    edx
  .text:0044BCA5                 call    __allmul
  .text:0044BCAA                 mov     esi, eax        ; product_1*product_2乘积的低word放入esi
  .text:0044BCAC                 lea     eax, [esp+20h+productId_segs_temp]
  .text:0044BCB0                 push    eax
  .text:0044BCB1                 lea     ecx, [ebx+14h]
  .text:0044BCB4                 push    offset asc_48C804 ; "%x"
  .text:0044BCB9                 push    ecx
  .text:0044BCBA                 mov     [esp+2Ch+new_ProdectID+4], edx
  .text:0044BCBE                 call    String2Integer
  .text:0044BCC3                 mov     edx, [esp+2Ch+new_ProdectID+4]
  .text:0044BCC7                 mov     eax, [esp+2Ch+productId_segs_temp] ; eax保存productId_3
  .text:0044BCCB                 add     esp, 0Ch
  .text:0044BCCE                 push    edx
  .text:0044BCCF                 push    esi
  .text:0044BCD0                 push    0
  .text:0044BCD2                 push    eax
  .text:0044BCD3                 call    __allmul
  .text:0044BCD8                 lea     ecx, [esp+20h+productId_segs_temp]
  .text:0044BCDC                 push    ecx
  .text:0044BCDD                 push    offset asc_48C804 ; "%x"
  .text:0044BCE2                 add     ebx, 1Eh
  .text:0044BCE5                 push    ebx
  .text:0044BCE6                 mov     esi, eax
  .text:0044BCE8                 mov     [esp+2Ch+new_ProdectID+4], edx
  .text:0044BCEC                 call    String2Integer
  .text:0044BCF1                 mov     edx, [esp+2Ch+new_ProdectID+4]
  .text:0044BCF5                 mov     eax, [esp+2Ch+productId_segs_temp]
  .text:0044BCF9                 add     esp, 0Ch
  .text:0044BCFC                 push    edx
  .text:0044BCFD                 push    esi
  .text:0044BCFE                 push    0
  .text:0044BD00                 push    eax
  .text:0044BD01                 call    __allmul        ; 结果edx高位,eax低位
  .text:0044BD06                 mov     esi, edx
  .text:0044BD08                 shr     esi, 10h
  .text:0044BD0B                 mov     [esp+20h+new_ProdectID], eax
  .text:0044BD0F                 mov     [esp+20h+new_ProdectID+4], edx
  .text:0044BD13                 xor     ebx, ebx
  .text:0044BD15                 mov     cl, 10h
  .text:0044BD17                 call    __allshr        ; {edx,eax}==new_prodectID >> 10h
  .text:0044BD1C                 mov     ecx, [esp+20h+new_ProdectID+4]
  .text:0044BD20                 xor     edx, edx
  .text:0044BD22                 push    1
  .text:0044BD24                 and     eax, 0FFFF0000h
  .text:0044BD29                 push    edx
  .text:0044BD2A                 add     esi, eax
  .text:0044BD2C                 adc     ebx, edx
  .text:0044BD2E                 mov     edx, [esp+28h+new_ProdectID]
  .text:0044BD32                 push    ecx
  .text:0044BD33                 push    edx
  .text:0044BD34                 call    __allmul        ; eax=0, edx为new_productID的低位
  .text:0044BD39                 push    0
  .text:0044BD3B                 add     esi, eax
  .text:0044BD3D                 push    8475h
  .text:0044BD42                 adc     ebx, edx
  .text:0044BD44                 push    ebx
  .text:0044BD45                 push    esi
  .text:0044BD46                 call    __alldiv        ; {edx,eax} == {edx,esi} / 0x8475
  .text:0044BD4B                 push    0
  .text:0044BD4D                 push    0AE6000h
  .text:0044BD52                 push    edx
  .text:0044BD53                 push    eax
  .text:0044BD54                 call    __allmul
  .text:0044BD59                 add     eax, 91F2884Dh
  .text:0044BD5E                 adc     edx, 2DCh
  .text:0044BD64                 mov     cl, 0Ah
  .text:0044BD66                 call    __allshr
  .text:0044BD6B                 push    0
  .text:0044BD6D                 push    2046h
  .text:0044BD72                 push    edx
  .text:0044BD73                 push    eax
  .text:0044BD74                 call    __allmul
  .text:0044BD79                 mov     ecx, 0FFFFFFFEh
  .text:0044BD7E                 sub     ecx, eax
  .text:0044BD80                 mov     eax, 0FFFFFFFFh
  .text:0044BD85                 sbb     eax, edx
  .text:0044BD87                 mov     edx, [esp+20h+sn_lowerDWord]
  .text:0044BD8B                 shld    edx, edi, 1
  .text:0044BD8F                 add     edi, edi
  .text:0044BD91                 cmp     edi, ecx
  .text:0044BD93                 jnz     short loc_44BDA5
  .text:0044BD95                 cmp     edx, eax
  .text:0044BD97                 jnz     short loc_44BDA5
  .text:0044BD99                 mov     eax, 1
  .text:0044BD9E                 pop     edi
  .text:0044BD9F                 pop     esi
  .text:0044BDA0                 pop     ebx
  .text:0044BDA1                 mov     esp, ebp
  .text:0044BDA3                 pop     ebp
  .text:0044BDA4                 retn
  .text:0044BDA5 ; ---------------------------------------------------------------------------
  .text:0044BDA5
  .text:0044BDA5 loc_44BDA5:                             ; CODE XREF: check_PID_SN+163j
  .text:0044BDA5                                         ; check_PID_SN+167j
  .text:0044BDA5                 pop     edi
  .text:0044BDA6                 pop     esi
  .text:0044BDA7                 xor     eax, eax
  .text:0044BDA9                 pop     ebx
  .text:0044BDAA                 mov     esp, ebp
  .text:0044BDAC                 pop     ebp
  .text:0044BDAD                 retn
  .text:0044BDAD check_PID_SN    endp
```

## 基本思路

通过分析，我了解到，其算法大概思路如下：首先，注册流程有效输入为Serial Number和Product ID，Registration No实际上没有参与注册的验证计算过程。SRS通过系统各种信息生成Product ID，然后用户需要提供与Product ID匹配的Serial Number方能注册。当然，Serial Number是需要你拿美刀换的。

简而言之，SRS将Product ID作fp变换，得到一个64bit长的整数，并将用户输入的序列号做fs变换同样得到一个64bit长整数。为了使注册成功，需要满足：

~~~
fp(gen()) = fs(SerialNumber) 成立      (1)
~~~

这其中，gen()的算法我们不需要管，因为其结果在界面Product ID框中已经显示了。我们需要找到fp和fs的实现算法，并顺利推出fs-1的实现。从而：
SerialNumber = fs<sup>-1</sup>( fp(ProductID) )   (2)

## Product ID变换算法描述

我们首先描述fp的实现。fp基本上是由我不知道原理的各种数值变换组成，为了精确表述，我直接用C语言描述：

``` c
Product ID变换函数描述
 __int64 getProductID(char* id){
     __int64 temp = 1;
     DWORD elem;
 
     for(int i = 0; i < 4; i++){
         sscanf(id + (i * 5), "%x", &elem);
         temp *= elem;
     }
 
     //高地位变换
     DWORD lowerDWord = temp; 
     DWORD upperDWord = temp >> 0x20;
     WORD lower = upperDWord;
     WORD upper = upperDWord >> 0x10;
     upperDWord = lower;
     upperDWord <<= 0x10;
     upperDWord |= upper;
     
     temp = lowerDWord;
     temp <<= 0x20;
     temp |= upperDWord;
     
 
     temp /= 0x8475;
     temp *= 0xAE6000;
     temp += 0x2DC91F2884D;
     temp >>= 0xA;
     temp *= 0x2046;
     temp = 0xfffffffffffffffe - temp;
 
     return temp;
 }
```

## Serial Number变换算法描述　　


接下来描述fs函数的实现。我们称变换后的Product ID为TransPID，并且LowerDW和HighDW表示一个64bit整数的低双字和高双字，shld表示对应汇编指令的函数,TransSerialNumberHighDW和TransSerialNumberLowDW分别表示变换后的序列号高双字和低双字。则有：

~~~
TransSerialNumberHighDW(SerialNumber)=shld(UpperDW(SerialNumber),LowerDW(SerialNumber),1)（3）
TransSerialNumberLowDW(SerialNumber) = LowerDW(SerialNumber) + LowerDW(SerialNumber)；　　　 （4）
fs(SerialNumber) =   TransSerialNumberHighDW<< 32 | TransSerialNumberLowDW；　　　　　　　　　（5）
~~~

## Serial Number逆向变换算法描述

了解了fs的实现，我们接下来需要着手研究实现fs-1的思路。由于fs高双字和低双字分别由不同的方式变换的（3）、（4），所以我们需要分别求出SerialNumber的高低双字。得到Serial Number的低双字很简单，由（4）可知，我们只要将LowerDW(fp(ProductID))除以2就行。但需注意的是，整个双字值域中，LowerDW(fp(ProductID)) / 2 + (2<<31)同样能满足条件（由于溢出导致的相等），这点很重要！

再来考虑高位的算法。我们需要把HighDW(fp(ProductID))往右移一位，这是左边补入的一位可以任意。这时需要注意，考虑到shld性质，有以下两种情况：

1. 如果HighDW(fp(ProductID))最低位是1则需要LowerDW(SerialNumber)的最高位为1

2. 如果HighDW(fp(ProductID))最低位是0则需要LowerDW(SerialNumber)的最高位为0

考虑到LowerDW(fp(ProductID)) / 2和LowerDW(fp(ProductID)) / 2 + (2<<31)均可满足条件，若是情况1则选取后者，因为此时可保证LowerDW(SerialNumber)的最高位为1。同理，若是情况2则需选择前者。

该算法的C语言描述如下：

``` c SerialNumber逆变换算法
 void getSerialNumber(__int64 productId, char* buffer){
     DWORD upper = productId >> 0x20;
     DWORD lower = productId;
     assert((lower & 1) == 0); //lower = snLower + snLower因此lower必须为偶数
 
 
     DWORD snUpper = upper >> 1;
 
     DWORD snLower =0;
     if((upper & 1) == 1){
         snLower = lower / 2;
         snLower += (1<<31);
     }else{
         snLower = lower / 2;
     }
 
     DWORDLONG sn = snUpper;
     sn <<= 0x20;
     sn |= snLower;
 
     WORD *p = (WORD*)&sn;
     sprintf(buffer, "%04x-%04x-%04x-%04x", p[0], p[1], p[2], p[3]);
 }
```

## 小结

IDA很强大，Crack很费时间，收获的免费SRS使用权和投入的大量时间不成正比。结论：以后还是尽量少搞Crack吧。