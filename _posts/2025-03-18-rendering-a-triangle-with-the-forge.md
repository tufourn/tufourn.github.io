---
title: Rendering a triangle with The Forge
description: Guide to setting up a minimal example with The Forge Framework.
date: 2025-03-18 00:21 -0400
categories: [programming, graphics programming]
tags: [graphics programming, the forge]
media_subpath: /assets/2025-03-18-rendering-a-triangle-with-the-forge
---

After implementing my own renderer from scratch with Vulkan, I think it's time to move on to using something a bit higher level. I decided to try out [The Forge](https://github.com/ConfettiFX/The-Forge) by ConfettiFX. Unfortunately, the documentation is rather lacking, and the examples are really the only resources available for learning it.

There is a [guide on building The Forge by LCG](https://rendering-techniques.learn-computer-graphics.com/doc/getting-started/building.html), but it's outdated and didn't work for me. I managed to get a triangle rendered on the screen after a few hours of tinkering. In this post, I'll share the steps I took to help others like me get started with the framework.

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

In Solution Explorer, create a new project by right clicking `Examples`, and select `Add->New Project`.

![Create new project](04_new_project.png)

Select the `Empty Project` template, name the project `triangle` and put it in the `PC Visual Studio 2019` directory.

![New project](05_new_project_2.png)

Now let's add a source file. Right click on `Source Files` and select `Add->New Item`.

![New source file](06_add_source.png)

Create a new directory `triangle` under `Unit_Tests\src` and put your source file there. I named mine `triangle.cpp`.

![Source file location](07_source_file.png)

Put the following code in `triangle.cpp`.

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

Make sure that the correct version of Visual Studio and Windows SDK is selected.

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

The example defines this in `Examples_3\Build_Props\VS\TF_Shared.props`.
```xml
<PreprocessorDefinitions>
  D3D12_AGILITY_SDK=1;
  D3D12_AGILITY_SDK_VERSION=715;
  %(PreprocessorDefinitions)
</PreprocessorDefinitions>
```

We're going to do the same. Open `triangle.vcxproj` and add these lines just before the last `</Project>` line. Adjust the path to `TF_Shared.props` if you need to.
```xml
  <ImportGroup Label="PropertySheets">
    <Import Project="..\..\..\..\Examples_3\Build_Props\VS\TF_Shared.props" />
  </ImportGroup>
```

Reload the project and launch. If everything went well a window should pop up.

![Window](14_window.png)

Now we can start writing the code for our triangle.

## Rendering a triangle
//todo
