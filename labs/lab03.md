---
layout: default
title: "Lab 3: Basic Cube I"
---

The last lab provided a base DirectX 11 application class that initializes the various DirectX 11 COM objects along with a derived subclass that creates a window. However, in order to create a *useful* application, we will need to add simple geometric objects, and write a simple pass through shader to render the objects. In this lab we will see how to create geometry (by hand) and in the next lab we will add shaders to render the object. Note that while we created the **D3DApp** base class in the last two labs, going forward we will just use Luna's version found in the **Common** directory from the textbook.

0. Getting Started
==================

Download [CS470_Lab03.zip](src/CS470_Lab03.zip), saving it into the **labs** directory.

Double-click on **CS470\_Lab03.zip** and extract the contents of the archive into a subdirectory called **CS470\_Lab03**. Place this subdirectory into the **Documents/Visual Studio 2013/Projects** directory.

Navigate into the **CS470\_Lab03** directory and double-click on **CS470\_Lab03.sln** (the file with the little Visual Studio icon with the 12 on it) which should immediately open Visual Studio with the project.

If the **Header Files**, **Resource Files** and **Source Files** folders in the **Solution Explorer** pane are not expanded, expand each by double clicking on them and double-click on **BasicCubeApp.cpp**.

**BasicCubeApp.cpp** contains the main program (i.e. **WinMain()**) which is identical to our test application program from before. Note that the type of **theApp** is now **BasicCubeApp**.

1. Basic Cube Application Class Declaration
===========================================

As explained in [lab01](lab01.html), class declarations are usually placed in header files (**.h**) with the corresponding method definitions in the source file (**.cpp**). Select the **BasicCubeApp.h** file to open the header file. Note this class will be a subclass of **D3DApp** and use an object (which we will define later) named **Cube**, thus we must include the base class header file and tell windows that it is a subclass along with providing the signatures of the methods we will need as:

```cpp
#include "d3dApp.h"
#include "Cube.h"
#include <string>

// Subclass declaration
class BasicCubeApp : public D3DApp
{
private:
	// Class attributes

public:
	BasicCubeApp(HINSTANCE);
	~BasicCubeApp();

	// Overriden methods
	bool Init();
	void OnResize();
	void UpdateScene(float dt);
	void DrawScene();
};
```

Since our derived class inherits all the attributes of the base class (for the basic windows and DirectX objects), we will only add attributes specific to the application. The first attributes will be a **Cube** object, the filename for the shader file, and COM objects for the shader *effects* we will be using. A shader can contain multiple *techniques* (i.e. a particular combination of a vertex, geometry, and/or pixel shader) so we will need an attribute for that COM object as well. Additionally we will add some attributes to store various scene properties such as the vertex buffer (known in DirectX as a layout), global transformation matrices, and the camera vectors. Note that DirectX provides typedef's for a variety of vector/matrix data types which should **ALWAYS** be used for graphics objects rather than natively declaring array variables.

```cpp
private:
	// Class attributes

	// Cube object
	Cube mBox;

	// Shader variables
	std::wstring mFXFileName;
	ID3DX11Effect* mFX;
	ID3DX11EffectTechnique* mTech;
	ID3DX11EffectMatrixVariable* mFXMatVar;

	// Vertex buffer
	ID3D11InputLayout* mVertexLayout;

	// Global transformations
	XMFLOAT4X4 mView;
	XMFLOAT4X4 mProj;
	XMFLOAT4X4 mWVP;

	// Camera vectors
	XMFLOAT3 mEye;
	XMFLOAT3 mAt;
	XMFLOAT3 mUp;
```

Lastly we will add two private methods - one to load the shader and another to set the vertex layout.

```cpp
private:
	// Shader methods
	void buildFX();
	void buildVertexLayouts();
```

2. BasicCubeApp Definition
==========================

The definitions for the class methods are in **BasicCubeApp.cpp**.

**Constructor**

First we define the constructor to properly initialize the class attributes (after calling the base class constructor) through additional list initializers which set all the pointers to null, then assign the matricies to identity, and set a default camera orientation.

```cpp
// Constructor
BasicCubeApp::BasicCubeApp(HINSTANCE hInstance) : D3DApp(hInstance), mFX(0),
	mTech(0), mVertexLayout(0), mFXMatVar(0)
{
	// Change the window caption
	mMainWndCaption = L"BasicCube Application";

	// Initialize matricies to identity
	XMMATRIX Ident = XMMatrixIdentity();
	XMStoreFloat4x4(&mView, Ident);
	XMStoreFloat4x4(&mProj, Ident);
	XMStoreFloat4x4(&mWVP, Ident);

	// Initialize camera orientation
	mEye    = {0.0f,0.0f,1.0f};
	mAt = {0.0f,0.0f,0.0f};
	mUp = {0.0f,1.0f,0.0f};
}
```

**Destructor**

The destructor will simply release the COM objects we added as attributes (remember the base class destructor will handle the other ones).

```cpp
// Destructor
BasicCubeApp::~BasicCubeApp()
{       
	ReleaseCOM(mFX);
	ReleaseCOM(mVertexLayout);
}
```

**Initialization**

The initialization method calls the base class initialization method (which sets up the window and DirectX),then initializes a **Cube** object, calls the methods to create the effect and vertex layout, and finally creates the view matrix (using a similar command to **gluLookAt()**) from the camera vectors.

 ```cpp
// Call cube object initialization
mBox.Init(md3dDevice);

// Create effect and vertex layout
mFXFileName = L"basiccube.fx";
buildFX();
buildVertexLayouts();

// Initialize view matrix from camera
mEye    = {5.0f,5.0f,5.0f};
mAt = {0.0f,0.0f,0.0f};
mUp = {0.0f,1.0f,0.0f};

XMVECTOR e = XMLoadFloat3(&mEye);
XMVECTOR a = XMLoadFloat3(&mAt);
XMVECTOR u = XMLoadFloat3(&mUp);

XMMATRIX camera = XMMatrixLookAtLH(e,a,u);
XMStoreFloat4x4(&mView, camera);
```

**Resize**

The resize method simply resets the projection frustum (again using a similar command to **gluPerspective()**).

```cpp
// Call base class resize
D3DApp::OnResize();

// Reset (90 degree fov) projection frustum
XMMATRIX proj = XMMatrixPerspectiveFovLH(0.25f*MathHelper::Pi, AspectRatio(), 1.0f, 1000.0f);
XMStoreFloat4x4(&mProj, proj);
```

**Draw Scene**

The majority of the work will be done in the **DrawScene()** method (again we will not use the **UpdateScene()** method as it will simply be a static object for now). After clearing the screen, we tell the pipeline to use our vertex layout (buffer) which we will create later.

```cpp
// Set input layout to object vertices
md3dImmediateContext->IASetInputLayout(mVertexLayout);
```

Since our vertices will be defining triangles, we need to set the topology to a *triangle list*

```cpp
// Set topology (triangle list)
md3dImmediateContext->IASetPrimitiveTopology(D3D11_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
```

Now we compute the complete modelview projection matrix (note that DirectX overloads the * operator for matrix multiplication) and set the corresponding shader variable (which we will associate in the **buildFX()** method). Note: *Unlike* OpenGL, DirectX uses *left* multiplication for concatenation of matrices.

```cpp
// Compute total modelview-projection matrix
XMMATRIX v = XMLoadFloat4x4(&mView);
XMMATRIX p = XMLoadFloat4x4(&mProj);    
XMMATRIX mWVP = v*p;
mFXMatVar->SetMatrix(reinterpret_cast<float*>(&mWVP));
```

Finally we are ready to draw our object by looping through all the passes in our technique (which will be discusses later), set the pass to use for rendering the object, and call the draw method for our object(s).

```cpp
// Get technique and loop over passes
D3DX11_TECHNIQUE_DESC td;
mTech->GetDesc(&td);
for (UINT p = 0; p < td.Passes; p++)
{
	// Set pass
	mTech->GetPassByIndex(p)->Apply(0, md3dImmediateContext);

	// Draw objects
	mBox.Draw(md3dImmediateContext);
}
```

Lastly, since we have drawn all our geometry, we are ready to swap the buffers to display the scene on the screen (i.e. equivalent to **glutSwapBuffers()**).

```cpp
// Swap buffers
mSwapChain->Present(0,0);
```

**Build Effect**

Unlike OpenGL where we needed to do quite a bit of work to load shaders into our application (including reading the source, compiling the source, linking, etc.), since DirectX relies on shaders it is much simpler (basically a single API call) to set the **mFX** attribute from a shader file.

```cpp
// Load effects file
ID3D10Blob* shader = 0;
ID3D10Blob* compilationErrors = 0;
HRESULT hr = D3DX11CompileFromFile(mFXFileName.c_str(),0,0,0,"fx_5_0",shaderFlags,0,0,&shader,&compilationErrors,0);


// Output debug info if compilation failed
if (FAILED(hr))
{
	// Check for compilation errors or warnings
	if (compilationErrors != 0)
	{
		MessageBoxA(0, (char*)compilationErrors->GetBufferPointer(), 0, 0);
		ReleaseCOM(compilationErrors);
	}
	DXTrace(__FILE__,(DWORD)__LINE__,hr,L"D3DX11CompileFromFile",true);
}
```

Next we create the effect from the shader and store it in our class attribute and free up the local COM object

```cpp
// Create effect from shader
HR(D3DX11CreateEffectFromMemory(shader->GetBufferPointer(), shader->GetBufferSize(), 
								0, md3dDevice, &mFX));
ReleaseCOM(shader);
```

Finally we obtain pointers to the technique we want and associate any shader variables we will be using (which for this application is the modelview-projection matrix that will be set in **DrawScene()** above).

```cpp
// Get technique from effect
mTech = mFX->GetTechniqueByName("BasicTech");

// Associate modelview-projection shader variable
mFXMatVar = mFX->GetVariableByName("gWVP")->AsMatrix();
```

**Build Vertex Layout**

Our last method for the **BasicCubeApp** class will build the vertex layout(s) used to associate our application vertex structure with the shader vertex structure. This layout will describe how the vertex buffer coming from the application is partitioned into the various vertex values. In this case it tells the shader that each vertex consists of 3 (4-byte) floating point values for the position. Since the position is a total of 12 bytes, the color field will be offset by 12 bytes and contain 4 (4-byte) floating point values.

```cpp
// Create vertex layout description from Vertex structure
D3D11_INPUT_ELEMENT_DESC vertexDesc[] = 
{
	{"POSITION",0,DXGI_FORMAT_R32G32B32_FLOAT,0,0,D3D11_INPUT_PER_VERTEX_DATA,0},
	{"COLOR",0,DXGI_FORMAT_R32G32B32A32_FLOAT,0,12,D3D11_INPUT_PER_VERTEX_DATA,0}
};

// Create vertex layout for pass 0
D3DX11_PASS_DESC pd;
mTech->GetPassByIndex(0)->GetDesc(&pd);

HR(md3dDevice->CreateInputLayout(vertexDesc,2,pd.pIAInputSignature,pd.IAInputSignatureSize,&mVertexLayout));
```

4. Compiling and running the program
====================================

Once you have completed typing in the code, you can build and run the program in one of two ways:

> -   Click the small green arrow in the middle of the top toolbar
> -   Hit **F5** (or **Ctrl-F5**)

Again not much will happen except the program will crash with a file not found error since we have not defined a shader file yet.

To quit the program simply close the window.

The last pieces of the puzzle yet to implement are creating the cube geometry in a format that DirectX can use and adding a shader file to tell DirectX how to render the geometry.

