# Win32应用程序DPI自适应的设计与实现
随着便携设备和高DPI显示器的普及，Windows自身对高DPI的支持也在不断更迭。对应用程序而言，如何与操作系统配合，实现DPI适配以达到最佳的显示效果愈发重要，我们平日常用的应用软件也陆续支持了DPI适配。本文将从概念原理、Windows DPI支持发展开始描述，并给出DPI适配方案的设计与实现（[NTAssassin](https://github.com/KNSoft/NTAssassin)），最后演示实现案例（[AlleyWind](https://github.com/KNSoft/AlleyWind)）效果。

<br/>
<br/>

## 一、 问题背景
假设在一个19英寸的显示器上显示10像素x20像素的文字。若分辨率为1920x1080尚可正常阅读，但如果这是3840x2160分辨率的4K高分屏，便小到难以看清，相当于只有原来的1/4。此时我们则使用DPI来进行缩放调节，将该文字调整到对应高分辨率下的大小进行显示，如20像素x40像素，获得合适的视觉效果。

DPI（Dots Per Inch，每英寸点数）用于描述图像每英寸中含有多少像素点，故等大的图像，DPI越高，包含的像素点数越多，图像内容越丰富，看起来便越清晰。

在Windows下，屏幕单位长度中有多少像素点由实际屏幕物理大小与分辨率决定，而常说的DPI则成了缩放的参考值。如默认DPI（系统缩放设置为100%）为常量96，系统缩放设置为125%时对应DPI为96x125%=120。系统本身与应用程序均以此DPI作参照，实现缩放。

<div align="center">
    <img src="Asset/System DPI Setting.png"/>
    <div>Win10中缩放相关设置</div>
</div>
<br/>

了解背景以后，看似只要将各长度对应DPI进行等比缩放就能实现适配，比如原本需要绘制100x200单位矩形，在125%缩放比例下绘制125x250单位即可，并且系统已经有内置的缩放机制供默认使用。然而实际情况往往复杂得多，从Windows对DPI支持不断的更迭便可看出，主要有以下问题：

+ 系统缩放容易导致应用程序模糊
  
    若使用系统自带的DPI适配（DWM Scaling），应用程序一般无需做任何更改。系统始终告知应用程序DPI设置为默认值96（100%），但应用程序进行显示相关操作时，系统内部将为其自动应用DPI缩放。如同位图缩放，这样的缩放容易导致应用程序显示模糊，尤其是字体。下图是MSDN中对比示意：

    <div align="center">
        <img src="Asset/Mixed-Mode DPI Scaling.jpg"/>
        <div>混合模式下应用程序PMv2缩放与系统缩放对比</div>
    </div>
    <br/>

    以下是AlleyWind应用程序系统缩放（模糊）与应用程序适配DPI缩放（不模糊）的对比，系统缩放明显模糊、浑厚，应用程序缩放则显得清晰、锐利：

    <div align="center">
        <img src="Asset/AlleyWind System Scaling.png"/>
        <div>系统缩放（125%）</div>
    </div>
    <br/>

    <div align="center">
        <img src="Asset/AlleyWind Application Scaling.png"/>
        <div>应用程序适配DPI缩放（125%）</div>
    </div>
    <br/>

+ 若应用程序自己控制缩放，界面编程繁复

    需要应用程序在进行绘制时，自己将所有要绘制的内容全部按DPI进行缩放，依赖代码进行计算、响应和控制等等，实现有一定复杂度。

+ 需要响应DPI变化，实时调节缩放

    目前绝大多数支持DPI适配的常用软件，在DPI更改时都能正确调整布局，不需重新启动才生效。随着高DPI显示器的普及，这样的场景不再仅仅局限于用户手动更改系统DPI设置这样少见甚至不必考虑的场合——窗口游走于不同DPI显示器之间也会出现。所以比起在开局一次性缩放，如今更需要设计成能支持灵活响应DPI变化的模式。

<br/>
<br/>

## 二、Windows的支持
+ Windows XP

    早在Windows XP，便提供了缩放设置选项供用户设置。应用程序可以通过API获取当前DPI，补足系统缩放的缺陷。

    <div align="center">
        <img src="Asset/Windows XP DPI Setting.png"/>
        <div>Windows XP中DPI设置</div>
    </div>
    <br/>

    使用[GetDeviceCaps](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-getdevicecaps)获取传入DC的DPI设置，`LOGPIXELSX`与`LOGPIXELSY`对应X与Y方向的DPI，目前二者始终相等，默认值（缩放为100%）是96（Windows SDK里定义为`USER_DEFAULT_SCREEN_DPI`）。

    此时，面临上述三个问题，并且系统缩放效果不佳。

+ Windows Vista与Windows 7

    随着DWM的引入，应用程序的系统缩放由其实现（DWM Scaling），针对图像进行缩放的效果比起XP针对部分元素的缩放好了很多。并且开始增加相关DPI函数，供应用程序选择自己实现缩放还是系统代为缩放。

    可以通过清单文件或者新增的[SetProcessDPIAware](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setprocessdpiaware)函数设置应用程序DPI缩放模式，告诉系统本程序已考虑了DPI缩放问题，不要应用系统缩放。增加[IsProcessDPIAware](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-isprocessdpiaware)函数以获取此项设置。

    Windows 7开始，DPI设置成为了每用户独立的设置。至此，虽然仍面临上述三个问题，但系统缩放效果得以改善，并且应用程序可以选择使用系统缩放还是自己实现。

+ Windows 8.1

    支持Per-Monitor (PM/PMv1) DPI缩放模式，不同显示器可以拥有不同DPI，新增[GetDpiForMonitor](https://learn.microsoft.com/en-us/windows/win32/api/shellscalingapi/nf-shellscalingapi-getdpiformonitor)函数获取特定显示器的DPI设置，同时相关DPI变更的消息机制也应运而生。
    
    由于多了DPI模式，原[SetProcessDPIAware](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setprocessdpiaware)和[IsProcessDPIAware](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-isprocessdpiaware)的`TRUE`和`FALSE`已无法准确指明，可以使用清单文件或者新增的[SetProcessDpiAwareness](https://learn.microsoft.com/en-us/windows/win32/api/shellscalingapi/nf-shellscalingapi-setprocessdpiawareness)函数设置应用程序DPI缩放模式，告诉系统本程序使用含Per-Monitor在内的哪种DPI缩放模式。Per-Monitor模式与之前相同，系统不再为应用程序进行缩放，但当DPI变更时，操作系统会通知应用程序，让应用程序适时调整，实现窗口在不同DPI下来去自如。新增的[GetProcessDpiAwareness](https://learn.microsoft.com/en-us/windows/win32/api/shellscalingapi/nf-shellscalingapi-getprocessdpiawareness)函数可以获取此项设置。</p>
    
    除了新增上述函数支持Per-Monitor模式的设置，还新增了窗口消息以落实该机制。当顶层窗口DPI变更时，如用户修改了DPI设置，或窗口移动到了不同DPI的显示器上，操作系统会为其下发[WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)消息，告诉窗口新的DPI与建议调整的位置和大小。看似提供了近乎保姆级别支持，但该消息只为顶层窗口派发，顶层窗口内部仍需自己进行计算和缩放子窗口。

    至此，上述问题中DPI变化的场景得以有机会优雅地面对，多显示器不同DPI的场景需要得以满足。

+ Windows 10 1607

    DPI缩放模式设置从进程独立变为线程独立，从此同一个进程的不同线程可以拥有不同的DPI缩放模式。看似能让应用程序更自由地控制DPI缩放，但随之而来的又是一对新API：[SetThreadDpiAwarenessContext](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setthreaddpiawarenesscontext)与[GetThreadDpiAwarenessContext](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getthreaddpiawarenesscontext)。此外还新加入了[GetAwarenessFromDpiAwarenessContext](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getawarenessfromdpiawarenesscontext)、[GetDpiForWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getdpiforwindow)、[GetDpiForSystem](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getdpiforsystem)、[GetSystemMetricsForDpi](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsystemmetricsfordpi)、[SystemParametersInfoForDpi](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfofordpi)等函数供应用程序使用，看上去似乎更便捷，但实用性和兼容问题还是要一同考虑的。还增加了[EnableNonClientDpiScaling](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enablenonclientdpiscaling)函数为顶层窗口提供非客户区DPI缩放支持，它很快便被即将到来的Per-Monitor v2模式吞并，生于Win10 1607，被抛弃于1703。
    
    此外，微软还为自家的UWP与WPF提供了原生支持。至此，相比Windows 8.1变更也不少，但未必可圈可点，甚至不乏鸡肋。可以结合实际情况考虑，沿用之前的模式。

+ Windows 10 1703

    带来Per-Monitor v2 (PMv2) DPI缩放模式，随系统层面全面完善对DPI适配：
    + 消息通知：DPI变更通知发送给整个窗口树，而之前的PMv1只发送给顶层窗口。还新增了[WM_DPICHANGED_BEFOREPARENT](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged-beforeparent)、[WM_DPICHANGED_AFTERPARENT](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged-afterparent)消息满足子窗口时序需要，新增[WM_GETDPISCALEDSIZE](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-getdpiscaledsize)消息满足非线性缩放需要。
    + 非客户区缩放：非客户区自动缩放，无需再调用[EnableNonClientDpiScaling](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enablenonclientdpiscaling)。菜单也会缩放。
    + Win32对话框及控件自动缩放
    + 主题资源显示自动缩放
    
    有了上述支持，经典Win32应用程序甚至只需在清单文件中声明或一行
    ```C
    SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
    ```
    设置DPI缩放模式为Per-Monitor v2，即可完美适配DPI。因为对话框及其菜单、内部的Win32通用控件、主题资源等等都已在系统层面自动响应DPI缩放，无需应用程序操心。若实在效果不理想，也有一同到来的新API[SetDialogDpiChangeBehavior](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setdialogdpichangebehavior)、[SetDialogControlDpiChangeBehavior](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setdialogcontroldpichangebehavior)可以修改针对Win32对话框或控件的默认DPI缩放行为，以备不时之需。随着Win32通用控件对高DPI的适配，Windows Forms也在1703中提供了有限的支持。

    此外，引入了新的设置DPI缩放模式API [SetProcessDpiAwarenessContext](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setprocessdpiawarenesscontext)，与1607新增的DPI模式相关API共用相同一套枚举值，命名风格也相似。
            
    至此，Windows基本完善对高DPI的支持，此版本具有重大意义。纵向从系统层面自低而上地解决了上述各问题，满足了相关使用场景，传统Win32控件更是从中受益，得以原生支持DPI适配。横向上完成了对传统Win32控件、UWP、WPF、Windows Forms、Direct2D等框架对DPI的支持，应用程序生态环境也基本完善。

+ Windows 10 1803与1809

    1703引入全新模式一举完善DPI的支持之后，1803与1809更多的是修订与补充，这里归并到一起简要描述。
    
    1803主要引入新的API让同一线程的不同窗口可以具有不同DPI缩放模式，相关函数有如[SetThreadDpiHostingBehavior](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setthreaddpihostingbehavior)、[GetThreadDpiHostingBehavior](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getthreaddpihostingbehavior)、[GetWindowDpiHostingBehavior](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getwindowdpihostingbehavior)。
    
    1809主要增加DPI_AWARENESS_CONTEXT_UNAWARE_GDISCALED模式，该模式从GDI层面改善不支持DPI适配的应用程序系统缩放效果，对应应用程序兼容性的“系统（增强）”。比起DWM Scaling产生的位图缩放，GDI层面的缩放效果可与应用程序自己的缩放一样美观。

<br/>
<br/>

 ## 三、DPI适配方案的设计与实现
直到Win10 1703系统才支持Win32应用程序完美自适应高DPI，时至今日，我们往往要考虑兼容较老的系统，直到Win7。我们不妨从Win8.1前后两个角度来考虑，Win8.1之前DPI设置无法独立到单显示器，所以也可不考虑DPI变化的场景。若必须考虑用户变更DPI，可以自行增加轮询机制。Win8.1开始，DPI支持针对到单个显示器，但系统也提供了[WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)消息通知应用程序，我们只需要注意在收到这个消息时进行DPI缩放即可。

简而言之，我们的应用程序使用Per-Monitor (v1) DPI缩放模式，在程序初始化与收到[WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)消息时进行DPI缩放操作，即可在最大程度上兼容各个操作系统。

剩下的问题便是DPI缩放逻辑，主要包含窗口缩放、字体适配、图像资源缩放三个方面。

首先我们需要根据窗口获取DPI，得知应该缩放多少倍。以下是NTAssassin中的实现：</p>
```C
BOOL NTAPI DPI_FromWindow(HWND Window, _Out_ PUINT DPIX, _Out_ PUINT DPIY)
{
    PKUSER_SHARED_DATA pKUSD = SharedUserData;
    if (pKUSD->NtMajorVersion > 6 || (pKUSD->NtMajorVersion == 6 && pKUSD->NtMinorVersion >= 3)) {
        if (!pfnGetDpiForMonitor) {
            pfnGetDpiForMonitor = (PFNGetDpiForMonitor)Sys_LoadAPI(SysDllNameShcore, "GetDpiForMonitor");
        }
        if (pfnGetDpiForMonitor) {
            HMONITOR hMon = MonitorFromWindow(Window, MONITOR_DEFAULTTONULL);
            if (hMon && pfnGetDpiForMonitor(hMon, MDT_EFFECTIVE_DPI, DPIX, DPIY) == S_OK) {
                return TRUE;
            }
        }
    }
    HDC hDC;
    hDC = GetDC(Window);
    *DPIX = GetDeviceCaps(hDC, LOGPIXELSX);
    *DPIY = GetDeviceCaps(hDC, LOGPIXELSY);
    ReleaseDC(Window, hDC);
    return FALSE;
}
```
这里在NT6.3（Win8.1）及以上系统中优先考虑Per-Monitor模式，此模式下应调用 [GetDpiForMonitor](https://learn.microsoft.com/en-us/windows/win32/api/shellscalingapi/nf-shellscalingapi-getdpiformonitor) 获取窗口所在显示器的DPI，其返回结果会随DPI设置的变化而变化。传统的 [GetDeviceCaps](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-getdevicecaps) 作为备选方案，它的返回值不随DPI设置相应变化。其次，X与Y始终相同，可不像以上二者兼顾。

接下来需要缩放窗口。最直接的DPI缩放逻辑为下图所示：

<div align="center">
    <img src="Asset/DPI Scaling Algorithm 1.png"/>
</div>
<br/>

计算公式为 `缩放后的值 = 初始值 × DPI` 。如某长度为100单位，在150%缩放比例下新长度应为100 × 150% = 150。

虽然简单，但问题也很明显——在DPI变化的场景下，初始大小难以再直接获取。当然可以通过DPI的变化进行换算，此时与下面介绍的逻辑殊途同归：

<div align="center">
    <img src="Asset/DPI Scaling Algorithm 2.png"/>
</div>
<br/>

计算公式为 `缩放后的值 = 旧值 × 新DPI ÷ 旧DPI`。如某长度为100单位，缩放比例由100%变为150%时，新长度应为100 × 150% ÷ 100% = 150。NTAssassin的实现如下：
```C
VOID NTAPI DPI_ScaleInt(_Inout_ PINT Value, _In_ UINT OldDPI, _In_ UINT NewDPI)
{
    *Value = Math_RoundInt(*Value * ((FLOAT)NewDPI / OldDPI));
}

VOID NTAPI DPI_ScaleUInt(_Inout_ PUINT Value, _In_ UINT OldDPI, _In_ UINT NewDPI)
{
    *Value = Math_RoundUInt(*Value * ((FLOAT)NewDPI / OldDPI));
}
```
注意对负数的四舍五入：
```C
#define Math_RoundInt(val) ((INT)((val) + ((val) > 0 ? 0.5 : -0.5)))
```

我们据此缩放顶层窗口及其子窗口的位置和大小等值。在Windows下，窗口位置与大小一般使用RECT结构体描述：
```C
typedef struct tagRECT
{
    LONG    left;
    LONG    top;
    LONG    right;
    LONG    bottom;
} RECT, *PRECT, NEAR *NPRECT, FAR *LPRECT;
```
四个成员值表示窗口区域的左、上、右、下。对于顶层窗口的子窗口而言，其位置相对于父窗口，直接对这四个值进行缩放，然后传给SetWindowPos函数即可。实现效果如图所示：

<div align="center">
    <img src="Asset/Rectagle Scale.png"/>
</div>

DPI由100%调整到125%，则子窗口由实线示意的原位置缩放到虚线示意的新位置，其左、上、右、下均增加25%，反之亦然。下面是NTAssassin的相关实现：
```C
VOID NTAPI DPI_ScaleRect(_Inout_ PRECT Rect, _In_ UINT OldDPIX, _In_ UINT NewDPIX, _In_ UINT OldDPIY, _In_ UINT NewDPIY)
{
    DPI_ScaleInt(&Rect->left, OldDPIX, NewDPIX);
    DPI_ScaleInt(&Rect->right, OldDPIX, NewDPIX);
    DPI_ScaleInt(&Rect->top, OldDPIY, NewDPIY);
    DPI_ScaleInt(&Rect->bottom, OldDPIY, NewDPIY);
}
```
以上将RECT结构体的四个成员值根据新旧DPI进行缩放。

顶层窗口自身的缩放逻辑有别于其子窗口。由于顶层窗口的位置相对于整个屏幕，我们需要考虑其新位置是否合适。如若仍按照上述逻辑缩放，本处于屏幕正中的顶层窗口缩放后位置必然偏移，不再位于正中：若DPI增大，则造成顶层窗口向屏幕右下方偏移；若DPI减小，则造成顶层窗口向屏幕左上方偏移。处理逻辑很简单，根据DPI变化，将水平方向缩放产生的差平摊给左、右，垂直方向缩放产生的差平摊给上、下。实现效果如图所示：

<div align="center">
    <img src="Asset/Top-level Window Scale.png"/>
</div>

还要考虑缩放后超越屏幕左上边界的情况（尤其是标题栏出界可能会很不舒服），此时将其校正回左上角。

窗口缩放告一段落，接下来是字体缩放。Windows对话框本身在初始化时就会选择字体，应用到各个控件上。我们再此基础上，当DPI发生变化时，调整字体大小，再次应用到各个控件上即可。如果一定要考虑子控件使用的字体并不统一，那就只能挨个获取子控件字体进行缩放后再应用回去了。

相关可能用到的API或消息有 [WM_GETFONT](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-getfont) （获取窗口字体）、[WM_SETFONT](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-setfont) （设置窗口字体）、[GetObject](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-getobject) （获取字体对象信息）、[CreateFontIndirect](https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-createfontindirectw) （创建字体）等。

图像资源缩放，如加载位图、光标、图标等。缩放思路没什么特别，只是要在相关操作时留意，加入缩放逻辑即可。也不用担心将固定大小的资源缩放产生较严重的失真，因为最终物理显示总是在期望的大小附近。如果考虑最终物理显示大小会有变化，以及反复的拉伸导致图像模糊或锯齿，可以为同一幅图像准备多份不同放大倍数的副本，根据DPI选择最接近的那份进行缩放。

最后连起来，整个DPI自适应流程简而言之就两点：
- 顶层窗口初始化时，缩放一次，后续每次收到[WM_DPICHANGED](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged)也进行缩放
- 顶层窗口缩放时调整自身大小，以及子窗口大小。获取字体对象字号，缩放大小后再重新将字体对象应用到各子控件上。

NTAssassin完整的DPI自适应实现全代码：[NTAssassin/NTADPI.c at master · KNSoft/NTAssassin (github.com)](https://github.com/KNSoft/NTAssassin/blob/main/Source/Library/NTADPI.c)，约200多行，尚有不足，仅供参考。下面是基于此设计实现AlleyWind程序DPI适配效果的演示：

<video src="Asset/AlleyWind Demo.mp4" autoplay preload="auto" type="video/mp4"></video>

https://github.com/KNSoft/Blog/assets/55521781/31e3cf00-1786-41bb-acf2-132c05288948

<br/>
<br/>

## 四、方案与文档参考
实现方案：

[NTAssassin/NTADPI.c at master · KNSoft/NTAssassin (github.com)](https://github.com/KNSoft/NTAssassin/blob/main/Source/Library/NTADPI.c)

[KNSoft/AlleyWind: An advanced Win32-based and open-sourced utility that helps you to manage system's windows (github.com)](https://github.com/KNSoft/AlleyWind)

<br/>

文档参考：  
[DPI and device-independent pixels - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/learnwin32/dpi-and-device-independent-pixels)

[High DPI Desktop Application Development on Windows - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows)

[Improving the high-DPI experience in GDI based Desktop Apps - Windows Developer Blog](https://blogs.windows.com/windowsdeveloper/2017/05/19/improving-high-dpi-experience-gdi-based-desktop-apps/)

<br/>

尚有不足，或有谬误，欢迎指正，本文与以上实现也将随[NTAssassin](https://github.com/KNSoft/NTAssassin)、[AlleyWind](https://github.com/KNSoft/AlleyWind)等项目一同进步。若需要进一步研究，推荐参考以上MSDN资料。

<br/>
<hr>

<img src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png"/><br/>
本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议 (CC BY-NC-SA 4.0)](http://creativecommons.org/licenses/by-nc-sa/4.0/) 进行许可。  
<br/>
[Ratin](https://github.com/RatinCN) <[ratin@knsoft.org](mailto:ratin@knsoft.org)>  
国家认证 系统架构设计师  
ReactOS Contributor
