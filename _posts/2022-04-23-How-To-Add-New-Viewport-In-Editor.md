---
title: How to Add New Viewport in Editor
date: 2022-04-23 10:16:24 +0800
categories: [Unreal Engine, Editor]
tags: [how to]
img_path: /img/2022-04-23-How-To-Add-New-Viewport-In-Editor/
---

This article will introduce how to add a customized viewport of current level in Unreal Editor.

> All content are based on Unreal Engine 4.27.2 (binary version from Epic Game Launcher). Other version may have some different in source code.
{: .prompt-tip}

## How is Editor Viewport Rendered?
Before creating new viewport, let's have a look at the existing viewports and figure out how they are rendered. This part helps to get the idea of which classes are needed when implementing customize viewport.

![Viewport-UML](UE-Viewport-UML.jpg)
_UML Diagram of Viewport Relative Classes_

The UML diagram above contains the major classes which are involved during viewport rendering. For Unreal users, the most familiar one is the main level viewport. 

![Main-Level-Viewport](UE-Level-Viewport.png)
_The Main Level Viewport_

This viewport is an instance of **SLevelViewport** class, one of the derived classes of **SEditorViewport**. As the UML diagram shows, **SEditorViewport** references to all necessary classes for rendering viewport content. It provides a method named *Invalidate()* which will trigger a redraw in next frame. **SViewport** instance will be instantiated as a child element of **SEditorViewport**, as the result, it will be redraw as well. At the same time, *InvalidateDisplay* function of **FViewport** class will also be invoked.
```
@line: 5352
if ( bInvalidateHitProxies )
{
    // Invalidate hit proxies and display pixels.
    Viewport->Invalidate();
}
else
{
    // Invalidate only display pixels.
    Viewport->InvalidateDisplay();
}
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\EditorViewportClient.cpp'}

*OnPaint* function of class **SViewport** will be invoked if viewport is marked as invalid previously. *OnDrawViewport* function of relative **FSceneViewport** instance also be called here. 
```
@line:122
TSharedPtr<ISlateViewport> ViewportInterfacePin = ViewportInterface.Pin();

// Tell the interface that we are drawing.
if (ViewportInterfacePin.IsValid())
{
    ViewportInterfacePin->OnDrawViewport( AllottedGeometry, MyCullingRect, OutDrawElements, LayerId, InWidgetStyle, bParentEnabled );
}
```
{: file='\UE_4.27\Engine\Source\Runtime\Slate\Private\Widgets\SViewport.cpp'}

*InvalidateDisplay* function of class **FSceneViewport** invokes corresponding **FViewportClient** instance's *RedrawRequested* function.
```
@line: 1534
void FSceneViewport::InvalidateDisplay()
{
	// Dirty the viewport.  It will be redrawn next time the editor ticks.
	if( ViewportClient != NULL )
	{
		ViewportClient->RedrawRequested( this );
	}
}
```
{: file='\UE_4.27\Engine\Source\Runtime\Engine\Private\Slate\SceneViewport.cpp'}

**FViewportClient** class will invoke relative **FViewport** instance's *Draw* function when *RedrawRequested* function is called. 
```
@line:798
virtual void RedrawRequested(FViewport* Viewport) { Viewport->Draw(); }
```
{: file='\UE_4.27\Engine\Source\Runtime\Engine\Public\UnrealClient.h'}

*Draw* function of class **FViewportClient** will be finally called.
```
@line:1615 FViewport::Draw
ViewportClient->Draw(this, &Canvas);
```
{: file='\UE_4.27\Engine\Source\Runtime\Engine\Private\UnrealClient.cpp'}

Class **FEditorViewportClient** is derived from **FViewportClient**, overrides the *Draw* function for rendering the scene. In the implementation of **FEditorViewportClient**, *Draw* function construct a **FSceneViewFamilyContext** instance for viewport displaying and rendering the content to given canvas.

## Add New Viewport
According to previous chapter, the two major classes for adding new viewport are **SEditorViewport** and **FEditorViewportClient**. Following content will show how to add a new viewport into Unreal editor.

### Add New Dockable Tab
Before actual coding, add a dockable tab as the container of new viewport is a good start. Since Unreal editor provides convenient method to create new editor window.
First, open up the *Plugins* tab. Click the *New Plugin* button at the right buttom corner.
![UE-Plugin-Tab](UE-Plugins-Tab.png)
_Plugins Tab_

In new opened window, select the *Editor Standalone Window* template and fill in the new plugin new. Then click the *Create Plugin* button and wait until progress finished.
![UE-New-Plugin](UE-New-Plugin-Dialog.png)
_New Plugin Dialog_

After the code compiled, you can find the button for open new window in *Window* menu.
![UE-Open-Window-Button](UE-Open-Window-Button.png)
_Open Window Button_

### Create Editor Viewport Client
As mentioned before, we need a new class dervided from **FEditorViewportClient**. Add new header and source file and type following code.
```
#pragma once

#include "CoreMinimal.h"

class SSlateViewport;

class SLATEWINDOW_API FSlateViewportClient : public FEditorViewportClient
{
public:

	FSlateViewportClient(const TSharedPtr<SSlateViewport>& InSlateViewport);
};
```
{: file='SlateViewportClient.h'}

```
#include "SlateViewportClient.h"

FSlateViewportClient::FSlateViewportClient(const TSharedPtr<SSlateViewport>& InSlateViewport)
	: FEditorViewportClient(&GLevelEditorModeTools(), nullptr, StaticCastSharedPtr<SEditorViewport>(InSlateViewport))
{
}
```
{: file='SlateViewportClient.cpp'}

Actually this class doesn't do anything except convert passed in **SSlateViewport** instance to **SEditorViewport**.
> This client can be extended with customization functions.
{: .prompt-tip}

### Create Editor Viewport
The other class should dervide from **SEditorViewport** class. Add new files and following codes.
```
#pragma once

#include "CoreMinimal.h"
#include "SEditorViewport.h"

class FSlateViewportClient;

class SLATEWINDOW_API SSlateViewport : public SEditorViewport
{
public:
	SLATE_BEGIN_ARGS(SSlateViewport)
	{}
	SLATE_END_ARGS()

	void Construct(const FArguments& InArgs);

protected:

	void ConstructSlateViewportClient();

	virtual TSharedRef<FEditorViewportClient> MakeEditorViewportClient() override;

private:

	TSharedPtr<FSlateViewportClient> SlateViewportClient;
};
```
{: file='SSlateViewport.h'}

```
#include "SSlateViewport.h"

#include "SlateViewportClient.h"

void SSlateViewport::Construct(const FArguments& InArgs)
{
	ConstructSlateViewportClient();

	SEditorViewport::Construct(SEditorViewport::FArguments());
}

void SSlateViewport::ConstructSlateViewportClient()
{
	if (!SlateViewportClient.IsValid())
	{
		SlateViewportClient = MakeShareable(new FSlateViewportClient(SharedThis(this)));
	}

	// Setup viewport client
	FLevelEditorViewportInstanceSettings ViewportInstanceSettings;
	ViewportInstanceSettings.ViewportType = LVT_Perspective;
	ViewportInstanceSettings.PerspViewModeIndex = VMI_Lit;
	ViewportInstanceSettings.OrthoViewModeIndex = VMI_BrushWireframe;
	ViewportInstanceSettings.bIsRealtime = false;

	FEngineShowFlags ViewportShowFlags(ESFIM_Game);
	ApplyViewMode(ViewportInstanceSettings.PerspViewModeIndex, true, ViewportShowFlags);
	ViewportShowFlags.SetSnap(1);

	SlateViewportClient->ViewportType = ViewportInstanceSettings.ViewportType;
	SlateViewportClient->EngineShowFlags = ViewportShowFlags;
	SlateViewportClient->LastEngineShowFlags = ViewportShowFlags;
	SlateViewportClient->CurrentBufferVisualizationMode = ViewportInstanceSettings.BufferVisualizationMode;
	SlateViewportClient->CurrentRayTracingDebugVisualizationMode = ViewportInstanceSettings.RayTracingDebugVisualizationMode;
	SlateViewportClient->ExposureSettings = ViewportInstanceSettings.ExposureSettings;
	SlateViewportClient->SetViewLocation( EditorViewportDefs::DefaultPerspectiveViewLocation );
	SlateViewportClient->SetViewRotation( EditorViewportDefs::DefaultPerspectiveViewRotation );
}

TSharedRef<FEditorViewportClient> SSlateViewport::MakeEditorViewportClient()
{
	return SlateViewportClient.ToSharedRef();
}
```
{: file='SSlateViewport.cpp'}

This class is in charge of initializing viewport client instance.

### Show New Viewport
The final step is embedding new viewport to plugin window. Open up the source code file named as *PLUGIN_NAME*.cpp, and find the *OnSpawnPluginTab* function. Change the code as below.
```
TSharedRef<SDockTab> FSlateWindowModule::OnSpawnPluginTab(const FSpawnTabArgs& SpawnTabArgs)
{
	return SNew(SDockTab)
		.TabRole(ETabRole::NomadTab)
		[
			// Embed new viewport
			SNew(SSlateViewport)
		];
}
```
{: file='SlateWindow.cpp'}

Compile the code and run the editor to check the result.
![UE-New-Viewport](UE-New-Viewport.png)
_New Viewport_
