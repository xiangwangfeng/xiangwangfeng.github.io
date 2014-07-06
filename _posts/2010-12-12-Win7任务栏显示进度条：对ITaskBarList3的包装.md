---
layout: post
title: Win7任务栏显示进度条：对ITaskBarList3的包装
---


差不多在做Phoebe的时候已经看到有一些程序实现了Win7的一些新特性，如Chrome就支持下载进度在任务栏上显示，当时本来想做做看，不过考虑到又要和原来的程序整合，而且意义也不是特别大，所以也就懒的弄。但是最近闪印基本又要发新版本了，自己这边基本又没什么bug，蛋疼得慌，无聊查了查MSDN上相应的接口定义 ，整了个大概。

基本上，Win7下任务栏的新特性都是通过ITaskBarList3这个接口来实现，其定义可以在Shobjidl.h中找到(Win7 SDK)，像我们用的是VS2008 SP1，默认的SDK版本号是6.1A，是找不到其定义的。所以比较直接的做法自然是安装Win7 SDK，将这个SDK设置成当前VS的默认SDK。但这样做的最大问题就是兼容性，Win7 SDK下很多头文件和原先的并不兼容，尤其是strsafe.h这个头文件，很多定义都发生了变化，跑老工程基本会有一堆的编译错误。尝试过将原先的strsafe.h和相应的lib文件替换掉Win7 SDK的内容，编译运行，一切OK，但是毕竟为了加这么点功能，把SDK里面原有的文件修改掉，带来的风险也太大了，所以这种方法还是不太可取。

于是相应的只能采取比较取巧的方法，因为本身ITaskBarList3是基于COM的，所以我们只要提供一个正确的GUID和相应方法定义即可，跑WIN7 SDK的include文件下把ITaskBarList3的定义整理出来即可：
ITaskBarList3.h

    #ifndef __ITaskbarList3_INTERFACE_DEFINED__
#define __ITaskbarList3_INTERFACE_DEFINED__
/* interface ITaskbarList3 */
/* [object][uuid] */
EXTERN_C const IID IID_ITaskbarList3;
#if defined(__cplusplus) && !defined(CINTERFACE)
typedef
enum TBPFLAG
{   
TBPF_NOPROGRESS    = 0,
TBPF_INDETERMINATE    = 0x1,
TBPF_NORMAL    = 0x2,
TBPF_ERROR    = 0x4,
TBPF_PAUSED    = 0x8
} TBPFLAG;
typedef struct THUMBBUTTON *LPTHUMBBUTTON;
MIDL_INTERFACE("EA1AFB91-9E28-4B86-90E9-9E9F8A5EEFAF")
ITaskbarList3 : public ITaskbarList2
{
public:
virtual HRESULT STDMETHODCALLTYPE MarkFullscreenWindow(
/* [in] */ __RPC__in HWND hwnd,
/* [in] */ BOOL fFullscreen) = 0;
// ITaskbarList3 members
STDMETHOD (SetProgressValue)     (HWND hwnd, ULONGLONG ullCompleted, ULONGLONG ullTotal) PURE;
STDMETHOD (SetProgressState)     (HWND hwnd, TBPFLAG tbpFlags) PURE;
STDMETHOD (RegisterTab)          (HWND hwndTab,HWND hwndMDI) PURE;
STDMETHOD (UnregisterTab)        (HWND hwndTab) PURE;
STDMETHOD (SetTabOrder)          (HWND hwndTab, HWND hwndInsertBefore) PURE;
STDMETHOD (SetTabActive)         (HWND hwndTab, HWND hwndMDI, DWORD dwReserved) PURE;
STDMETHOD (ThumbBarAddButtons)   (HWND hwnd, UINT cButtons, LPTHUMBBUTTON pButton) PURE;
STDMETHOD (ThumbBarUpdateButtons)(HWND hwnd, UINT cButtons, LPTHUMBBUTTON pButton) PURE;
STDMETHOD (ThumbBarSetImageList) (HWND hwnd, HIMAGELIST himl) PURE;
STDMETHOD (SetOverlayIcon)       (HWND hwnd, HICON hIcon, LPCWSTR pszDescription) PURE;
STDMETHOD (SetThumbnailTooltip)  (HWND hwnd, LPCWSTR pszTip) PURE;
STDMETHOD (SetThumbnailClip)     (HWND hwnd, RECT *prcClip) PURE;
};
#else     /* C style interface */
#endif     /* C style interface */
#endif     /* __ITaskbarList3_INTERFACE_DEFINED__ */

配置整完了，使用就很简单了，就三步(进度条)：

 1. CoCreate ITaskbarList3接口
 2. 调用SetProgressState来设置进度条状态
 3. 调用SetProgressValue更新进度条

不管相应的注意事项却不少，这也正是需要进行包装的原因：

 1. 对ITaskbarList3进行初始化的时机应该是在收到TaskbarButtonCreated的消息之后，而比较诡异的是微软是不预先定义这个消息ID的，必须通过向系统注册相应消息的方法来获取：
DWORD WM_TASKBARCREATED = ::RegisterWindowMessage(L"TaskbarButtonCreated");
 2. 在非UI线程中调用接口相应的方法是无效的，而且更加诡异的是这个接口本身对这方面有任何出错处理或者提示(至少在DEBUG下有相应的ASSERT出来也是好的)，无论在哪个线程调用相应接口都是返回S_OK。所以需要一个转发消息的窗口进行消息的转发来实现线程的转换，这也是对这个接口进行封装的最大理由。（PS：感谢老汉老汉告知原来在CreateEx时传入HWMD_MESSAGE的参数可以创建一个仅做消息转发而无需更多资源的窗口）
 3. 一些必要的出错处理，比如对系统类型的判断。
包装完后就有了一个简单的的用于任务栏进度条显示的类,具体的测试代码和工程可以参考这里：[单击下载][1]

[1]:http://amaoproject.googlecode.com/files/ProgressTaskBar.7z