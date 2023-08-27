# 为未公开API生成Lib并与MSBuild集成（一）
本文从Lib文件格式结构开始，描述Lib文件如何包含Dll的导出函数信息以及夹带Obj文件供链接，并从零构造Lib使得其包含任意未公开API导出函数信息。最后与[MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild)集成，衔接到Visual Studio C/C++项目中完成闭环。

注：本文“未公开API”均特指Windows SDK中连Lib文件也没有其导入符号的未公开API。

## 一、 问题背景
以我们熟悉的`CreateProcessInternalW`为例，它由`Kernel32.dll`导出。Windows SDK里没有它的声明，Kernel32.lib里也没有它的符号。如果想调用，一般使用动态加载或生成Stub Dll借用其Lib的方式：

### 1. 动态加载
按照函数定义，定义其指针类型（参数略），然后通过`LoadLibaray`+`GetProcAddress`之类的方式动态加载Dll和寻址：
    
```C
// typedef BOOL(WINAPI* PFNCreateProcessInternalW)(...);

NTSTATUS Status;
HMODULE Kernel32Handle;
PFNCreateProcessInternalW pfnCreateProcessInternalW;

Status = LdrLoadDll(NULL, NULL, &(UNICODE_STRING)RTL_CONSTANT_STRING(L"Kernel32.dll"), &Kernel32Handle);
if (NT_SUCCESS(Status))
{
    Status = LdrGetProcedureAddress(Kernel32Handle,
                                    &(ANSI_STRING)RTL_CONSTANT_STRING("CreateProcessInternalW"),
                                    0,
                                    &pfnCreateProcessInternalW);
    if (NT_SUCCESS(Status))
    {
        // pfnCreateProcessInternalW(...)
    }
    // LdrUnloadDll(Kernel32Handle);
}
```
> PS: 如果想像以上Demo代码一样在UM使用KM及未公开定义，可以关注[SystemInformer (ProcessHacker)](https://www.systeminformer.com/)的[phnt](https://github.com/winsiderss/systeminformer/tree/master/phnt)，我也积攒下了[Wintexports](https://github.com/KNSoft/Wintexports)自用。

显然，对于支持目标系统里必然存在的函数，远不及直接`CreateProcessInternalW(...)`调用方便，**调用未公开API的数量越多，动态加载的代码越冗杂繁复**，于是考虑下面的办法。

### 2. 生成Stub Dll借用其Lib  
编写Stub Dll，其具有除实现为空外完全相同的函数定义（参数略）：
```C
BOOL
WINAPI
CreateProcessInternalW(...)
{
    return FALSE;
}
```
注意，`CreateProcessInternalW`和绝大多数Windows API一样使用stdcall调用约定，x86下Lib的导入符号名称与Dll导出函数名称不一样，前者带有调用约定的修饰而后者没有，要用[模块定义文件 (.def)](https://learn.microsoft.com/en-us/cpp/build/reference/module-definition-dot-def-files)定义其导出：
```
LIBRARY KERNEL32
EXPORTS
    CreateProcessInternalW
```
最后编译Stub Dll，补充函数定义，取其一同生成的Lib，便能直接`CreateProcessInternalW(...)`这般调用了。
    
一眼看去，似乎仍有改进的空间，重点在于Lib的生成：
1. 首先，理论上我们只需要Dll名称和API名称（包含它的修饰名，x86的stdcall要用），空函数体以及[模块定义文件 (.def)](https://learn.microsoft.com/en-us/cpp/build/reference/module-definition-dot-def-files)是信息冗余的；
2. 其次，理论上没有任何函数实现，完全不需要在生成过程中将编译器卷进来浪费时间；
3. 最次，Lib格式的本质是档案，里面可以花式夹带Dll导出符号和目标文件。可以根据需要，同时包含多个Dll的导出符号，或者一对一地为每个未公开API所在Dll生成一个Lib。

这些“改进”太微不足道，但有时候我就这么点追求。先参考一些已有的工具：

+ [ReactOS](https://reactos.org/)的[spec2def](https://github.com/reactos/reactos/blob/master/sdk/tools/spec2def/spec2def.c)生成工具  
    它根据自定的spec文件描述函数原型生成Lib。spec比起手写空函数体加配套[模块定义文件 (.def)](https://learn.microsoft.com/en-us/cpp/build/reference/module-definition-dot-def-files)要简单的多：
    ```
    @ stdcall CreateProcessInternalW(ptr wstr wstr ptr ptr long long ptr wstr ptr ptr long)
    ```
    据此生成汇编源文件再生成Stub Dll+Lib。卷入汇编器总比卷入C编译器生成得快。
        
+ [MASM32](https://www.masm32.com/)的inc2l工具  
    它根据汇编inc文件里的过程声明生成汇编源文件然后生成Stub Dll+Lib。

+ [MSVC 库管理器 (Lib.exe)](https://learn.microsoft.com/en-us/cpp/build/reference/lib-reference)  
    它虽然可以根据[模块定义文件 (.def)](https://learn.microsoft.com/en-us/cpp/build/reference/module-definition-dot-def-files)直接生成Lib，但其生成导入符号的名称类型总指定为`IMPORT_OBJECT_NAME`，即表示导入名称与导入符号名称完全相同，不适用于x86下stdcall调用约定的API（此场景下应用`IMPORT_OBJECT_NAME_UNDECORATE`，表示导入名称为导入符号名称去除调用约定的修饰后的结果）。

比起以上已有的轮子还能进一步卷的，是做到能用[spec2def](https://github.com/reactos/reactos/blob/master/sdk/tools/spec2def/spec2def.c)这样简洁的函数定义，还能不卷入汇编器/编译器这类代码生成工具，甚至还能实现一个Lib包含多个Dll的导出符号。不妨从文件结构开始，从零构建Lib。

## 二、 生成包含Dll导出符号的Lib

### 1. Lib文件结构

先祭出Microsoft Learn的祖传文档——[PE Format - Archive (Library) File Format - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#archive-library-file-format)。

我已将踩过的坑对应的文档错漏提交给[Microsoft Docs](https://github.com/MicrosoftDocs)并得到合入([PR #1555](https://github.com/MicrosoftDocs/win32/pull/1555), [PR #1586](https://github.com/MicrosoftDocs/win32/pull/1586), [PR #1622](https://github.com/MicrosoftDocs/win32/pull/1622))，此外需要注意的本文会提及。与某些单纯复制或者翻译微软文档就拿来发的博客相比，算是实践出真知了。

**Lib (Archive) 文件主要包含导入符号（Import Symbol），供链接器链接。这些符号可以指向外部Dll的导出，也可以存在于Lib自身夹带的Obj文件里。**

整体布局比较简单。开头固定的文件签名后，一系列成员（Archive Member）依次排列，每个成员都以成员头(`IMAGE_ARCHIVE_MEMBER_HEADER`)开始，后跟成员数据。

<table>
    <tr>
        <th colspan="2">成员</th>
        <th>内容</th>
        <th>备注</th>
    </tr>
    <tr>
        <td colspan="2">Signature</td>
        <td><code>"!&lt;arch&gt;\n"</code> (<code>IMAGE_ARCHIVE_START)</code></td>
        <td>固定文件签名8 (<code>IMAGE_ARCHIVE_START_SIZE</code>)字节大小</td>
    </tr>
    <tr>
        <td rowspan="10">1st Linker Member</td>
        <td>Archive Member Header</td>
        <td>Name: <code>"/"</code> (<code>IMAGE_ARCHIVE_LINKER_MEMBER</code>)</td>
        <td></td>
    </tr>
    <tr>
        <td>Number of Symbols</td>
        <td>符号（名称）总数n</td>
         <td></td>
    </tr>
    <tr>
        <td rowspan="4">Offsets</td>
        <td>符号1偏移</td>
        <td rowspan="4">按偏移升序排序。多符号名指向同一符号时，这里相邻的偏移才会相等</td>
    </tr>
    <tr>
        <td>符号2偏移</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td>符号n偏移</td>
    </tr>
    <tr>
        <td rowspan="4">String Table</td>
        <td>符号1名称</td>
        <td rowspan="4">各字符串以<code>'\0'</code> (<code>ANSI_NULL</code>)结尾。这里的名称与Offsets的符号偏移按顺序一一对应</td>
    </tr>
    <tr>
        <td>符号2名称</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td>符号n名称</td>
    </tr>
    <tr>
        <td rowspan="15">2nd Linker Member</td>
        <td>Archive Member Header</td>
        <td>Name: <code>"/"</code> (<code>IMAGE_ARCHIVE_LINKER_MEMBER</code>)</td>
        <td></td>
    </tr>
    <tr>
        <td>Number of Members</td>
        <td>符号成员总数m</td>
        <td>与符号总数n不同，这里的m是符号成员总数。
如果有x个符号名称与其它符号名称指向了同一个符号成员，则m=n-x。</td>
    </tr>
    <tr>
        <td rowspan="4">Offsets</td>
        <td>符号成员1偏移</td>
    </tr>
    <tr>
        <td>符号成员2偏移</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td>符号成员m偏移</td>
    </tr>
    <tr>
        <td>Number of Symbols</td>
        <td>符号（名称）总数n</td>
        <td>按符号名称排序</td>
    </tr>
    <tr>
        <td rowspan="4">Indices</td>
        <td>符号1对应符号成员在Offsets中的序号</td>
    </tr>
    <tr>
        <td>符号2对应符号成员在Offsets中的序号</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td>符号n对应符号成员在Offsets中的序号</td>
    </tr>
    <tr>
        <td rowspan="4">String Table</td>
        <td>符号1名称</td>
        <td rowspan="4">各字符串以<code>'\0'</code> (<code>ANSI_NULL</code>)结尾。
这里的名称与Indices的序号按顺序一一对应</td>
    </tr>
    <tr>
        <td>符号2名称</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td>符号n名称</td>
    </tr>
    <tr>
        <td rowspan="4">Longnames Member</td>
        <td>Archive Member Header</td>
        <td>Name: <code>"//"</code> (<code>IMAGE_ARCHIVE_LONGNAMES_MEMBER</code>)</td>
        <td rowspan="4">可选，用于存放名称过长成员的名称。
各字符串以<code>'\0'</code> (<code>ANSI_NULL</code>)结尾</td>
    </tr>
    <tr>
        <td rowspan="3">String Table</td>
        <td>长成员名称1</td>
    </tr>
    <tr>
        <td>长成员名称2</td>
    </tr>
    <tr>
        <td>...</td>
    </tr>
    <tr>
        <td rowspan="2">Symbol 1</td>
        <td>Archive Member Header</td>
        <td></td>
        <td rowspan="5">见表格外的单独说明</td>
    </tr>
    <tr>
        <td colspan="2">Obj文件内容<br/>或<br/>Import Header+导入符号名字符串+Dll名称字符串</td>
        </td>
    </tr>
    <tr>
        <td>Symbol 2</td>
        <td colspan="2">...</td>
    </tr>
    <tr>
        <td>...</td>
        <td colspan="2">...</td>
    </tr>
    <tr>
        <td>Symbol m</td>
        <td colspan="2">...</td>
    </tr>
</table>

+ Archive Member Header (成员头)  
    Lib文件中所有成员都以`IMAGE_ARCHIVE_MEMBER_HEADER`结构体开头，实际有用且需要的只有`Name`和`Size`字段，分别表示名称和大小。`Name`也用以指示为上表签名后的三个特殊成员，如果名称不够16字节，剩余的用空格`' '`补齐，如果超过16字节（去除末尾固定的`'/'`只剩15字节），则名称字符串存放到 Longnames Member （长名称成员）里，`Name`字段指示其所在偏移。`Size`表示成员数据大小，不包含成员头自身大小60（`IMAGE_SIZEOF_ARCHIVE_MEMBER_HDR`或`sizeof(IMAGE_ARCHIVE_MEMBER_HEADER)`）字节，也不包含用于对齐的`'\n'`。
    
+ 1st Linker Member 与 2nd Linker Member  
    作用和结构都类似，记录符号的名称和所在文件偏移供链接器定位，信息上是冗余的。前者Unix Style，后者MS Style，并且后者按名称排序使得按名称查找符号更高效，MS链接器（[Link.exe](https://learn.microsoft.com/en-us/cpp/build/reference/linking)）使用。

+ Longnames Member  
    后续各符号的成员名如果超长，则名称形式不再是`"xxxx/"`（16字节放不下），而是`"/n"`（`n`即为该名称字符串所在Longnames Member内容中的偏移）。

+ Symbol  
    Lib包含的各个符号实际内容。前面说过，可以是指向外部Dll的导出，也可以是夹带的Obj。

    + 指向外部Dll的导出  
        成员名称为Dll名称，如`"KERNEL32.dll"`，内容为Import Header+导入符号名字符串+Dll名称字符串。虽然Dll名称重复出现，但后续MS链接器生成导入表时会分别使用，这两处不同或者有误会导致生成的导入表奇怪或不正确。要注意调用约定、Dll导出名、导入符号名的关系，内容示例：  

        | 平台 | DLL导出名 | Import Header | 导入符号名
        | --- | --- | --- |  --- |
        | x86 | CreateProcessInternalW | Type=`IMPORT_OBJECT_CODE`<br/>Name Type=`IMPORT_OBJECT_NAME_UNDECORATE` | _CreateProcessInternalW@48 |
        | x64 | CreateProcessInternalW | Type=`IMPORT_OBJECT_CODE`<br/>Name Type=`IMPORT_OBJECT_NAME` | _CreateProcessInternalW |

        此外，还要同时导出带"\_\_imp_"前缀的符号供链接器使用，避免调用外部Dll导出时生成不必要的Jmp Stub指令。即`_CreateProcessInternalW@48`与`__imp__CreateProcessInternalW@48`都指向同个Import Header。

    + 夹带Obj  
    成员名称为Obj在Lib中的全路径，如果在根目录，等同于Obj文件名。内容直接为Obj文件内容。如果多个Lib导入符号都在同一个夹带的Obj内，则都指向同个Symbol，链接器会在这个Symbol夹带的Obj里寻找它们。

+ 其它注意事项  
    + 每个成员（Archive Member Header）所在偏移要2字节对齐，如果成员大小为奇数，则在其后插入`'\n'`(`IMAGE_ARCHIVE_PAD`)，使紧随其后的成员所在偏移得以对齐。插入的`'\n'`不计入成员大小。

    + 数值有的用十进制字符串表示，有的用大端存储，MS Style的2nd Linker Member里各数值用小端存储，要看清文档说明。

### 2. 用于链接Dll的Lib
按照上述基础的Lib文件结构，构造包含Dll导出符号（如`_CreateProcessInternalW@48`与`__imp__CreateProcessInternalW@48`）的Lib可供链接器链接，但MS链接器（[Link.exe](https://learn.microsoft.com/en-us/cpp/build/reference/linking)）生成的二进制导入表却缺失了对应Dll的IID（`IMAGE_IMPORT_DESCRIPTOR`）。

想必从Windows SDK里最小的Dll导入Lib找线索更容易。用[7-Zip](https://7-zip.org/)打开SAS.lib里的1.txt（也就是1st Linker Member），看到其包含符号如下：
```
1.SAS.dll    __IMPORT_DESCRIPTOR_SAS
2.SAS.dll    __NULL_IMPORT_DESCRIPTOR
3.SAS.dll    SAS_NULL_THUNK_DATA
4.SAS.dll    _SendSAS@4
4.SAS.dll    __imp__SendSAS@4
```
可见，除了包含Dll导出符号（`_SendSAS@4`与`__imp__SendSAS@4`），还有：
1. Dll的IID符号（`__IMPORT_DESCRIPTOR_SAS`）
2. Dll的空Thunk（`SAS_NULL_THUNK_DATA`）
3. 导入表末尾结束的空IID（`__NULL_IMPORT_DESCRIPTOR`）

它们分别位于3个不同夹带的Obj里，MS链接器（[Link.exe](https://learn.microsoft.com/en-us/cpp/build/reference/linking)）构造导入表时使用它们。

在一个用于链接Dll的Lib里`__NULL_IMPORT_DESCRIPTOR`只需要有1个，所以存放在单独的一个Obj会比较好。成员名无所谓，不用保留名称即可。

前二者名称分别为`"__IMPORT_DESCRIPTOR_[Dll名]"`和`"[Dll名]_NULL_THUNK_DATA"`（**注意开头字符为`'\x7F'`**），可以放在同个Obj里。如果Lib包含多个Dll的导出符号，它们也要对应地有多个，成员名也是Dll的名称。所在Obj需要有`__NULL_IMPORT_DESCRIPTOR`的外部引用记录，否则链接Dll时`__NULL_IMPORT_DESCRIPTOR`可能未被主动引入，导致的PE映像导入表没能以空IID结尾。

与Windows SDK的Lib里的Obj相比，我们可以去除其中的调试符号，也可以简化3个Obj之间的复杂关系。

### 3. 代码实现
我的代码实现：

[KNSoft/C4Lib - C4Lib/Source/PEImage/ObjectFile.cs](https://github.com/KNSoft/C4Lib/blob/main/Source/PEImage/ObjectFile.cs)中`NewNullIIDObject`方法用于构造`__NULL_IMPORT_DESCRIPTOR`所在Obj，`NewDllImportStubObject`方法用于构造`__IMPORT_DESCRIPTOR_XXX`与`XXX_NULL_THUNK_DATA`所在Obj。

[KNSoft/C4Lib - C4Lib/Source/PEImage/ArchiveFile.cs](https://github.com/KNSoft/C4Lib/blob/main/Source/PEImage/ArchiveFile.cs)是Lib构造的核心实现，调用`AddImport`方法（包含多个重载）为Lib文件添加Object文件或Dll导出的导入项。

[KNSoft/Precomp4C - Precomp4C/Source/Tasks/DllStubTask.cs](https://github.com/KNSoft/Precomp4C/blob/main/Source/Tasks/DllStubTask.cs)是MSBuild自定义生成任务，调用上述方法，根据输入的XML描述，输出Lib：
```XML
<DllStub>
	<Dll Name="KERNEL32.dll">
		<Export Name="CreateProcessInternalW" CallConv="__stdcall" Arg="ptr ptr ptr ptr ptr long long ptr ptr ptr ptr ptr" />
	</Dll>
	<Dll Name="SECHOST.dll">
		<Export Name="LsaLookupOpenLocalPolicy" CallConv="__stdcall" Arg="ptr long ptr" />
        <!-- ... -->
    </Dll>
</DllStub>
```

使用[7-Zip](https://7-zip.org/)查看产出Lib的1.txt，内容如：
```
KNSoft    __NULL_IMPORT_DESCRIPTOR
1.KERNEL32.dll    __IMPORT_DESCRIPTOR_KERNEL32
1.KERNEL32.dll    KERNEL32_NULL_THUNK_DATA
2.KERNEL32.dll    _CreateProcessInternalW@48
2.KERNEL32.dll    __imp__CreateProcessInternalW@48
1.SECHOST.dll    __IMPORT_DESCRIPTOR_SECHOST
1.SECHOST.dll    SECHOST_NULL_THUNK_DATA
2.SECHOST.dll    _LsaLookupOpenLocalPolicy@12
2.SECHOST.dll    __imp__LsaLookupOpenLocalPolicy@12
...
```
可以看到它同时包含了`KERNEL32.dll`与`SECHOST.dll`的导出，`__NULL_IMPORT_DESCRIPTOR`只有一份，`__IMPORT_DESCRIPTOR_XXX`与`XXX_NULL_THUNK_DATA`归并到了一个Obj里，并自动生成了带`__imp_`开头的别名符号，已可正常链接，功德+1。

要功德圆满，功能上就要形成闭环。我能想到好的方式是与[MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild)集成，这也是使用C#实现的主要因素。

也许我们天天都在使用[MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild)这个强大的生成平台，却对它了解甚少，如果稍加了解，
+ 至少生成脚本里可以使用MSBuild命令代替VS的devenv，得到更即时而优雅的生成输出；
+ 至少可以添加个[Directory.Build.props](https://learn.microsoft.com/en-us/visualstudio/msbuild/customize-by-directory)，省去挨个项目调整配置的繁琐；
+ 至少不用尴尬地将Copy命令或者Zip打包命令写在“Pre-Build Event”里；
+ 至少可以为插入的外部生成步骤指定输入和输出，从[MSBuild 增量生成](https://learn.microsoft.com/en-us/visualstudio/msbuild/incremental-builds)功能中收益；
+ ...

我想，它值得独占一篇。接下来的第二篇（未完待续）将以自定义代码生成任务为起点，了解[MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild)，将类似spec文件的Dll导出信息通过自定义代码生成任务编译生成Lib，进而可与Visual Studio C/C++项目无缝衔接。

<br/>

***

文档参考：  
[PE Format - Archive (Library) File Format - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#archive-library-file-format)

<br/>

如有谬误，恳请指正。本文与以上实现也将随以下项目不断更新：
+ [KNSoft/C4Lib](https://github.com/KNSoft/C4Lib)：自用的C#函数库，包含本文Lib生成的代码实现（[C4Lib/Source/PEImage/DllStub.cs](https://github.com/KNSoft/C4Lib/blob/main/Source/PEImage/DllStub.cs)）
+ [KNSoft/Precomp4C](https://github.com/KNSoft/Precomp4C)：[MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild)自定义生成工具，可通过引入nuget包，直接将项目中描述Dll导出符号的XML编译为Lib，进而参与解决方案中其它项目的生成

<br/>

***

原文链接：[为未公开API生成Lib并与MSBuild集成（一）](https://github.com/KNSoft/Blog/tree/main/Windows编程/为未公开API生成Lib并与MSBuild集成（一）)

<img src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png"/><br/>
本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议 (CC BY-NC-SA 4.0)](http://creativecommons.org/licenses/by-nc-sa/4.0/) 进行许可。  
<br/>
[Ratin](https://github.com/RatinCN) <[ratin@knsoft.org](mailto:ratin@knsoft.org)>  
国家认证 系统架构设计师  
ReactOS Contributor