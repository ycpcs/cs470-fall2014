---
layout: default
title: "Lab 2: DirectX Framework II"
---

In the last lab we wrote the basic windows functions, declared the **D3DApp** class, and wrote the constructor/destructor methods (along with stubs for the other functions). In this lab we will complete the methods that contain the functionality of the **D3DApp** base class. While our application will still not render anything, it will open a window that is capable of rendering DirectX graphics.

0. Getting Started
==================

Since we will be continuing with the **CS470\_Lab01** project from last time, we will simply open that project.

Open Visual Studio 2013 through **Start -\> All Programs -\> Programming -\> Visual Studio 2013**.

On the start page select **Open Project...**, navigate to your **labs/CS470\_Lab01** directory, and select the CS470\_Lab01.sln. (Alternatively you can simply choose it if it appears in the **Recent Projects** section.)

Open the source file (if it isn't open already) **D3DApp.cpp**.

1. Run Method
=============

This method executes the application by either dispatching any messages or rendering the scene (with a 16 ms delay - 60 fps).

    // Execute application
    int D3DApp::run()
    {
        MSG msg = {0};

        while (msg.message != WM_QUIT)
        {
            // Process window messages
            if (PeekMessage(&msg,0,0,0,PM_REMOVE))
            {
                TranslateMessage(&msg);
                DispatchMessage(&msg);
            }
            // Otherwise render frame (using stub update)
            else
            {
                Sleep(16);
                UpdateScene();
                DrawScene();
            }
        }

        return (int)msg.wParam;
    }

2. Initialization Methods
=========================

Now the methods get more interesting. The **InitMainWindow()** method creates the window (which was handled by GLUT in CS370) while **InitDirect3D()** initializes various DirectX structures. Refer to on-line MSDN Library documentation for more detailed information regarding the structure fields.

**InitMainWindow()**

The first structure sets various properties for the window such as the style, cursor, icon, and class name and also links the message handling function (remember from the **extern** in the class declaration)

    // Initialization
    void D3DApp::initApp()
    {
        // Initialize main window structure
        WNDCLASS wc;
        wc.style        	= CS_HREDRAW | CS_VREDRAW;
        wc.lpfnWndProc      = MainWndProc;
        wc.cbClsExtra       = 0;
        wc.cbWndExtra       = 0;
        wc.hInstance        = mhAppInst;
        wc.hIcon        	= LoadIcon(0, IDI_APPLICATION);
        wc.hCursor      	= LoadCursor(0, IDC_ARROW);
        wc.hbrBackground    = (HBRUSH)GetStockObject(NULL_BRUSH);
        wc.lpszMenuName     = 0;
        wc.lpszClassName    = L"D3DWndClassName";

Next we need to register the class with Windows (terminating the program if it fails)

    // Register window class
    if (!RegisterClass(&wc))
    {
        MessageBox(0,L"RegisterClass FAILED!",0,0);
        return false;
    }

At this point we adjust our window size to account for the border so that our client area (for drawing) is what we specified earlier (800x600).

    // Get full rectangle size based on desired client size
    RECT R = {0,0,mClientWidth,mClientHeight};
    AdjustWindowRect(&R,WS_OVERLAPPEDWINDOW,false);
    int width = R.right - R.left;
    int height = R.bottom - R.top;

And finally we actually create the window (storing the handle into the corresponding attribute), show it on the screen, and post a message to redraw it

    // Create window
    mhMainWnd = CreateWindow(L"D3DWndClassName",mMainWndCaption.c_str(),WS_OVERLAPPEDWINDOW,
                CW_USEDEFAULT,CW_USEDEFAULT,width,height,0,0,mhAppInst,0);
    if (!mhMainWnd)
    {
        MessageBox(0,L"CreateWindow FAILED!",0,0);
        return false;
    }
    ShowWindow(mhMainWnd,SW_SHOW);
    UpdateWindow(mhMainWnd);

**InitDirect3D()**

At this point we have a basic window but no way to draw DirectX graphics into it. Thus we will first create a device and device context to ensure it supports DirectX 11, and then check for 4x MSAA support quality.

    // Create D3D device
    UINT createDeviceFlags = 0;
	
	#if defined(DEBUG) || defined(_DEBUG)  
		createDeviceFlags |= D3D11_CREATE_DEVICE_DEBUG;
	#endif
	
	// Set feature level 
	D3D_FEATURE_LEVEL featureLevel; 
	HRESULT hr = D3D11CreateDevice(0, md3dDriverType, 0, createDeviceFlags, 0, 0,
								   D3D11_SDK_VERSION, &md3dDevice, &featureLevel,
								   &md3dImmediateContext);
	
	// Could not create device 
	if (FAILED(hr)) 
	{ 
		MessageBox(0, L"D3D11CreateDevice Failed!", 0, 0); 
		return false; 
	}
	
	// Device does not support DirectX 11 
	if (featureLevel != D3D_FEATURE_LEVEL_11_0) 
	{ 
		MessageBox(0, L"Direct3D Feature Level 11 Unsupported!", 0, 0); 
		return false; 
	}
	
	// Everything ok, so check for 4x MSAA support quality
	HR(md3dDevice->CheckMultisampleQualityLevels(DXGI_FORMAT_R8G8B8A8_UNORM, 4, &m4xMsaaQuality)); 
	assert(m4xMsaaQuality > 0);

Now we must set up the swap chain which allows us to switch between the front and back buffer, i.e. double buffering. We first set various fields for the swap chain structure for the frame buffer size, refresh rate (60Hz), color format (32-bit RGBA in this case), disable multisampling (for now), tell DirectX that this is an output target (i.e. we will be drawing in it), and to attach it to our main window.

    // Initialize D3D swap chain structure
    DXGI_SWAP_CHAIN_DESC sd;
    sd.BufferDesc.Width         			= mClientWidth;
    sd.BufferDesc.Height            		= mClientHeight;
    sd.BufferDesc.RefreshRate.Numerator 	= 60;
    sd.BufferDesc.RefreshRate.Denominator   = 1;
    sd.BufferDesc.Format            		= DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.BufferDesc.ScanlineOrdering      	= DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
    sd.BufferDesc.Scaling           		= DXGI_MODE_SCALING_UNSPECIFIED;

    // 4x MSAA Quality
    if (mEnable4xMsaa)
    {
        sd.SampleDesc.Count		= 4;
        sd.SampleDesc.Quality 	= m4xMsaaQuality - 1;
    } else {
        sd.SampleDesc.Count		= 1;
        sd.SampleDesc.Quality 	= 0;
    }

    sd.BufferUsage 	= DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.BufferCount	= 1;
    sd.OutputWindow = mhMainWnd;
    sd.Windowed		= true;
    sd.SwapEffect	= DXGI_SWAP_EFFECT_DISCARD;
    sd.Flags		= 0;

Finally we need to create the swap chain through several COM objects and call the resize method which performs the remaining initializations (which also uses the same code when the window is resized)
  
	IDXGIDevice* dxgiDevice = 0; 
	HR(md3dDevice->QueryInterface(__uuidof(IDXGIDevice), (void**)&dxgiDevice);
	
	IDXGIAdapter* dxgiAdapter = 0; 
	HR(dxgiDevice->GetParent(__uuidof(IDXGIAdapter), (void**)&dxgiAdapter);
	
	IDXGIFactory* dxgiFactory = 0; 
	HR(dxgiAdapter->GetParent(__uuidof(IDXGIFactory), (void**)&dxgiFactory);
	
	HR(dxgiFactory->CreateSwapChain(md3dDevice, &sd, &mSwapChain);
	
	ReleaseCOM(dxgiDevice); 
	ReleaseCOM(dxgiAdapter); 
	ReleaseCOM(dxgiFactory);
	
	OnResize();

**Init()**

The **Init()** method will simply call **InitMainWindow()** and **InitDirect3D()** to ensure everything was setup properly.

    // Initialize main window
    if(!InitMainWindow())
    {
        return false;
    }

    // Initialize Direct3D 11
    if(!InitDirect3d())
    {
        return false;
    }

3. Resize Method
================

This method will perform the remaining initialization of the render target view, depth/stencil buffer, and viewport. This is done in a separate method as these three objects must also be reset whenever the window is resized (but the swap chain and device remain unchanged). However first the old objects must be released (which upon initialization they are null) before new ones can be created and the swap chain buffer must be resized to the new width and height.

    // Release previous COM objects
    ReleaseCOM(mRenderTargetView);
    ReleaseCOM(mDepthStencilView);
    ReleaseCOM(mDepthStencilBuffer);
	
    // Resize swap chain buffer
  	HR(mSwapChain->ResizeBuffers(1,mClientWidth,mClientHeight,DXGI_FORMAT_R8G8B8A8_UNORM,0));

At this point we (re)create the render target view through the device based on the swap chain buffer

    // (Re)create render target view
    ID3D11Texture2D* backBuffer;
    HR(mSwapChain->GetBuffer(0,__uuidof(ID3D11Texture2D),reinterpret_cast<void**>(&backBuffer)));
    HR(md3dDevice->CreateRenderTargetView(backBuffer,0,&mRenderTargetView));
    ReleaseCOM(backBuffer);

We now define the fields of the depth/stencil buffer structure (which is similar to a texture) of the same size as the render target (with 24 bits for depth and 8 bits for stencil), appropriate multisampling (matched to our swap chain), and create the buffer

    // (Re)create depth/stencil buffer
    D3D11_TEXTURE2D_DESC depthStencilDesc;
    depthStencilDesc.Width      = mClientWidth;
    depthStencilDesc.Height     = mClientHeight;
    depthStencilDesc.MipLevels	= 1;
    depthStencilDesc.ArraySize	= 1;
    depthStencilDesc.Format     = DXGI_FORMAT_D24_UNORM_S8_UINT;

    if (mEnable4xMsaa)
    {
        depthStencilDesc.SampleDesc.Count = 4;
        depthStencilDesc.SampleDesc.Quality = m4xMsaaQuality - 1;
    }
    else
    {
        depthStencilDesc.SampleDesc.Count = 1;
        depthStencilDesc.SampleDesc.Quality = 0;
    }

    depthStencilDesc.Usage      	= D3D11_USAGE_DEFAULT;
    depthStencilDesc.BindFlags      = D3D11_BIND_DEPTH_STENCIL;
    depthStencilDesc.CPUAccessFlags = 0;
    depthStencilDesc.MiscFlags      = 0;
    HR(md3dDevice->CreateTexture2D(&depthStencilDesc,0,&mDepthStencilBuffer));

Using the buffer, we can create the depth/stencil view and bind it to the render target context

    // (Re)create depth/stencil view
    HR(md3dDevice->CreateDepthStencilView(mDepthStencilBuffer,0,&mDepthStencilView));

    // Bind to render target
    md3dImmediateContext->OMSetRenderTargets(1,&mRenderTargetView,mDepthStencilView);

Finally (just like with OpenGL) we set a viewport that matches our screen extents. Note that in DirectX all of our depth values will be normalized to the range [0,1].

    // Set viewport
    mScreenViewport.TopLeftX = 0;
    mScreenViewport.TopLeftY = 0;
    mScreenViewport.Width	 = static_cast<float>(mClientWidth);
    mScreenViewport.Height   = static_cast<float>(mClientHeight);
    mScreenViewport.MinDepth = 0.0f;
    mScreenViewport.MaxDepth = 1.0f;
    md3dImmediateContext->RSSetViewports(1,&mScreenViewport);

> }

4. Message Handler Method
=========================

The final method in the class will handle some of the messages generated by the window. These messages will primarily be when the window is activated, resized, or destroyed. Again any derived classes will want to override this method to extend the functionality to other messages such as user input. The method itself should be fairly self explanatory.

    // Message handler
    LRESULT D3DApp::msgProc(UINT msg, WPARAM wParam, LPARAM lParam)
    {
        switch(msg)
        {
            // Window activated
            case WM_ACTIVATE:
                break;
            // Window resized
            case WM_SIZE:
                // Save new dimensions
                mClientWidth = LOWORD(lParam);
                mClientHeight = HIWORD(lParam);

                // Update size flags
                if (md3dDevice)
                {
                    if (wParam == SIZE_MINIMIZED)
                    {
                        mMinimized = true;
                        mMaximized = false;
                    }
                    else if (wParam == SIZE_MAXIMIZED)
                    {
                        mMinimized = false;
                        mMaximized = true;
                        OnResize();
                    }
                    else if (wParam == SIZE_RESTORED)
                    {
                        // From minimzed
                        if (mMinimized)
                        {
                            mMinimized = false;
                            onResize();
                        }
                        // From maximized
                        else if (mMaximized)
                        {
                            mMaximized = false;
                            onResize();
                        }
                        // Still dragging - don't resize yet
                        else if (mResizing)
                        {
                        }
                        // Other system resize calls
                        else
                        {
                            onResize();
                        }
                    }
                }
                break;
            // Beginning resizing
            case WM_ENTERSIZEMOVE:
                mResizing = true;
                break;
            // Done resizing
            case WM_EXITSIZEMOVE:
                mResizing = false;
                onResize();
                break;
            // Window being destroyed
            case WM_DESTROY:
                PostQuitMessage(0);
                break;
            // Limit window from becoming too small
            case WM_GETMINMAXINFO:
                ((MINMAXINFO*)lParam)->ptMinTrackSize.x = 200;
                ((MINMAXINFO*)lParam)->ptMinTrackSize.y = 200;
                break;
            // Otherwise pass message to main window handler
            default:
                return DefWindowProc(mhMainWnd,msg,wParam,lParam);
        }

        return 0;
    }

5. (Default) Draw Scene Method
==============================

To complete the application, we will add some simple code to the **DrawScene()** method in **D3DTestApp()** (recall that we declared this method as purely virtual such that all subclasses *must* override it.) Since we do not have any objects in the scene we will not add anything to **UpdateScene()** but again still need to implement the method in the subclass. This default draw scene method will do nothing more than clear the screen (render target view) and depth/stencil view. Eventually our subclasses will be responsible for any object rendering.

    // Draw scene
    void D3DApp::drawScene()
    {
        // Check for valid rendering context and swap chain
        assert(md3dImmediateContext);
        assert(mSwapChain);
        // Simply clear views
        md3dImmediateContext->ClearRenderTargetView(mRenderTargetView, reinterpret_cast<const float*>(&Colors::Blue));
        md3dImmdediateContext->ClearDepthStencilView(mDepthStencilView, D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL,1.0f,0);

        // Swap back buffer
        HR(mSwapChain->Present(0, 0));
    }

6. Compiling and running the program
====================================

Once you have completed typing in the code, you can build the program in one of two ways:

> -   Click the small green arrow in the middle of the top toolbar
> -   Hit **F5** (or **Ctrl-F5**)

At this point you should have a blank blue window (that actually is set up for DirectX 11).

Next time we'll see how to subclass **D3DApp** to create a *functional* DirectX 11 program.

