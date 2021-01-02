# 2.1 snpslmd2018.06-SP1的破解

本节介绍的内容，反汇编的信息，全部来自于64位linux的snpslmd2018.06-sp1。

## sssverify的几个patch点

### ECC Patch
和其他的ECC patch一样，打的补丁是l_pubkey_verify函数，直接返回0。

### 嵌入计算vendor string的计算
在**sclSSScodeDecrypt()**函数后面，增加**sclSSScodeEncrypt()**函数，计算出正确的vendor string。

```assembly
46659d:	e8 3e 83 02 00       	callq  48e8e0 <YhFoUkpVwniaQbq+0x28a0>
4665a2:	48 89 c7             	mov    %rax,%rdi
4665a5:	49 89 c4             	mov    %rax,%r12
4665a8:	e8 b3 48 00 00       	callq  46ae60 <sclSSScodeDelete+0x220>  // sclSSScodeDecrypt
4665ad:	4c 89 e7             	mov    %r12,%rdi					// sssCodeStrc* a1
4665b0:	48 89 c6             	mov    %rax,%rsi					// sssDataStrc* a2
4665b3:	e8 58 5b 00 00       	callq  46c110 <sclSSScodeDelete+0x14d0> // sclSSScodeEncrypt
```


需要注意的是，这里的vendor string计算中，只能计算出非RSA的部分，RSA的部分还需要进一步研究。从debug跟踪情况来看，RSA需要用到的私钥，在客户端并没有出现，因此即使需要也只能做公钥替换。

### Encrypt函数的修改
sclSSScodeEncrypt函数里面也需要有一个修改:

```cpp
if ( is_callhome_disabled(100LL, v10, v28 >> 16, v9, v6, v7) )
    {
      v8[22] = 1;
      LOBYTE(v12) = 23;
	}
```

这个地方，在vendor string包含RSA部分的时候，默认是不进入的，48位的vendor string是需要让他进入，否则回出现2016以后的版本，stack error的问题，license启动和checkout倒是可以正常进行。（这里需要说明，EFA的sssverify这里胡乱填进了一个非0的数据，使得其license可以正常使用，但肯定不是真正encrypt的时候期望填入的）。

修改前的汇编代码如下：

```assembly
.text:000000000046C388                 call    _is_callhome_disabled
.text:000000000046C38D                 test    eax, eax
.text:000000000046C38F                 mov     edi, 16h
```

修改的方案，是用xor eax, eax; inc eax;两条语句来替代，使得test之前eax为1，call指令是5B，剩余的部分用nop指令填充。

### 终端输出
这部分的修改是紧接着sclSSScodeEncrypt()函数的插入的，主要是从code结构体中取出vendor string的部分，利用C库的puts函数完成输出，最后调用exit函数退出，免得修改后的程序崩溃。

```assembly
46659d:	e8 3e 83 02 00       	callq  48e8e0 <YhFoUkpVwniaQbq+0x28a0>
4665a2:	48 89 c7             	mov    %rax,%rdi
4665a5:	49 89 c4             	mov    %rax,%r12
4665a8:	e8 b3 48 00 00       	callq  46ae60 <sclSSScodeDelete+0x220>  // sclSSScodeDecrypt
4665ad:	4c 89 e7             	mov    %r12,%rdi					// sssCodeStrc* a1
4665b0:	48 89 c6             	mov    %rax,%rsi					// sssDataStrc* a2
4665b3:	e8 58 5b 00 00          callq  46c110 <sclSSScodeDelete+0x14d0> // sclSSScodeEncrypt
4665b8:  49 8b 7c 24 10         mov    0x10(%r12),%rdi
4665bd:  e8 ae bc ff ff			callq  ostrPtr                           // 0x462270 
4665c2:  48 89 c7               mov    %rax,%rdi
4665c5:  e8 a6 86 fa ff         callq  _puts					     // 0x40EC70
4665ca:  bf 00 00 00 00         mov    $0x0,%edi
4665cf:  e8 7c 83 fa ff         callq  _exit                             // 0x40e950

```

### 文件校验补丁
加入ECC以后的synopsys程序，增加了文件校验补丁，如果出现了二进制修改，会输出一个内部错误的打印。注意这个打印是不能直接从可执行文件中搜到的。Synopsys的高级字符串控制，都是在程序中存储一个和0x97进行异或后的字符串，使用的时候，直接将字符串的每一个字符作为参数压入，调用scl_xfeature（）函数现场解析。

如果文件名没有被混淆，W4Pgi9QuIQZP86B（）函数是文件校验及返回的关键函数。关键代码片段如下所示。

蓝色字部分的函数就是比较两个校验字符串是否相等，如果if分支被触发了，里面的内容就是错误代码的记录。其中红色字体的两行代码，都可以帮助从混淆名字的汇编代码中定位得到这个函数，找到这个破解点。这个地方是2019年版本的破解关键所在。X64的汇编代码，函数调用的时候，第一个参数使用%rdi寄存器传递，第二个参数使用%rsi寄存器传递，因此这种函数调用的时候，某一个参数是常数的汇编指令，出现的概率不会太多，简单排除一下就可以完成函数的定位。

最简单的修改方案，就是在tddf1eNvOdo2Mjw()函数调用的地方去掉函数调用，换乘xor $eax, $eax，然后nop补齐；高级一些的改法，是去弄明白两个字符串的计算原理，修改后人家也不能再改你的可执行文件了。

```cpp
if ( v28 && *(_QWORD *)(v28 + 8) )
{
	v45 = ostrNew();
	v31 = ostrNew();
	v41 = ostrNew();
	if ( v34 == 2 )
	{
		strncpy(&v68, ptr + 4, 0x200uLL);// found pattern here from the constant 0x200
		ostrSets(v41, &v68);
	}
	else if ( v34 == 1 )
	{
		ostrSets(v41, ptr + 4);
	}
	ostrSets(v31, v11[1]);
	sub_46D7E0(v31, v41, v45);
	ostrDelete(v31);
	ostrDelete(v41);
	if ( (unsigned int)tddf1eNvOdo2Mjw(v45, v44) )
	{
		*v2 = 0;
		*v3 = 1;
		ostrSets(*(_QWORD *)(v4 + 104), &v47);
		sub_473100(v4, 14LL);
	}

```

### 如何定位关键函数

本节讲述怎么从一个新的sssverify中找到关键函数所在。
可执行程序sssverify里面有下面的一个函数，用于解密sss信息
```cpp
sssDataStrc *__fastcall sclSSScodeDecrypt(sssCodeStrc *a1)
```

一般来说这个函数的名字会被模糊掉，如何可以找出来呢？这个函数的一个参考C反编译函数参照附件。
```cpp
sprintf(s, "0x%08x", 86400 * (a1->intStartDate / 86400));
```

这个函数里面有除法，是一个很典型的汇编运算。


