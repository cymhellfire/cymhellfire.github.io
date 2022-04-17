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

In **RegisterCommands()** function of *DebuggerCommands.cpp*, a **UI_Command** named **PlayInViewPort** takes above tooltips.

```
 @line:334
	UI_COMMAND(PlayInViewport, "Selected Viewport", "Play this level in the active level editor viewport", EUserInterfaceActionType::Check, FInputChord());
```
{: file="\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp"}

Following the command **PlayInViewport**, it's easy to find out that this commands is mapping with callback functions in the same file. 

```
 @ line: 421
	ActionList.MapAction(Commands.PlayInViewport,
		FExecuteAction::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInViewport_Clicked),
		FCanExecuteAction::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInViewport_CanExecute),
		FIsActionChecked::CreateStatic(&FInternalPlayWorldCommandCallbacks::PlayInModeIsChecked, PlayMode_InViewPort),
		FIsActionButtonVisible::CreateStatic(&FInternalPlayWorldCommandCallbacks::CanShowNonPlayWorldOnlyActions)
	);
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp'}

As the mapping logic, when Play button is clicked, the **PlayInViewport_Clicked** function will be triggered. This function should be the key to start a play session in editor environment.

```
 @ line: 1765
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
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\Kismet2\DebuggerCommands.cpp'}

At the beginning, this function record the current mode in **FEngineAnalytics** for **RepeatLastPlay** (another play mode like **PlayInViewport**) usage. And then, it constructs a *SessionParams* to request a play session from editor instance. For now, it only takes active viewport and start orientation in count.

## Start Play Session
In common case, the **RequestPlaySession** function of class **UEditorEngine** class will be invoked.
```
 @ line: 825
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
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\PlayLevel.cpp'}

According to the code above, this function does some preparations instead of creating play session directly. The major part is duplicating the *EditorPlaySettings* for play session. And the *PlaySessionRequest* will be used in next tick, in the other word, the creation of play session is delayed by one frame.

**Tick** function of **UEditorEngine** will check the *PlaySessionRequest* variable every frame for any play session pending to start.
```
 @ line: 1625
	// Kick off a Play Session request if one was queued up during the last frame.
	if (PlaySessionRequest.IsSet())
	{
		StartQueuedPlaySessionRequest();
	}
	else if( bIsToggleBetweenPIEandSIEQueued )
	{
		ToggleBetweenPIEandSIE();
	}
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\EditorEngine.cpp'}

Function **StartQueuedPlaySessionRequest** is pretty simple, it invoke **StartQueuedPlaySessionRequestImpl** and clear the *PlaySessionRequest* value to avoid create play session repeatly.
```
 @ line: 1007
void UEditorEngine::StartQueuedPlaySessionRequest()
{
	if (!PlaySessionRequest.IsSet())
	{
		UE_LOG(LogPlayLevel, Warning, TEXT("StartQueuedPlaySessionRequest() called whith no request queued. Ignoring..."));
		return;
	}

	StartQueuedPlaySessionRequestImpl();

	// Ensure the request is always reset after an attempt (which may fail)
	// so that we don't get stuck in an infinite loop of start attempts.
	PlaySessionRequest.Reset();
}
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\PlayLevel.cpp'}

**StartQueuedPlaySessionRequestImpl** function gathering neccessary information for variable *PlayInEditorSessionInfo* and decide whether the new session run in a separate process. And finally invoke corresponding function based on *SessionDestination* set in *PlaySessionRequest*.
```
 @ line: 1022
 void UEditorEngine::StartQueuedPlaySessionRequestImpl()
{
	if (!ensureAlwaysMsgf(PlaySessionRequest.IsSet(), TEXT("StartQueuedPlaySessionRequest should not be called without a request set!")))
	{
		return;
	}

	// End any previous sessions running in separate processes.
	EndPlayOnLocalPc();

	// If there's level already being played, close it. (This may change GWorld). 
	if (PlayWorld && PlaySessionRequest->SessionDestination == EPlaySessionDestinationType::InProcess)
	{
		// Cache our Play Session Request, as EndPlayMap will clear it. When this function exits the request will be reset anyways.
		FRequestPlaySessionParams OriginalRequest = PlaySessionRequest.GetValue();
		// Immediately end the current play world.
		EndPlayMap(); 
		// Restore the request as we're now processing it.
		PlaySessionRequest = OriginalRequest;
	}

	// We want to use the ULevelEditorPlaySettings that come from the Play Session Request.
	// By the time this function gets called, these settings are a copy of either the CDO, 
	// or a user provided instance. The settings may have been modified by the game instance
	// after the request was made, to allow game instances to pre-validate settings.
	const ULevelEditorPlaySettings* EditorPlaySettings = PlaySessionRequest->EditorPlaySettings;
	check(EditorPlaySettings);

	PlayInEditorSessionInfo = FPlayInEditorSessionInfo();
	PlayInEditorSessionInfo->PlayRequestStartTime = FPlatformTime::Seconds();
	PlayInEditorSessionInfo->PlayRequestStartTime_StudioAnalytics = FStudioAnalytics::GetAnalyticSeconds();

	// Keep a copy of their original request settings for any late
	// joiners or async processes that need access to the settings after launch.
	PlayInEditorSessionInfo->OriginalRequestParams = PlaySessionRequest.GetValue();

	// Load the saved window positions from the EditorPlaySettings object.
	for (const FIntPoint& Position : EditorPlaySettings->MultipleInstancePositions)
	{
		FPlayInEditorSessionInfo::FWindowSizeAndPos& NewPos = PlayInEditorSessionInfo->CachedWindowInfo.Add_GetRef(FPlayInEditorSessionInfo::FWindowSizeAndPos());
		NewPos.Position = Position;
	}

	// If our settings require us to launch a separate process in any form, we require the user to save
	// their content so that when the new process reads the data from disk it will match what we have in-editor.
	bool bUserWantsInProcess;
	EditorPlaySettings->GetRunUnderOneProcess(bUserWantsInProcess);

	bool bIsSeparateProcess = PlaySessionRequest->SessionDestination != EPlaySessionDestinationType::InProcess;
	if (!bUserWantsInProcess)
	{
		int32 NumClients;
		EditorPlaySettings->GetPlayNumberOfClients(NumClients);

		EPlayNetMode NetMode;
		EditorPlaySettings->GetPlayNetMode(NetMode);

		// More than one client will spawn a second process.		
		bIsSeparateProcess |= NumClients > 1;

		// If they want to run anyone as a client, a dedicated server is started in a separate process.
		bIsSeparateProcess |= NetMode == EPlayNetMode::PIE_Client;
	}

	if (bIsSeparateProcess && !SaveMapsForPlaySession())
	{
		// Maps did not save, print a warning
		FText ErrorMsg = LOCTEXT("PIEWorldSaveFail", "PIE failed because map save was canceled");
		UE_LOG(LogPlayLevel, Warning, TEXT("%s"), *ErrorMsg.ToString());
		FMessageLog(NAME_CategoryPIE).Warning(ErrorMsg);
		FMessageLog(NAME_CategoryPIE).Open();
		
		CancelRequestPlaySession();
		return;
	}

	// We'll branch primarily based on the Session Destination, because it affects which settings we apply and how.
	switch (PlaySessionRequest->SessionDestination)
	{
	case EPlaySessionDestinationType::InProcess:
		// Create one-or-more PIE/SIE sessions inside of the current process.
		StartPlayInEditorSession(PlaySessionRequest.GetValue());
		break;
	case EPlaySessionDestinationType::NewProcess:
		// Create one-or-more PIE session by launching a new process on the local machine.
		StartPlayInNewProcessSession(PlaySessionRequest.GetValue());
		break;
	case EPlaySessionDestinationType::Launcher:
		// Create a Play Session via the Launcher which may be on a local or remote device.
		StartPlayUsingLauncherSession(PlaySessionRequest.GetValue());
		break;
	default:
		check(false);
	}
}
```
{: file='\UE_4.27\Engine\Source\Editor\UnrealEd\Private\PlayLevel.cpp'}

> Any running session will be stopped when new session is requested.
{: .prompt-tip }

**CreateInnerProcessPIEGameInstance** function of **UEditorEngine** class will be finally invoked. This function create a new **GameInstance** and initialize it for **PIE** mode. Before dig deeper of **CreateInnerProcessPIEGameInstance**, let's go through the callstack and do a rough describe for those functions on stack.
![Create Instance Callstack](CreateInstance-Callstack.png)
_Callstack of CreateInnerProcessPIEGameInstance_

> This callstack occurred when *Play in Selected Viewport* and *Play Standalone* set. When play with other settings, the callstack may be different from shown one.
{: .prompt-tip }

|Function|Main Operations|
|:-------|:--------------|
|StartPlayInEditorSession| Do resource check like compile dirty blueprints, and prepare logging classes. Decide NetMode for each instance based on created count. |
|CreateNewPlayInEditorInstance| Gathering info for *PIELoginInfo* for instance shared the process with editor. Or launch new process for other instance(for server/multiple clients). |
|OnLoginPIEComplete_Deferred| Check login result of new created game instances. Stop current session if there is any instance failed to login. |
|CreateInnerProcessPIEGameInstance| The function actually create new game instance and do initialization. |


