---
title: What Happened after Press Play Button in Editor
date: 2022-04-17 14:30:00 +0800
categories: [Unreal Engine, Editor]
tags: [learning ue]
img_path: /img/2022-04-17-What-Happened-After-Press-Play-Button-In-Editor/
---

This post mainly talk about the underlying logic of Play button in UE Editor.

> All content of this post is based on Unreal Engine 4.27.2 (Binary version from Epic Launcher).
{: .prompt-tip }

## Underlying Callback Function
Searching the tooltips of Play button in Editor source code, will lead you to the relative code.

![Play Button Tooltips](Play-Button-Tooltips.png) {: width="415" height="146" }
_Hover cursor on the Play button_

In RegisterCommands() function of *DebuggerCommands.cpp*, a **UI_Command** named **PlayInViewPort** takes above tooltips. Code like follows:

````C++
... line:334
	UI_COMMAND(PlayInViewport, "Selected Viewport", "Play this level in the active level editor viewport", EUserInterfaceActionType::Check, FInputChord());
...
````
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp"}

Following the command **PlayInViewport**, it's easy to find out that this commands is mapping with callback in the same file. 

````C++
... line: 421
	ActionList.MapAction(Commands.PlayInViewport,
		FExecuteAction::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInViewport_Clicked),
		FCanExecuteAction::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInViewport_CanExecute),
		FIsActionChecked::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInModeIsChecked, PlayMode_InViewPort),
		FIsActionButtonVisible::CreateStatic(&FInternalPlayWorldCommandCallbacks::CanShowNonPlayWorldOnlyActions)
	);
...
````
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp"}

As the mapping logic, when Play button is clicked, the **PlayInViewport_Clicked** function will be triggered. This function should be the key to start a play session in editor environment.

````C++
... line: 1765
void FInternalPlayWorldCommandCallbacks::PlayInViewport_Clicked()
{
	FLevelEditorModule& LevelEditorModule = FModuleManager::GetModuleChecked<FLevelEditorModule>(TEXT("LevelEditor"));

	/** Set PlayInViewPort as the last executed play command */
	const FPlayWorldCommands& Commands = FPlayWorldCommands::Get();

	SetLastExecutedPlayMode(PlayMode_InViewPort);

	RecordLastExecutedPlayMode();

	TSharedPtr<IAssetViewport> ActiveLevelViewport = LevelEditorModule.GetFirstActiveViewport();

	const bool bAtPlayerStart = (GetPlayModeLocation() == PlayLocation_DefaultPlayerStart);

	FRequestPlaySessionParams SessionParams;

	// Make sure we can find a path to the view port.  This will fail in cases where the view port widget
	// is in a backgrounded tab, etc.  We can't currently support starting PIE in a backgrounded tab
	// due to how PIE manages focus and requires event forwarding from the application.
	if (ActiveLevelViewport.IsValid() && FSlateApplication::Get().FindWidgetWindow(ActiveLevelViewport->AsWidget()).IsValid())
	{
		SessionParams.DestinationSlateViewport = ActiveLevelViewport;
		if (!bAtPlayerStart)
		{
			// Start the player where the camera is if not forcing from player start
			SessionParams.StartLocation = ActiveLevelViewport->GetAssetViewportClient().GetViewLocation();
			SessionParams.StartRotation = ActiveLevelViewport->GetAssetViewportClient().GetViewRotation();
		}
	}

	if (!HasPlayWorld())
	{
		// If there is an active level view port, play the game in it, otherwise make a new window.
		GUnrealEd->RequestPlaySession(SessionParams);
	}
	else
	{
		// There is already a play world active which means simulate in editor is happening
		// Toggle to PIE
		check(!GIsPlayInEditorWorld);
		GUnrealEd->RequestToggleBetweenPIEandSIE();
	}
}
...
````
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp"}

At the beginning, this function record the current mode in **FEngineAnalytics** for **RepeatLastPlay** (another play mode like **PlayInViewport**) usage. And then, it constructs a *SessionParams* to request a play session from editor instance. For now, it only takes active viewport and start orientation in count.

## Start Play Session
In common case, the **RequestPlaySession** function of class **UEditorEngine** class will be invoked.
````C++
... line: 825
void UEditorEngine::RequestPlaySession(const FRequestPlaySessionParams& InParams)
{
	// Store our Request to be operated on next Tick.
	PlaySessionRequest = InParams;

	// If they don't want to use a specific set of Editor Play Settings, fall back to the CDO.
	if (!PlaySessionRequest->EditorPlaySettings)
	{
		PlaySessionRequest->EditorPlaySettings = GetMutableDefault<ULevelEditorPlaySettings>();
	}

	// Now we duplicate their Editor Play Settings so that we can mutate it as part of startup
	// to help rule out invalid configuration combinations.
	FObjectDuplicationParameters DuplicationParams(PlaySessionRequest->EditorPlaySettings, GetTransientPackage());
	// Kept alive by AddReferencedObjects
	PlaySessionRequest->EditorPlaySettings = CastChecked<ULevelEditorPlaySettings>(StaticDuplicateObjectEx(DuplicationParams));


	// ToDo: Allow the CDO for the Game Instance to modify the settings after we copy them
	// so that they can validate user settings before attempting a launch.

	// Play Sessions can use the Game Mode to determine the default Player Start position
	// or the start position can be overridden by the incoming launch arguments
	PRAGMA_DISABLE_DEPRECATION_WARNINGS
	GIsPIEUsingPlayerStart = !InParams.StartLocation.IsSet();
	PRAGMA_ENABLE_DEPRECATION_WARNINGS

	if (InParams.SessionDestination == EPlaySessionDestinationType::Launcher)
	{
		check(InParams.LauncherTargetDevice.IsSet());
	}
}
... 
````
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\PlayLevel.cpp"}

According to the code above, this function does some preparations instead of creating play session directly. The major part is duplicating the *EditorPlaySettings* for play session. And the *PlaySessionRequest* will be used in next tick, in the other word, the creation of play session is delayed by one frame.

**Tick** function of **UEditorEngine** will check the *PlaySessionRequest* variable every frame for any play session pending to start.
````C++
... line: 1625
	// Kick off a Play Session request if one was queued up during the last frame.
	if (PlaySessionRequest.IsSet())
	{
		StartQueuedPlaySessionRequest();
	}
	else if( bIsToggleBetweenPIEandSIEQueued )
	{
		ToggleBetweenPIEandSIE();
	}
...
````
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\EditorEngine.cpp"}


