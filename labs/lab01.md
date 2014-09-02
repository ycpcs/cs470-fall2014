---
layout: default
title: "Lab 1: DirectX Framework I"
---

In CS370 we used GLUT to handle the window management for our OpenGL programs. DirectX provides a similar utility toolkit known as DXUT which provides similar window management functionality along with some additional features such as mesh loading capabilities. However aside from simple demonstration programs, DXUT is not generally used for actual DirectX programs. To best understand the inner workings of DirectX, we will forgo DXUT and instead write a basic DirectX application framework that we can extend as needed throughout the course. While this (and the next) lab will *not* produce a program that does anything, it will create the base application class that we will use in future programs. This base class is derived from the **D3DApp** class from our textbook *Introduction to 3D Game Programming with DirectX 11* by Frank Luna.

0. Getting Started
==================

Open Visual Studio 2013 through **Start -\> All Programs -\> Programming -\> Visual Studio 2013**.

On the start page select **New Project...**, expand the Visual C++ tab if necessary, select the **General** category, and choose **Empty Project**.

Name the project **CS470\_Lab01** and click **OK** (make sure the checkbox labelled **Create directory for solution** is checked).

Visual Studio should create a project for you that contains *no files* as we will be manually adding *everything*.

First we will include all the files from the **Common** directory. Right click in the solution explorer on the **CS470\_Lab01** and select **Add-\>New Filter** and rename the filter **Common**. Then right click **Common** and select **Add-\>Existing Item...**. Navigate to the **Common** directory from [lab0](lab00.html), select all the **.h** and **.cpp** files *except* for **d3dApp.h** and **d3dApp.cpp** as we will be creating these two files.

Next we need to configure the properties for the project. Select **Project-\>CS470\_Lab01 Properties** which should bring up the property sheets for the program, expand the **Configuration Properties** tab. Select **General** and switch the **Character Set** to **Use Unicode Character Set**.

Next, we will need to tell Visual Studio where the DirectX 11 headers and libraries are (from the SDK) as well as link the DirectX libraries into our project.

Select **VC++ Directories**, select **Include Directories**, click the arrow at the end of the row, and select **\<Edit...\>**. When the dialog box opens, select the first icon at the top (which looks like the icon to add a new folder) which should add a new line into the text box. Add the following paths (use the up/down arrows to arrange them in the specified order):
 
	$(IncludePath) 
	$(DXSDK_DIR)Include 
	C:"path to whereever Luna's Common folder is"

Select **VC++ Directories**, select **Library Directories**, click the arrow at the end of the row, and select **\<Edit...\>**. When the dialog box opens, select the first icon at the top (which looks like the icon to add a new folder) which should add a new line into the text box. Add the following paths (use the up/down arrows to arrange them in the specified order):
 
	$(LibraryPath) 
	$(DXSDK_DIR)Lib\x86 
	C:"path to wherever Luna's Common folder is"

Finally we need to link against the DirectX libraries. Expand the **Linker** tab and choose **Input**. Then edit the **Additional Dependencies** field to add the following libraries:

    d3d11.lib
    d3dx11d.lib
    D3DCompiler.lib
    Effects11d.lib
    dxerr.lib
    dxgi.lib
    dxguid.lib 

1. D3DApp Declaration
=====================

In C++, class declarations are usually placed in header files with the method definitions in the source file. We will follow this convention in this course.

Create a new header file (right click the **Header Files** folder and select **Add-\>New Item...** making sure to choose **Header File (.h)**) named **D3DApp.h**.

Add the following lines to the file

```cpp
#ifndef D3DAPP_H
#define D3DAPP_H

// Includes
#include "d3dUtil.h"
#include <string>

// Class declaration
class D3DApp
{
	// Attributes
protected:

	// Methods
public:

};

#endif  // D3DAPP_H
```

which includes the necessary header files and creates the (empty) class where the attributes will be **protected** and the methods will be **public**. Note: The **\#ifndef** simply ensures the header file is only included once in the project.

At this point we are ready to add the class attributes (variables) in the **protected** section. Note that by convention, class attributes begin with **m**. The following set of attributes will store information about the window

```cpp
HINSTANCE       mhAppInst;
HWND            mhMainWnd;
bool            mMinimized;
bool            mMaximized;
bool            mResizing;
UINT            m4xMsaaQuality;
std::wstring	mMainWndCaption;
int         	mClientWidth;
int         	mClientHeight;
bool            mEnable4xMsaa;
```

The next set of attributes will store pointers to the various DirectX COM objects and other information for DirectX

```cpp
D3D_DRIVER_TYPE 		md3dDriverType;
ID3D11Device*       	md3dDevice;
ID3D11DeviceContext*    md3dImmediateContext;
IDXGISwapChain*     	mSwapChain;
ID3D11Texture2D*    	mDepthStencilBuffer;
ID3D11RenderTargetView* mRenderTargetView;
ID3D11DepthStencilView* mDepthStencilView;
D3D11_VIEWPORT      	mScreenViewport;
```

The methods will be added to the **public** section of the class. The first set of methods will be the class constructor/destructor.

```cpp
D3DApp(HINSTANCE hInstance);
virtual ~D3DApp();
```

The next method will start the DirectX application

```cpp
int Run();
```

The next two methods will simply be for initialization of the window and Direct3D

```cpp
bool InitMainWindow();
bool InitDirect3D();
```

Finally the last set of methods are for the DirectX framework (note again they are all *virtual* and thus should be overridden in any derived subclasses with **UpdateScene()** **DrawScene()** *required* to be overridden).

```cpp
virtual bool Init();
virtual void OnResize();
virtual void UpdateScene(float dt)=0;
virtual void DrawScene()=0;
virtual LRESULT MsgProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam);
```

The first of these methods will initialize the window and DirectX, the second method will be called when the window is resized, the third and fourth methods will update the object states and render them in the scene, and the final method is the message handler for the application.

Finally we will add two getter methods for the window instance variables and the aspect ratio of the screen

```cpp
// Getter methods
HINSTANCE   AppInst() const;
HWND        MainWnd() const;
float       AspectRatio() const; 
```

This class will serve as the base class for any DirectX application and thus most of the work will be done in the overridden methods in the derived subclasses.

2. D3DApp Definition (partial)
==============================

As discussed above, the class definition will go into the source file.

Create a new source file (right click the **Source Files** folder and select **Add-\>New Item...** making sure to choose **Source File (.cpp)**) named **D3DApp.cpp**.

Add the following code to the source file:

```cpp
// Include class declaration
#include "D3DApp.h"

// Window message handler callback
namespace
{
	// Local pointer variable for forwarding messages
	D3DApp* gd3dApp = 0;
}

LRESULT CALLBACK MainWndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
	// Forward any messages
	return gd3dApp->MsgProc(hwnd, msg, wParam, lParam);
}

// Class methods
```

**Constructor**

The constructor will simply set initial values to create an 800x600 window, use the graphics hardware device, set the COM pointers to null, make the clear color blue, and disable multisampling.

```cpp
// Constructor
D3DApp::D3DApp(HINSTANCE hInstance)
{
	// Window attributes
	mhAppInst 		= hInstance;
	mMinimized  	= false;
	mMaximized  	= false;
	mResizing   	= false;
	m4xMsaaQuality  = 0;
	mMainWndCaption = L"D3D10 Application";
	mhMainWnd   	= 0;
	mClientWidth    = 800;
	mClientHeight   = 600;

	// DirectX attributes
	md3dDriverType       = D3D_DRIVER_TYPE_HARDWARE;
	md3dDevice      	 = 0;
	md3dImmediateContext = 0;
	mSwapChain      	 = 0;
	mDepthStencilBuffer  = 0;
	mRenderTargetView    = 0;
	mDepthStencilView    = 0;
	ZeroMemory(&mScreenViewport, sizeof(D3D11_VIEWPORT));

	// Set local pointer to this application for message handling
	gd3dApp = this;
}
```

**Destructor**

The destructor simply releases all the COM objects (using a utility macro defined in **d3dUtil.h**).

```cpp
// Destructor
D3DApp::~D3DApp()
{
	// Release D3D COM objects
	ReleaseCOM(mRenderTargetView);
	ReleaseCOM(mDepthStencilView);
	ReleaseCOM(mSwapChain);
	ReleaseCOM(mDepthStencilBuffer);

	if (md3dImmediateContext)
	{
		md3dImmediateContext->ClearState();
	}

	ReleaseCOM(md3dImmediateContext);
	ReleaseCOM(md3dDevice);
}
```

**Other Methods**

We will forego completing the other class methods (except the getters) until the next lab. For now we will simply put stubs in place.

```cpp
// Execute application
int D3DApp::Run()
{
	MSG msg = {0};

	return (int)msg.wParam;
}

bool D3DApp::InitMainWindow()
{
	return true;
}

bool D3DApp::InitDirect3D()
{
	return true;
}

// Initialization
bool D3DApp::Init()
{
	return true;
}

// Resize
void D3DApp::OnResize()
{
	// Check objects are not null
	assert(md3dImmediateContext);
	assert(md3dDevice);
	assert(mSwapChain);
}

// Message Handler
LRESULT D3DApp::MsgProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
	return 0;
}

// Getter methods
HINSTANCE D3DApp::AppInst() const
{
	return mhAppInst;
}

HWND D3DApp::MainWnd() const
{
	return mhMainWnd;
}

float D3DApp::AspectRatio() const
{
	return static_cast<float>(mClientWidth) / (mClientHeight);
}
```

3. Test Application
===================

Similar to standard C programs which require a **main()** function, Windows programs require a function named **WinMain()**. In our programs, we will simply subclass **D3DApp** to add a **WinMain()** function and override the various methods in **D3DApp** to create the scene.

Create a new source file (right click the **Source Files** folder and select **Add-\>New Item...** making sure to choose **Source File (.cpp)**) named **D3DTestApp.cpp**.

```cpp
// Include base class declaration
#include "D3DApp.h"

// Subclass declaration
class D3DTestApp : public D3DApp
{
public:
	D3DTestApp(HINSTANCE);
	~D3DTestApp();

	// Overriden methods
	bool Init();
	void OnResize();
	void UpdateScene(float dt);
	void DrawScene();
};

// Class methods
D3DTestApp::D3DTestApp(HINSTANCE hInstance) : D3DApp(hInstance)
{};

D3DTestApp::~D3DTestApp()
{};

bool D3DTestApp::Init()
{
	// Simply use base class initializer
	if (!D3DApp::Init())
	{
		return false;
	}

	return true;
}

void D3DTestApp::OnResize()
{
	// Simply use base class resize
	D3DApp::OnResize();
}

void D3DTestApp::UpdateScene(float dt)
{};

void D3DTestApp::DrawScene()
{};


// Windows main function
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE prevInstance, PSTR cmdLine, int showCmd)
{
	// Add some debug flags if debugging
#if defined(DEBUG) | defined(_DEBUG)
	_CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
#endif

	// Instantiate application object
	D3DTestApp theApp(hInstance);

	// Initialize the application
	if (!theApp.Init())
	{
		return 0;
	}

	// Execute the application
	return theApp.Run();
}
```

4. Compiling and running the program
====================================

Once you have completed typing in the code, you can build the program in one of two ways:

> -   Click the small green arrow in the middle of the top toolbar
> -   Hit **F5** (or **Ctrl-F5**)

At this point the program should compile without warnings or errors and will simply exit when executed.

Next time we'll complete the definition of the **D3DApp** class to produce a blank window (for DirectX 11).

