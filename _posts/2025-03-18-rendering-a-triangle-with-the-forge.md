---
title: Rendering a triangle with The Forge
description: Guide to setting up a minimal example with The Forge Framework.
date: 2025-03-18 00:21 -0400
categories: [programming, graphics programming]
tags: [graphics programming, the forge]
media_subpath: /assets/2025-03-18-rendering-a-triangle-with-the-forge
---

After implementing my own renderer from scratch with Vulkan, I think it's time to move on to using something a bit higher level. I decided to try out [The Forge](https://github.com/ConfettiFX/The-Forge) by ConfettiFX. Unfortunately, the documentation is rather lacking, and the examples are really the only resources available for learning it.

There is a [guide on building The Forge by LCG](https://rendering-techniques.learn-computer-graphics.com/doc/getting-started/building.html), but it's outdated and didn't work for me. I managed to get a triangle rendered on the screen after a few hours of tinkering. In this post, I'll share the steps I took, to help others like me get started with the framework.

This guide is mainly geared towards people with experience in Vulkan, or any of the modern graphics APIs.

## Prerequisites (Windows)
**TL;DR**: Download Visual Studio 2019 version 16.10 [here](https://download.visualstudio.microsoft.com/download/pr/1192d0de-5c6d-4274-b64d-c387185e4f45/b6bf2954c37e1caf796ee06436a02c79f7b13ae99c89b8a3b3b023d64a5935e4/vs_Community.exe), select and install "Desktop development with C++" package and Windows 10 SDK (10.0.17763.0).

ConfettiFX provides IDE project files for the supported platforms, and since I use Windows, that will be Visual Studio.

The Forge is only tested on Visual Studio 2019 with Windows SDK 10.0.17763.0, so that's what we're going to use. If we take a look at `Application/Config.h`, it specifically wants `_MSC_VER == 1929`, which is Visual Studio 2019 version 16.10.
```c
// Whitelist of compiler versions
#if (_MSC_VER == 1929) // VS 2019 all VC++ compilers
#else
#pragma message("Bad Visual Studio version: (" QUOTE(_MSC_VER) " " QUOTE(_MSC_FULL_VER) " " QUOTE(_MSC_BUILD) ").")
#error "Bad Visual Studio version"
#endif
```

I used Wayback Machine to access the [Visual Studio download page from June 1st, 2021](https://web.archive.org/web/20210601035621/https://visualstudio.microsoft.com/downloads/), downloaded and installed it with the required Windows SDK version.

![Visual Studio install page](01_vs_install.png)

## Cloning and building The Forge

Before we try to do anything with the framework, let's make sure we can actually run it. First clone the repository:

```shell
git clone https://github.com/ConfettiFX/The-Forge.git
```

To run the examples, you have to download the assets used by them by running the `PRE_BUILD.bat` file. Then open `The-Forge\Examples_3\Unit_Tests\PC Visual Studio 2019\Unit_Tests.sln`. Set `01_Transformations` as the Startup Project in the Solution Explorer.

![Select startup project](02_select_example.png)

Then launch by clicking on `Local Windows Debugger`. If everything went well the following window with the solar system should open.

![Solar system demo](03_demo.png)

Now we're ready to create our triangle project.

## Creating a new project

In Solution Explorer, create a new project by right clicking `Examples`, and select `Add/New Project`.

![Create new project](04_new_project.png)

Select the `Empty Project` template, name the project `triangle` and put it in the `PC Visual Studio 2019` directory.

![New project](05_new_project_2.png)

Now let's add a source file. Right click on `Source Files` and select `Add/New Item`.

![New source file](06_add_source.png)

Create a new directory `triangle` under `Unit_Tests\src` and put your source file there. I named mine `triangle.cpp`.

![Source file location](07_source_file.png)

Put the following code in `triangle.cpp`. Adjust the include path if you need to.

```cpp
#include "../../../../Common_3/Application/Interfaces/IApp.h"

class Triangle : public IApp
{
    bool Init() override
    {
        return true;
    }

    void Exit() override
    {
        
    }

    bool Load(ReloadDesc* pReloadDesc) override
    {
        (void)pReloadDesc;
        return true;
    }

    void Unload(ReloadDesc* pReloadDesc) override
    {
        (void)pReloadDesc;
    }

    void Update(float deltaTime) override
    {
        (void)deltaTime;
    }

    void Draw() override
    {
        
    }

    const char* GetName() override
    {
        return "Triangle";
    }
};


DEFINE_APPLICATION_MAIN(Triangle)
```

We'll need to configure our new project. Right click on the project and select `Properties`.

In `Configuration Properties/General`, set `Output Directory` to:

```
$(SolutionDir)$(Platform)\$(Configuration)\$(ProjectName)\
```

And set  `Intermediate Directory` to:

```
$(SolutionDir)\$(Platform)\$(Configuration)\Intermediate\$(ProjectName)\
```

Make sure that the correct version of Visual Studio and Windows SDK are selected.

![General](08_general.png)

In `Configuration Properties/Debugging`, set `Working Directory` to `$(OutDir)`

![Debugging](09_debugging.png)

In `Configuration Properties/VC++ Directories`, set `Library Directories` to:

```
$(SolutionDir)\$(Platform)\$(Configuration);$(LibraryPath)
```

In `Configuration Properties/Linker/Input`, set `Additional Dependencies` to:

```
Xinput9_1_0.lib;ws2_32.lib;Renderer.lib;OS.lib;%(AdditionalDependencies)
```

![Linker input](10_linker_input.png)

We also need to set the build dependencies. Right click on the project, select `Build Dependencies/Project Dependencies`.

![Project dependencies](11_build_deps.png)

Select `OS` and `Renderer`

![Project dependencies](12_build_deps_2.png)

Now, this error will appear if we try to build:

![Agility SDK](13_agility.png)

The examples define this in `Examples_3\Build_Props\VS\TF_Shared.props`.

```xml
<PreprocessorDefinitions>
  D3D12_AGILITY_SDK=1;
  D3D12_AGILITY_SDK_VERSION=715;
  %(PreprocessorDefinitions)
</PreprocessorDefinitions>
```

We're going to do the same. Edit `triangle.vcxproj` and add these lines just before the last `</Project>` line. Adjust the path to `TF_Shared.props` if you need to.

```xml
  <ImportGroup Label="PropertySheets">
    <Import Project="..\..\..\..\Examples_3\Build_Props\VS\TF_Shared.props" />
  </ImportGroup>
```

Reload the project and launch. If everything went well a window should pop up.

![Window](14_window.png)

Now we can start writing the code for our triangle.

## Rendering a triangle

The Forge uses FSL (Forge Shading Language). We first need to integrate FSL by right clicking the project, select `Build Dependencies/Build Customizations` and choose the `fsl.target` file.

Now we can start writing the shaders for our triangle. First create a new filter for our shaders and name it `FSLShaders`

![Filter](15_filter.png)

Then right click on `FSLShaders` and select `Add/New Item`. 

![Filter item](16_filter_item.png)

Create a new directory `Shaders` under `Unit_Tests\src\triangle` and place your shaders there, making sure it ends with `.fsl`. I named mine `triangle.vert.fsl` and `triangle.frag.fsl`. Also add a new file `shaders.list` in the same directory.

![Shader dir](17_shader_dir.png)

Put the following code in `triangle.vert.fsl`. Refer to the [FSL Programming Guide](https://github.com/ConfettiFX/The-Forge/wiki/FSL-Programming-Guide) for more details.

```
STRUCT(VSInput)
{
    DATA(float4, Position, POSITION);
};

STRUCT(VSOutput)
{
    DATA(float4, Position, SV_Position);
};

ROOT_SIGNATURE(DefaultRootSignature)
VSOutput VS_MAIN(VSInput In)
{
    INIT_MAIN;
    VSOutput Out;

    Out.Position = float4(In.Position.x, In.Position.y, In.Position.z, In.Position.w);
    RETURN(Out);
}
```

Put this in `triangle.frag.fsl`

```
STRUCT(VSOutput)
{
    DATA(float4, Position, SV_Position);
};

ROOT_SIGNATURE(DefaultRootSignature)
float4 PS_MAIN(VSOutput In)
{
    INIT_MAIN;
    float4 Out;

    Out = float4(1.0, 0.0, 0.0, 1.0);
    RETURN(Out);
}
```

And put this in `shaders.list`. Adjust the include path if needed.

```
#include "../../../../../Common_3/Graphics/FSL/defaults.h"

#rootsig default.rootsig
#end

#rootsig compute.rootsig
#end

#vert triangle.vert
#include "triangle.vert.fsl"
#end

#frag triangle.frag
#include "triangle.frag.fsl"
#end
```

Right click on the project and select `Properties`. In `Configuration Properties/FSLShader`:
- set `Language` to `DIRECT3D12`
- set `OutDir` to `$(OutDir)Shaders`
- set `BinaryOutDir` to `$(OutDir)CompiledShaders`
- set `Compile shader after generation` to `Yes`

![Shader properties](18_shader_props.png)

Now put this in `triangle.cpp`. If you've used Vulkan then this should look pretty familiar.

```cpp
// Interfaces
#include "../../../../Common_3/Application/Interfaces/IApp.h"

#include "../../../../Common_3/Utilities/RingBuffer.h"

// Renderer
#include "../../../../Common_3/Graphics/Interfaces/IGraphics.h"
#include "../../../../Common_3/Resources/ResourceLoader/Interfaces/IResourceLoader.h"

// FSL
#include "../../../../Common_3/Graphics/FSL/defaults.h"

Renderer*  pRenderer = NULL;
Queue*     pGraphicsQueue = NULL;
GpuCmdRing gGraphicsCmdRing = {};

SwapChain* pSwapChain = NULL;
Semaphore* pImageAcquiredSemaphore = NULL;

Shader* pTriangleShader = NULL;
Buffer* pTriangleVertexBuffer = NULL;
Pipeline* pTrianglePipeline = NULL;

const uint32_t gFramesInFlight = 2;

const float gTrianglePoints[] = {
    -0.5f, -0.5f, 0.0f, 1.0f,
    0.5f, -0.5f, 0.0f, 1.0f,
    0.0f, 0.5f, 0.0f, 1.0f,
};

class Triangle : public IApp
{
    bool Init() override
    {
        RendererDesc settings;
        memset(&settings, 0, sizeof(settings));
        initGPUConfiguration(settings.pExtendedSettings);
        initRenderer(GetName(), &settings, &pRenderer);

        if (!pRenderer)
        {
            ShowUnsupportedMessage("Failed to Initialize renderer!");
            return false;
        }
        setupGPUConfigurationPlatformParameters(pRenderer, settings.pExtendedSettings);

        QueueDesc queueDesc = {};
        queueDesc.mType = QUEUE_TYPE_GRAPHICS;
        queueDesc.mFlag = QUEUE_FLAG_INIT_MICROPROFILE;
        initQueue(pRenderer, &queueDesc, &pGraphicsQueue);

        GpuCmdRingDesc cmdRingDesc = {};
        cmdRingDesc.pQueue = pGraphicsQueue;
        cmdRingDesc.mPoolCount = gFramesInFlight;
        cmdRingDesc.mCmdPerPoolCount = 1;
        cmdRingDesc.mAddSyncPrimitives = true;
        initGpuCmdRing(pRenderer, &cmdRingDesc, &gGraphicsCmdRing);

        initSemaphore(pRenderer, &pImageAcquiredSemaphore);

        initResourceLoaderInterface(pRenderer);

        RootSignatureDesc rootDesc = {};
        INIT_RS_DESC(rootDesc, "default.rootsig", "compute.rootsig");
        initRootSignature(pRenderer, &rootDesc);

        uint64_t triangleDataSize = 4 * 3 * sizeof(float);
        BufferLoadDesc triangleVbDesc = {};
        triangleVbDesc.mDesc.mDescriptors = DESCRIPTOR_TYPE_VERTEX_BUFFER;
        triangleVbDesc.mDesc.mMemoryUsage = RESOURCE_MEMORY_USAGE_GPU_ONLY;
        triangleVbDesc.mDesc.mSize = triangleDataSize;
        triangleVbDesc.pData = gTrianglePoints;
        triangleVbDesc.ppBuffer = &pTriangleVertexBuffer;
        addResource(&triangleVbDesc, NULL);

        waitForAllResourceLoads();

        return true;
    }

    void Exit() override
    { 
        removeResource(pTriangleVertexBuffer);

        exitGpuCmdRing(pRenderer, &gGraphicsCmdRing);
        exitSemaphore(pRenderer, pImageAcquiredSemaphore);

        exitRootSignature(pRenderer);
        exitResourceLoaderInterface(pRenderer);

        exitQueue(pRenderer, pGraphicsQueue);

        exitRenderer(pRenderer);
        exitGPUConfiguration();
        pRenderer = NULL;
    }
    
    bool Load(ReloadDesc* pReloadDesc) override
    {
        if (pReloadDesc->mType & RELOAD_TYPE_SHADER)
        {
            addShaders();
        }

        if (pReloadDesc->mType & (RELOAD_TYPE_RESIZE | RELOAD_TYPE_RENDERTARGET))
        {
            if (!addSwapChain())
            {
                return false;
            }
        }

        if (pReloadDesc->mType & (RELOAD_TYPE_SHADER | RELOAD_TYPE_RENDERTARGET))
        {
            addPipelines();
        }

        return true;
    }

    void Unload(ReloadDesc* pReloadDesc) override
    {
        waitQueueIdle(pGraphicsQueue);

        if (pReloadDesc->mType & (RELOAD_TYPE_SHADER | RELOAD_TYPE_RENDERTARGET))
        {
            removePipelines();
        }

        if (pReloadDesc->mType & RELOAD_TYPE_SHADER)
        {
            removeShaders();
        }

        if (pReloadDesc->mType & (RELOAD_TYPE_RESIZE | RELOAD_TYPE_RENDERTARGET))
        {
            removeSwapChain(pRenderer, pSwapChain);
        }
    }

    void Update(float deltaTime) override
    {
        (void)deltaTime;
    }

    void Draw() override
    {
        if ((bool)pSwapChain->mEnableVsync != mSettings.mVSyncEnabled)
        {
            waitQueueIdle(pGraphicsQueue);
            ::toggleVSync(pRenderer, &pSwapChain);
        }

        uint32_t swapchainImageIndex;
        acquireNextImage(pRenderer, pSwapChain, pImageAcquiredSemaphore, NULL, &swapchainImageIndex);

        RenderTarget*     pRenderTarget = pSwapChain->ppRenderTargets[swapchainImageIndex];
        GpuCmdRingElement elem = getNextGpuCmdRingElement(&gGraphicsCmdRing, true, 1);

        FenceStatus fenceStatus;
        getFenceStatus(pRenderer, elem.pFence, &fenceStatus);
        if (fenceStatus == FENCE_STATUS_INCOMPLETE)
        {
            waitForFences(pRenderer, 1, &elem.pFence);
        }

        resetCmdPool(pRenderer, elem.pCmdPool);

        Cmd* cmd = elem.pCmds[0];
        beginCmd(cmd);

        RenderTargetBarrier barriers[] = {
            {pRenderTarget, RESOURCE_STATE_PRESENT, RESOURCE_STATE_RENDER_TARGET},
        };
        cmdResourceBarrier(cmd, 0, NULL, 0, NULL, 1, barriers);

        BindRenderTargetsDesc bindRenderTargets = {};
        bindRenderTargets.mRenderTargetCount = 1;
        bindRenderTargets.mRenderTargets[0] = {pRenderTarget, LOAD_ACTION_CLEAR };
        cmdBindRenderTargets(cmd, &bindRenderTargets);
        cmdSetViewport(cmd, 0.0f, 0.0f, (float)pRenderTarget->mWidth, (float)pRenderTarget->mHeight, 0.0f, 1.0f);
        cmdSetScissor(cmd, 0, 0, pRenderTarget->mWidth, pRenderTarget->mHeight);

        const uint32_t triangleVbStride = sizeof(float) * 4;
        cmdBindPipeline(cmd, pTrianglePipeline);
        cmdBindVertexBuffer(cmd, 1, &pTriangleVertexBuffer, &triangleVbStride, NULL);
        cmdDraw(cmd, 3, 0);

        barriers[0] = { pRenderTarget, RESOURCE_STATE_RENDER_TARGET, RESOURCE_STATE_PRESENT };
        cmdResourceBarrier(cmd, 0, NULL, 0, NULL, 1, barriers);

        endCmd(cmd);

        Semaphore* waitSemaphores[] = { pImageAcquiredSemaphore };

        QueueSubmitDesc submitDesc = {};
        submitDesc.mCmdCount = 1;
        submitDesc.mSignalSemaphoreCount = 1;
        submitDesc.mWaitSemaphoreCount = TF_ARRAY_COUNT(waitSemaphores);
        submitDesc.ppCmds = &cmd;
        submitDesc.ppSignalSemaphores = &elem.pSemaphore;
        submitDesc.ppWaitSemaphores = waitSemaphores;
        submitDesc.pSignalFence = elem.pFence;
        queueSubmit(pGraphicsQueue, &submitDesc);

        QueuePresentDesc presentDesc = {};
        presentDesc.mIndex = (uint8_t)swapchainImageIndex;
        presentDesc.mWaitSemaphoreCount = 1;
        presentDesc.pSwapChain = pSwapChain;
        presentDesc.ppWaitSemaphores = &elem.pSemaphore;
        presentDesc.mSubmitDone = true;

        queuePresent(pGraphicsQueue, &presentDesc);
    }

    const char* GetName() override {
        return "Triangle";
    }

    void addPipelines()
    {
        VertexLayout triangleVertexLayout = {};
        triangleVertexLayout.mBindingCount = 1;
        triangleVertexLayout.mBindings[0].mStride = sizeof(float4);
        triangleVertexLayout.mAttribCount = 1;
        triangleVertexLayout.mAttribs[0].mSemantic = SEMANTIC_POSITION;
        triangleVertexLayout.mAttribs[0].mFormat = TinyImageFormat_R32G32B32A32_SFLOAT;
        triangleVertexLayout.mAttribs[0].mBinding = 0;
        triangleVertexLayout.mAttribs[0].mLocation = 0;
        triangleVertexLayout.mAttribs[0].mOffset = 0;

        RasterizerStateDesc rasterizerStateDesc = {};
        rasterizerStateDesc.mCullMode = CULL_MODE_NONE;

        PipelineDesc desc = {};
        desc.mType = PIPELINE_TYPE_GRAPHICS;

        GraphicsPipelineDesc& pipelineSettings = desc.mGraphicsDesc;
        pipelineSettings.mPrimitiveTopo = PRIMITIVE_TOPO_TRI_LIST;
        pipelineSettings.mRenderTargetCount = 1;
        pipelineSettings.pDepthState = NULL;
        pipelineSettings.pColorFormats = &pSwapChain->ppRenderTargets[0]->mFormat;
        pipelineSettings.mSampleCount = pSwapChain->ppRenderTargets[0]->mSampleCount;
        pipelineSettings.mSampleQuality = pSwapChain->ppRenderTargets[0]->mSampleQuality;
        pipelineSettings.pShaderProgram = pTriangleShader;
        pipelineSettings.pVertexLayout = &triangleVertexLayout;
        pipelineSettings.pRasterizerState = &rasterizerStateDesc;
        addPipeline(pRenderer, &desc, &pTrianglePipeline);
    }

    void removePipelines()
    {
        removePipeline(pRenderer, pTrianglePipeline);
    }

    void addShaders()
    {
        ShaderLoadDesc triangleShader = {};
        triangleShader.mVert.pFileName = "triangle.vert";
        triangleShader.mFrag.pFileName = "triangle.frag";

        addShader(pRenderer, &triangleShader, &pTriangleShader);
    }

    void removeShaders()
    {
        removeShader(pRenderer, pTriangleShader);   
    }

    bool addSwapChain()
    {
        SwapChainDesc swapChainDesc = {};
        swapChainDesc.mWindowHandle = pWindow->handle;
        swapChainDesc.mPresentQueueCount = 1;
        swapChainDesc.ppPresentQueues = &pGraphicsQueue;
        swapChainDesc.mWidth = mSettings.mWidth;
        swapChainDesc.mHeight = mSettings.mHeight;
        swapChainDesc.mImageCount = getRecommendedSwapchainImageCount(pRenderer, &pWindow->handle);
        swapChainDesc.mColorFormat = getSupportedSwapchainFormat(pRenderer, &swapChainDesc, COLOR_SPACE_SDR_SRGB);
        swapChainDesc.mColorSpace = COLOR_SPACE_SDR_SRGB;
        swapChainDesc.mEnableVsync = mSettings.mVSyncEnabled;
        ::addSwapChain(pRenderer, &swapChainDesc, &pSwapChain);

        return pSwapChain != NULL;
    }
};

DEFINE_APPLICATION_MAIN(Triangle)
```

Then launch. If everything went well a red triangle should appear.

![Triangle](19_triangle.png)
