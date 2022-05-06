---
title: How to Add Customized Sequence Track
date: 2022-05-04 14:41:02 +0800
categories: [Unreal Engine, Editor]
tags: [how to, sequencer]
img_path: /img/2022-05-04-How-To-Add-Customized-Sequence-Track/
---

## How Sequencer Works?

This chapter focuses on the *Sequencer* relavent classes to figure out the basic mechanism that *Sequencer* works. The UML diagram below shows some of the relavent classes and the relationship between them.

![UE-Sequencer-UML](UE-Sequencer-Track-UML.jpg)
_Sequeuncer UML_

> Actually the *Sequencer* are driven by **ECS** architecture for better performance, but it's not the time to dig in **ECS**.
{: .prompt-tip}

### Sequencer Data Struct Classes
In the middle of the diagram, **UMovieSceneSequence** **UMovieScene** **UMovieSceneTrack** and **UMovieSceneSection** are the basic classes construct the *Sequencer* data. They have decreasing granularity from top to the bottom. Behavior relavent data are serialized in **UMovieSceneSection** instances. Here is a screenshot to show how they are composed in *Sequencer*.

![UE-Editor-Sequencer-Screenshot](UE-Editor-Sequencer-Screenshot.png)
_Sequencer Window_

### Sequencer Window Drawing Classes
Classes on the left are related to editor window rendering. The key class for adding customized track and sections is **FSequencerSection**. This class is in charged of creating the *Widget* of corresponding section and painting its background. Without this class, no section can be seen in the *Sequencer* window. The functions shows in UML diagram is the same as the content of refresh rendering callstack.

### Sequencer Runtime Classes
The right part contains classes take in count when evaluate the sequence at runtime. 
- **UMovieSceneSequencePlayer** is the toppest class to receive Play/Pause/Stop command. It also controls the underlying classes to perform corresponding behavior.
- **FMovieSceneRootEvaluationTemplateInstance** is the root class holds all the runtime evaluation data of current sequeuece.
- **UMovieSceneCompiledDataManager** has both serialization/deserialization functionality for sequence evaluation data. It also does compile work when any part of current sequence has changed, this ensures that the *Sequencer* window shows the latest data. It will ask corresponding **UMovieSceneTrack** object for creating **FMovieSceneEvalTemplate** instance for new adding section, and record the new template into **FMovieSceneEvaluationTrack** class.
- **FMovieSceneEvaluationTemplate** and **FMovieSceneEvaluationTrack** are two common data organizer classes.
- **FMovieSceneEvalTemplate** are the key class to perform evaluating behavior of sequence sections. Here list out three functions will be executed during the sequence playing.
  - *Setup()* triggers only once when evaluation entered the section. It's usually used to initialize the section logic and create **IPersistentEvaluationData**.
  - *TearDown()* is the same as *Setup* but triggered when section is end. Resources and reference releasing operation can be executed here.
  - *Evaluate()* function will be executed every frame during playing. Any continunous logic should be placed here. In generally, **IMovieSceneExecutionToken** instances are created here.

![UE-EvalTemplate-Event](UE-EvalTemplate-Event.png)
_Execute Timing_

- Any class derived from **IMovieSceneExecutionToken** interface is used as a executing token. The logic in its *Execute()* function will be executed during evaluation. It's usually created by **FMovieSceneEvalTemplate** class and added to **FMovieSceneExecutionTokens** struct for later evaluation.
- Any class derived from **IPersistentEvaluationData** interface is used as data container for section behavior. It's created by **FMovieSceneEvalTemplate** and recorded in **FPersistentEvaluationData** struct.

> Other classes are not important for add customized section, no more details placed here.
{: .prompt-tip}

## Add Customized Track
A brand new track needs one *Runtime* module and one *Editor* module for fully operational.
- *Runtime* module: Implements the classes contains actual logic of new track.
- *Editor* module: Implements the editor helper classes that ensure new track and sections can be rendered in **Sequencer** window.

### Runtime Module
#### UMovieSceneNewTrack
The very basic class is derived from **UMovieSceneTrack** and **IMovieSceneTrackTemplateProducer**. Create a new header file and type following code.

```
UCLASS()
class CUSTOMIZETRACK_API UMovieSceneNewTrack: public UMovieSceneNameableTrack, public IMovieSceneTrackTemplateProducer
{
	GENERATED_BODY()

public:
	UMovieSceneNewTrack()
	{
#if WITH_EDITORONLY_DATA
		TrackTint = FColor::Red;
#endif
	}

	// UMovieSceneTrack interface
	virtual void AddSection(UMovieSceneSection& Section) override { Sections.Add(&Section); }
	virtual bool SupportsType(TSubclassOf<UMovieSceneSection> SectionClass) const override;
	virtual UMovieSceneSection* CreateNewSection() override;
	virtual const TArray<UMovieSceneSection*>& GetAllSections() const override { return Sections; }
	virtual bool HasSection(const UMovieSceneSection& Section) const override { return Sections.Contains(&Section); }
	virtual bool IsEmpty() const override { return Sections.Num() == 0; }
	virtual void RemoveSection(UMovieSceneSection& Section) override { Sections.Remove(&Section); }
	virtual void RemoveSectionAt(int32 SectionIndex) override { Sections.RemoveAt(SectionIndex); }
	virtual bool SupportsMultipleRows() const override { return true; }

	// IMovieSceneTrackTemplateProducer interface
	virtual FMovieSceneEvalTemplatePtr CreateTemplateForSection(const UMovieSceneSection& InSection) const override;

#if WITH_EDITOR
	virtual FText GetDefaultDisplayName() const override;
#endif

protected:
	UPROPERTY()
	TArray<UMovieSceneSection*> Sections;
};
```
{: file='UMovieSceneNewTrack.h'}

The comments show that the functions' source. Variable **Sections** records all children sections of this track. *CreateTemplateForSection()* function is used to create **FMovieSceneEvalTemplate** instance for evaluation. 
Create new file named as **UMovieSceneNewTrack.cpp** and type following code.
```
#define LOCTEXT_NAMESPACE "MovieSceneNewTrack"

bool UMovieSceneNewTrack::SupportsType(TSubclassOf<UMovieSceneSection> SectionClass) const
{
	return SectionClass == UMovieSceneNewTrackSection::StaticClass();
}

UMovieSceneSection* UMovieSceneNewTrack::CreateNewSection()
{
	return NewObject<UMovieSceneNewTrackSection>(this, NAME_None, RF_Transactional);
}

FMovieSceneEvalTemplatePtr UMovieSceneNewTrack::CreateTemplateForSection(const UMovieSceneSection& InSection) const
{
	if (const UMovieSceneNewTrackSection* NewTrackSection = Cast<const UMovieSceneNewTrackSection>(&InSection))
	{
		return FMovieSceneNewTrackSectionTemplate(*NewTrackSection);
	}

	return FMovieSceneEvalTemplatePtr();
}

#if WITH_EDITOR
FText UMovieSceneNewTrack::GetDefaultDisplayName() const
{
	return LOCTEXT("NewTrack", "New Track");
}
#endif

#undef LOCTEXT_NAMESPACE
```
{: file='UMovieSceneNewTrack.cpp'}

#### UMovieSceneNewTrackSection
As mentioned before, a section class is also required. Create a new file named **MovieSceneNewTrackSection.h** and add following code.
```
UCLASS()
class CUSTOMIZETRACK_API UMovieSceneNewTrackSection : public UMovieSceneSection
{
	GENERATED_BODY()
public:
	UMovieSceneNewTrackSection() {}
};
```
{: file='MovieSceneNewTrackSection.h'}

Leave it empty for now, any extend logic can be added later.

#### FMovieSceneNewTrackSectionTemplate
Now create the runtime evaluation relative classes. Add new file named as **MovieSceneNewTrackSectionTemplate.h** and add following code.
```
class UMovieSceneNewTrackSection;

USTRUCT()
struct FMovieSceneNewTrackSectionTemplate : public FMovieSceneEvalTemplate
{
	GENERATED_BODY()

	FMovieSceneNewTrackSectionTemplate()
		: Section(nullptr)
	{}
	FMovieSceneNewTrackSectionTemplate(const UMovieSceneNewTrackSection& InSection);

	virtual void SetupOverrides() override
	{
		EnableOverrides(RequiresSetupFlag | RequiresTearDownFlag);
	}

	virtual void Evaluate(const FMovieSceneEvaluationOperand& Operand, const FMovieSceneContext& Context,
		const FPersistentEvaluationData& PersistentData, FMovieSceneExecutionTokens& ExecutionTokens) const override;
protected:
	virtual void Setup(FPersistentEvaluationData& PersistentData, IMovieScenePlayer& Player) const override;
	virtual void TearDown(FPersistentEvaluationData& PersistentData, IMovieScenePlayer& Player) const override;

	virtual UScriptStruct& GetScriptStructImpl() const override { return *StaticStruct(); }

	UPROPERTY()
	const UMovieSceneNewTrackSection* Section;
};
```
{: file='MovieSceneNewTrackSectionTemplate.h'}

Following the source file content.
```
struct FNewSectionEvaluationData;

struct FNewSectionExecutionToken : IMovieSceneExecutionToken
{
	virtual void Execute(const FMovieSceneContext& Context, const FMovieSceneEvaluationOperand& Operand, FPersistentEvaluationData& PersistentData, IMovieScenePlayer& Player) override
	{
		FNewSectionEvaluationData& EvaluationData = PersistentData.GetSectionData<FNewSectionEvaluationData>();
	}
};

struct FNewSectionEvaluationData : IPersistentEvaluationData
{
	
};

FMovieSceneNewTrackSectionTemplate::FMovieSceneNewTrackSectionTemplate(const UMovieSceneNewTrackSection& InSection)
{
	Section = &InSection;
}

void FMovieSceneNewTrackSectionTemplate::Evaluate(const FMovieSceneEvaluationOperand& Operand,
	const FMovieSceneContext& Context, const FPersistentEvaluationData& PersistentData,
	FMovieSceneExecutionTokens& ExecutionTokens) const
{
	ExecutionTokens.Add(FNewSectionExecutionToken());
}

void FMovieSceneNewTrackSectionTemplate::Setup(FPersistentEvaluationData& PersistentData,
	IMovieScenePlayer& Player) const
{
	PersistentData.AddSectionData<FNewSectionEvaluationData>();
}

void FMovieSceneNewTrackSectionTemplate::TearDown(FPersistentEvaluationData& PersistentData,
	IMovieScenePlayer& Player) const
{
	
}
```
{: file='MovieSceneNewTrackSectionTemplate.cpp'}

This file contains three different class/struct. Except class **FMovieSceneNewTrackSectionTemplate**, derived class of **IMovieSceneExecutionToken** and **IPersistentEvaluationData** are also implemented here. For demonstration purpose, there is no extra logic in these classes.

For now, the runtime part is finished. It's time to focus on editor side.

### Editor Module
#### FNewTrackEditor
There should be a dedicated track editor class for every single track class. Track editor class covers following functionalities:
- Create track/section instances and establish the relation between new instance and parent sequence.
- Construct *Sequencer* window menus for new track and section classes.
Create a new header file named **NewTrackEditor.h** in *Editor* module and add following code.
```
class UMovieSceneNewTrackSection;

class CUSTOMIZETRACKEDITOR_API FNewTrackEditor : public FMovieSceneTrackEditor
{
public:
	static TSharedRef<ISequencerTrackEditor> CreateTrackEditor(TSharedRef<ISequencer> InSequencer);

	FNewTrackEditor(TSharedRef<ISequencer> InSequencer);

	// ISequencerTrackEditor interface
	virtual void BuildAddTrackMenu(FMenuBuilder& MenuBuilder) override;
	virtual TSharedPtr<SWidget> BuildOutlinerEditWidget(const FGuid& ObjectBinding, UMovieSceneTrack* Track, const FBuildEditWidgetParams& Params) override;
	virtual bool SupportsType(TSubclassOf<UMovieSceneTrack> TrackClass) const override;
	virtual bool SupportsSequence(UMovieSceneSequence* InSequence) const override;

	virtual TSharedRef<ISequencerSection> MakeSectionInterface(UMovieSceneSection& SectionObject, UMovieSceneTrack& Track, FGuid ObjectBinding) override;

protected:
	UMovieSceneNewTrackSection* AddNewSection(UMovieScene* MovieScene, UMovieSceneTrack* InTrack);

	void HandleAddNewTrackMenuEntryExecute();
	void OnAddNewSection(UMovieSceneTrack* MovieSceneTrack);
	TSharedRef<SWidget> BuildAddNewSectionMenu(UMovieSceneTrack* MovieSceneTrack);
};
```
{: file='NewTrackEditor.h'}

In the corresponding source file NewTrackEditor.cpp, add following code.
```
#define LOCTEXT_NAMESPACE "FNewTrackEditor"

TSharedRef<ISequencerTrackEditor> FNewTrackEditor::CreateTrackEditor(TSharedRef<ISequencer> InSequencer)
{
	return MakeShareable(new FNewTrackEditor(InSequencer));
}

FNewTrackEditor::FNewTrackEditor(TSharedRef<ISequencer> InSequencer)
	: FMovieSceneTrackEditor(InSequencer)
{
}

void FNewTrackEditor::BuildAddTrackMenu(FMenuBuilder& MenuBuilder)
{
	MenuBuilder.AddMenuEntry(
		LOCTEXT("AddNewTrack", "New Track"),
		LOCTEXT("AddNewTrackTooltip", "Adds a new track that contains new sections."),
		FSlateIcon(FEditorStyle::GetStyleSetName(), "Sequencer.Tracks.Fade"),
		FUIAction(
			FExecuteAction::CreateRaw(this, &FNewTrackEditor::HandleAddNewTrackMenuEntryExecute)
			)
		);
}

TSharedPtr<SWidget> FNewTrackEditor::BuildOutlinerEditWidget(const FGuid& ObjectBinding, UMovieSceneTrack* Track,
	const FBuildEditWidgetParams& Params)
{
	return FSequencerUtilities::MakeAddButton(
		LOCTEXT("AddNewSection", "Section"),
		FOnGetContent::CreateSP(this, &FNewTrackEditor::BuildAddNewSectionMenu, Track),
		Params.NodeIsHovered, GetSequencer());
}

TSharedRef<SWidget> FNewTrackEditor::BuildAddNewSectionMenu(UMovieSceneTrack* MovieSceneTrack)
{
	FMenuBuilder MenuBuilder(true, nullptr);

	MenuBuilder.AddMenuEntry(
		LOCTEXT("AddNewSection", "New Section"),
		LOCTEXT("AddNewSectionTooltip", "Add a new section."),
		FSlateIcon(),
		FUIAction(FExecuteAction::CreateSP(this, &FNewTrackEditor::OnAddNewSection, MovieSceneTrack)));

	return MenuBuilder.MakeWidget();
}

void FNewTrackEditor::OnAddNewSection(UMovieSceneTrack* MovieSceneTrack)
{
	UMovieScene* FocusedMovieScene = GetFocusedMovieScene();
	if (FocusedMovieScene == nullptr)
	{
		return;
	}

	UMovieSceneNewTrackSection* NewSection = AddNewSection(FocusedMovieScene, MovieSceneTrack);

	GetSequencer()->NotifyMovieSceneDataChanged(EMovieSceneDataChangeType::MovieSceneStructureItemAdded);
}

UMovieSceneNewTrackSection* FNewTrackEditor::AddNewSection(UMovieScene* MovieScene, UMovieSceneTrack* InTrack)
{
	const FScopedTransaction Transaction(LOCTEXT("AddNewSection_Transaction", "Add new section to track."));

	InTrack->Modify();

	UMovieSceneNewTrackSection* NewSection = CastChecked<UMovieSceneNewTrackSection>(InTrack->CreateNewSection());

	TRange<FFrameNumber> SectionRange = MovieScene->GetPlaybackRange();
	NewSection->SetRange(SectionRange);

	int32 RowIndex = -1;
	for (const UMovieSceneSection* Section : InTrack->GetAllSections())
	{
		RowIndex = FMath::Max(RowIndex, Section->GetRowIndex());
	}
	NewSection->SetRowIndex(RowIndex + 1);

	InTrack->AddSection(*NewSection);

	return NewSection;
}

bool FNewTrackEditor::SupportsType(TSubclassOf<UMovieSceneTrack> TrackClass) const
{
	return TrackClass == UMovieSceneNewTrack::StaticClass();
}

bool FNewTrackEditor::SupportsSequence(UMovieSceneSequence* InSequence) const
{
	return true;
}

void FNewTrackEditor::HandleAddNewTrackMenuEntryExecute()
{
	UMovieScene* FocusedMovieScene = GetFocusedMovieScene();

	if (FocusedMovieScene == nullptr || FocusedMovieScene->IsReadOnly())
	{
		return;
	}

	const FScopedTransaction Transaction(LOCTEXT("AddNewTrack_Transaction", "Add New Track"));
	FocusedMovieScene->Modify();

	auto NewTrack = FocusedMovieScene->AddMasterTrack<UMovieSceneNewTrack>();
	check(NewTrack);

	auto CurSequencer = GetSequencer();
	CurSequencer->NotifyMovieSceneDataChanged(EMovieSceneDataChangeType::MovieSceneStructureItemAdded);
}

TSharedRef<ISequencerSection> FNewTrackEditor::MakeSectionInterface(UMovieSceneSection& SectionObject,
	UMovieSceneTrack& Track, FGuid ObjectBinding)
{
	UMovieSceneNewTrackSection* NewTrackSection = Cast<UMovieSceneNewTrackSection>(&SectionObject);
	check(SupportsType(SectionObject.GetOuter()->GetClass()) && NewTrackSection != nullptr);

	return MakeShareable( new FNewTrackSection(*NewTrackSection));
}

#undef LOCTEXT_NAMESPACE
```
{: file='NewTrackEditor.cpp'}

This class derived from **FMovieSceneTrackEditor** and specified the support type as **UMovieSceneNewTrack**. There is several menu constructor functions, here post some screenshot to introduce the correspondence between code and actual menu.
- *BuildAddTrackMenu*: Construct the button in **Add Track** menu.
![UE-BuildAddTrackMenu](UE-BuildAddTrackMenu.png)
_BuildAddTrackMenu_
- *BuildOutlinerEditWidget*: Construct the "+" button on the track item.
![UE-BuildOutlinerEditWidget](UE-BuildOutlinerEditWidget.png)
_BuildOutlinerEditWidget_
- *BuildTrackContextMenu*: Construct extra options in the right click popup menu of track item.
![UE-BuildTrackContextMenu](UE-BuildTrackContextMenu.png)
_BuildTrackContextMenu_
In the example code, **BuildTrackContextMenu** is not implemented since it's not necessary.

Another important function is *MakeSectionInterface*. This function instantiate **FNewTrackSection** (this will be introduce in next part) for every new section instance. Without this function, no section can be rendered in *Sequencer* window.

#### FNewTrackSection
Derived from **FSequencerSection** and do the rendering job for section instances. Create new source files named **NewTrackSection.h** and **NewTrackSection.cpp** then add following code.
```
class CUSTOMIZETRACKEDITOR_API FNewTrackSection : public FSequencerSection
{
public:
	FNewTrackSection(UMovieSceneSection& InSectionObject);

	virtual int32 OnPaintSection(FSequencerSectionPainter& Painter) const override;

	virtual FText GetSectionTitle() const override;
};
```
{: file='NewTrackSection.h'}

```
FNewTrackSection::FNewTrackSection(UMovieSceneSection& InSectionObject)
	: FSequencerSection(InSectionObject)
{
}

int32 FNewTrackSection::OnPaintSection(FSequencerSectionPainter& Painter) const
{
	return Painter.PaintSectionBackground();
}

FText FNewTrackSection::GetSectionTitle() const
{
	return FText::FromString(TEXT("NewSection"));
}
```
{: file='NewTrackSection.cpp'}

Two functions are implemented for rendering purpose.
- *OnPaintSection*: Simply draw a rectangle as the section duration in sequence track.
- *GetSectionTitle*: Override the text displayed on top of the section item.

![UE-TrackSection](UE-TrackSection.png)
_Rendering Result_

> **FSequencerSection** also provides more functions like *GenerateSectionWidget* to customize the section appearance.
{: .prompt-tip}

#### Register New Track Editor
The final step to enable new track edtior in *Sequencer* window is register the new track editor class to *Sequencer* module.
Open the source file of editor module, **CustomizeTrackEditor.cpp** in my case. Add registration code to *StartupModule* function and unregistration in *Shutdown* like following.
```
void FCustomizeTrackEditorModule::StartupModule()
{
	UE_LOG(LogTemp, Log, TEXT("[CustomizeTrackEditor] Startup"));

	ISequencerModule& SequencerModule = FModuleManager::Get().LoadModuleChecked<ISequencerModule>("Sequencer");
	// Register new track editor
	NewTrackCreateEditorHandle = SequencerModule.RegisterTrackEditor(FOnCreateTrackEditor::CreateStatic(&FNewTrackEditor::CreateTrackEditor));
}

void FCustomizeTrackEditorModule::ShutdownModule()
{
	UE_LOG(LogTemp, Log, TEXT("[CustomizeTrackEditor] Shutdown"));

	ISequencerModule& SequencerModule = FModuleManager::Get().GetModuleChecked<ISequencerModule>( "Sequencer" );
	// Unregister callback handle
	SequencerModule.UnRegisterTrackEditor(NewTrackCreateEditorHandle);
}
```
{: file='CustomizeTrackEditor.cpp'}

Save and compile, the new track should be available in *Sequencer* window.

## Add New Event Track
Previous chapter described how to add a section formed track, this one will talk about the event track. Unlike the section tracks, event track usually has only one infinite length section. All the activated events are located on the section area. Its mechanism is also different, before actual adding new track, let's check the new involved classes for making event track works.

### New Involved Classes
Here is a UML diagram includes necessary classes for event track.

![UE-Sequencer-Event-Track-UML](UE-Sequencer-Event-Track-UML.jpg)
_Event Track UML_

> All other classes shown before is hidden in this UML. All classes use actual name in sample code.
{: .prompt-tip}

The middle part is very similiar to section track UML diagram. Here introduce new invovled classes one by one:
- New section class need implemented **IMovieSceneEntityProvider** interface which provides ability to generate triggerable event entities.
- **FMovieSceneNewEventChannel** derived from **FMovieSceneChannel** class. It manipulates time and data struct of all created events. 
- **UMovieSceneNewEventSystem** is the logic executor class. It's triggered during evaluation and run the event logic once the time matches.
- **FMovieSceneNewEventTriggerData** is the actual data used when event is triggered. The instances are created by *ImportEntityImpl* function of **UMovieSceneNewEventTriggerSection** class and sent to **UMovieSceneNewEventSystem** class for later triggering. The difference between it and **FMovieSceneNewEvent** is that this struct also contains evaluation relevant data like the time point of trigger.

### Runtime Module
#### UMovieSceneNewEventTrack
The track class is every similiar to section track's one. Only thing to do is change the support type to correspongding classes.

#### UMovieSceneNewEventTriggerSection
The section class need to turn on infinite range support since the event section always covers the hold track to ensure all events are available. Implementing **IMovieSceneEntityProvider** interface is also a must done work.

```
UCLASS()
class CUSTOMIZETRACK_API UMovieSceneNewEventTriggerSection : public UMovieSceneSection
	, public IMovieSceneEntityProvider
{
	GENERATED_BODY()
public:

	UMovieSceneNewEventTriggerSection();

	// IMovieSceneEntityProvider
	virtual bool PopulateEvaluationFieldImpl(const TRange<FFrameNumber>& EffectiveRange, const FMovieSceneEvaluationFieldEntityMetaData& InMetaData, FMovieSceneEntityComponentFieldBuilder* OutFieldBuilder) override;
	virtual void ImportEntityImpl(UMovieSceneEntitySystemLinker* EntityLinker, const FEntityImportParams& Params, FImportedEntity* OutImportedEntity) override;

	UPROPERTY()
	FMovieSceneNewEventChannel EventChannel;
};
```
{: file='MovieSceneNewEventTriggerSection.h'}

```
UMovieSceneNewEventTriggerSection::UMovieSceneNewEventTriggerSection()
{
	bSupportsInfiniteRange = true;
	SetRange(TRange<FFrameNumber>::All());

#if WITH_EDITOR
	ChannelProxy = MakeShared<FMovieSceneChannelProxy>(EventChannel, FMovieSceneChannelMetaData());
#endif
}

bool UMovieSceneNewEventTriggerSection::PopulateEvaluationFieldImpl(const TRange<FFrameNumber>& EffectiveRange, 
	const FMovieSceneEvaluationFieldEntityMetaData& InMetaData, FMovieSceneEntityComponentFieldBuilder* OutFieldBuilder)
{
	const int32 MetaDataIndex = OutFieldBuilder->AddMetaData(InMetaData);

	TArrayView<const FFrameNumber> Times = EventChannel.GetData().GetTimes();
	for (int32 Index = 0; Index < Times.Num(); ++Index)
	{
		if (EffectiveRange.Contains(Times[Index]))
		{
			TRange<FFrameNumber> Range(Times[Index]);
			OutFieldBuilder->AddOneShotEntity(Range, this, Index, MetaDataIndex);
		}
	}

	return true;
}

void UMovieSceneNewEventTriggerSection::ImportEntityImpl(UMovieSceneEntitySystemLinker* EntityLinker,
	const FEntityImportParams& Params, FImportedEntity* OutImportedEntity)
{
	using namespace UE::MovieScene;

	const int32 EventIndex = static_cast<int32>(Params.EntityID);

	TArrayView<const FFrameNumber> Times = EventChannel.GetData().GetTimes();
	TArrayView<const FMovieSceneNewEvent> Events = EventChannel.GetData().GetValues();
	if (!ensureMsgf(Events.IsValidIndex(EventIndex), TEXT("Attempting to import an event entity for an invalid index (Index: %d, Num: %d)"), EventIndex, Events.Num()))
	{
		return;
	}

	if (Events[EventIndex].EventName.IsEmpty())
	{
		return;
	}

	UMovieSceneNewEventTrack* EventTrack = GetTypedOuter<UMovieSceneNewEventTrack>();
	const FSequenceInstance& ThisInstance = EntityLinker->GetInstanceRegistry()->GetInstance(Params.Sequence.InstanceHandle);
	FMovieSceneContext Context = ThisInstance.GetContext();

	if (Context.GetStatus() == EMovieScenePlayerStatus::Stopped || Context.IsSilent())
	{
		return;
	}

	// Get the logic executor system
	UMovieSceneNewEventSystem* EventSystem = EntityLinker->LinkSystem<UMovieSceneNewEventSystem>();

	FMovieSceneNewEventTriggerData TriggerData = {
		Events[EventIndex].EventName,
		Times[EventIndex] * Context.GetSequenceToRootTransform()
	};

	EventSystem->AddEvent(ThisInstance.GetRootInstanceHandle(), TriggerData);

	// Mimic the structure changing in order to ensure that the instantiation phase runs
	EntityLinker->EntityManager.MimicStructureChanged();
}
```
{: file='MovieSceneNewEventTriggerSection.cpp'}

- *ImportEntityImpl* function create **FMovieSceneNewEventTriggerData** instances based on event data stores in **FMovieSceneNewEventChannel** class, and push them to **UMovieSceneNewEventSystem** class for evaluation purpose. It will be invoked when any event should be triggered during evaluation.
- *PopulateEvaluationFieldImpl* function adds existing events to track builder that ensure all events are rendered in *Sequencer* window.

#### UMovieSceneNewEventSystem
**UMovieSceneNewEventSystem** class takes passed in **FMovieSceneNewEventTriggerData** and execute the actual event logic based on data inside trigger data instances.

```
class IMovieScenePlayer;

USTRUCT()
struct FMovieSceneNewEventTriggerData
{
	GENERATED_BODY()

	UPROPERTY()
	FString TriggerName;

	FFrameTime RootTime;
};

UCLASS()
class CUSTOMIZETRACK_API UMovieSceneNewEventSystem : public UMovieSceneEntitySystem
{
	GENERATED_BODY()
public:

	void AddEvent(UE::MovieScene::FInstanceHandle RootInstance, const FMovieSceneNewEventTriggerData& TriggerData);

	// UMovieSceneEntitySystem interface
	virtual void OnRun(FSystemTaskPrerequisites& InPrerequisites, FSystemSubsequentTasks& Subsequents) override;
	virtual bool IsRelevantImpl(UMovieSceneEntitySystemLinker* InLinker) const override;

protected:

	bool HasEvents() const;
	void TriggerAllEvents();

private:

	static void TriggerEvents(TArrayView<const FMovieSceneNewEventTriggerData> Events, IMovieScenePlayer* Player);

private:
	TMap<UE::MovieScene::FInstanceHandle, TArray<FMovieSceneNewEventTriggerData>> EventsByRoot;
};
```
{: file='MovieSceneNewEventSystem.h'}

```
DECLARE_CYCLE_STAT(TEXT("NewEvent Systems"), MovieSceneEval_NewEvents, STATGROUP_MovieSceneECS);
DECLARE_CYCLE_STAT(TEXT("Trigger NewEvents"), MovieSceneEval_TriggerNewEvents, STATGROUP_MovieSceneECS);

bool UMovieSceneNewEventSystem::HasEvents() const
{
	return EventsByRoot.Num() > 0;
}

bool UMovieSceneNewEventSystem::IsRelevantImpl(UMovieSceneEntitySystemLinker* InLinker) const
{
	return HasEvents();
}

void UMovieSceneNewEventSystem::AddEvent(UE::MovieScene::FInstanceHandle RootInstance,
	const FMovieSceneNewEventTriggerData& TriggerData)
{
	check(!TriggerData.TriggerName.IsEmpty());
	EventsByRoot.FindOrAdd(RootInstance).Add(TriggerData);
}

void UMovieSceneNewEventSystem::OnRun(FSystemTaskPrerequisites& InPrerequisites, FSystemSubsequentTasks& Subsequents)
{
	if (HasEvents())
	{
		TriggerAllEvents();
	}
}

void UMovieSceneNewEventSystem::TriggerAllEvents()
{
	using namespace UE::MovieScene;

	SCOPE_CYCLE_COUNTER(MovieSceneEval_NewEvents);

	FInstanceRegistry* InstanceRegistry = Linker->GetInstanceRegistry();
	struct FTriggerBatch
	{
		TArray<FMovieSceneNewEventTriggerData> TriggerData;
		IMovieScenePlayer* Player;
	};
	TArray<FTriggerBatch> TriggerBatches;

	for (TPair<FInstanceHandle, TArray<FMovieSceneNewEventTriggerData>>& Pair : EventsByRoot)
	{
		const FSequenceInstance& RootInstance = InstanceRegistry->GetInstance(Pair.Key);

		if (RootInstance.GetContext().GetDirection() == EPlayDirection::Forwards)
		{
			Algo::SortBy(Pair.Value, &FMovieSceneNewEventTriggerData::RootTime);
		}
		else
		{
			Algo::SortBy(Pair.Value, &FMovieSceneNewEventTriggerData::RootTime, TGreater<>());
		}

		FTriggerBatch& TriggerBatch = TriggerBatches.Emplace_GetRef();
		TriggerBatch.TriggerData = Pair.Value;
		TriggerBatch.Player = RootInstance.GetPlayer();
	}

	// We need to clean our state before actually triggering the events because one of those events could
	// call back into an evaluation (for instance, by starting play on another sequence). If we don't clean
	// this before, would would re-enter and re-trigger past events, resulting in an infinite loop!
	EventsByRoot.Empty();

	for (const FTriggerBatch& TriggerBatch : TriggerBatches)
	{
		TriggerEvents(TriggerBatch.TriggerData, TriggerBatch.Player);
	}
}

void UMovieSceneNewEventSystem::TriggerEvents(TArrayView<const FMovieSceneNewEventTriggerData> Events,
	IMovieScenePlayer* Player)
{
	for (const FMovieSceneNewEventTriggerData& Event : Events)
	{
		SCOPE_CYCLE_COUNTER(MovieSceneEval_TriggerNewEvents);

		UE_LOG(LogTemp, Log, TEXT("[NewEvent] %s"), *Event.TriggerName);
	}
}
```
{: file='MovieSceneNewEventSystem.cpp'}

- *IsRelevantImpl* function decides whether this system is available when try to access it by **EntityLinker**.
- *OnRun* is the function contains need logic to execute events.
In this case, the system only print a log when event is triggered.

#### FMovieSceneNewEventChannel
**FMovieSceneNewEventChannel** manipulates all event data for parent section.

```
USTRUCT()
struct CUSTOMIZETRACK_API FMovieSceneNewEventChannel : public FMovieSceneChannel
{
	GENERATED_BODY()

	FORCEINLINE TMovieSceneChannelData<FMovieSceneNewEvent> GetData()
	{
		return TMovieSceneChannelData<FMovieSceneNewEvent>(&KeyTimes, &KeyValues, &KeyHandles);
	}

	/**
	 * Constant version of GetData()
	 */
	FORCEINLINE TMovieSceneChannelData<const FMovieSceneNewEvent> GetData() const
	{
		return TMovieSceneChannelData<const FMovieSceneNewEvent>(&KeyTimes, &KeyValues);
	}

	FORCEINLINE TArrayView<const FFrameNumber> GetTimes() const
	{
		return KeyTimes;
	}

	FORCEINLINE TArrayView<const FMovieSceneNewEvent> GetValues() const
	{
		return KeyValues;
	}

	// FMovieSceneChannel interface
	virtual void GetKeys(const TRange<FFrameNumber>&
		WithinRange, TArray<FFrameNumber>* OutKeyTimes, TArray<FKeyHandle>* OutKeyHandles) override;
	virtual void GetKeyTimes(TArrayView<const FKeyHandle> InHandles, TArrayView<FFrameNumber> OutKeyTimes) override;
	virtual void SetKeyTimes(TArrayView<const FKeyHandle> InHandles, TArrayView<const FFrameNumber> InKeyTimes) override;
	virtual void DuplicateKeys(TArrayView<const FKeyHandle> InHandles, TArrayView<FKeyHandle> OutNewHandles) override;
	virtual void DeleteKeys(TArrayView<const FKeyHandle> InHandles) override;
	virtual void DeleteKeysFrom(FFrameNumber InTime, bool bDeleteKeysBefore) override;
	virtual void ChangeFrameResolution(FFrameRate SourceRate, FFrameRate DestinationRate) override;
	virtual TRange<FFrameNumber> ComputeEffectiveRange() const override;
	virtual int32 GetNumKeys() const override;
	virtual void Reset() override;
	virtual void Offset(FFrameNumber DeltaPosition) override;

private:
	UPROPERTY(meta=(KeyTimes))
	TArray<FFrameNumber> KeyTimes;

	UPROPERTY(meta=(KeyValues))
	TArray<FMovieSceneNewEvent> KeyValues;

	FMovieSceneKeyHandleMap KeyHandles;
};

namespace MovieSceneClipboard
{
	template<> inline FName GetKeyTypeName<FMovieSceneNewEvent>()
	{
		return "MovieSceneNewEvent";
	}
}

template<>
struct TMovieSceneChannelTraits<FMovieSceneNewEventChannel> : TMovieSceneChannelTraitsBase<FMovieSceneNewEventChannel>
{
	enum { SupportsDefaults = false };
};

inline bool EvaluateChannel(const FMovieSceneNewEventChannel* InChannel, FFrameTime InTime, FMovieSceneNewEvent& OutValue)
{
	// Can't evaluate event section data in the typical sense
	return false;
}
```
{: file='MovieSceneNewEventChannel.h'}

> Notice that some individual functions are also placed here. They are neccessary for register channel interface in editor module later.
{: .prompt-tip}

```
void FMovieSceneNewEventChannel::GetKeys(const TRange<FFrameNumber>& WithinRange, TArray<FFrameNumber>* OutKeyTimes,
	TArray<FKeyHandle>* OutKeyHandles)
{
	GetData().GetKeys(WithinRange, OutKeyTimes, OutKeyHandles);
}

void FMovieSceneNewEventChannel::GetKeyTimes(TArrayView<const FKeyHandle> InHandles,
	TArrayView<FFrameNumber> OutKeyTimes)
{
	GetData().GetKeyTimes(InHandles, OutKeyTimes);
}

void FMovieSceneNewEventChannel::SetKeyTimes(TArrayView<const FKeyHandle> InHandles,
	TArrayView<const FFrameNumber> InKeyTimes)
{
	GetData().SetKeyTimes(InHandles, InKeyTimes);
}

void FMovieSceneNewEventChannel::DuplicateKeys(TArrayView<const FKeyHandle> InHandles,
	TArrayView<FKeyHandle> OutNewHandles)
{
	GetData().DuplicateKeys(InHandles, OutNewHandles);
}

void FMovieSceneNewEventChannel::DeleteKeys(TArrayView<const FKeyHandle> InHandles)
{
	GetData().DeleteKeys(InHandles);
}

void FMovieSceneNewEventChannel::DeleteKeysFrom(FFrameNumber InTime, bool bDeleteKeysBefore)
{
	GetData().DeleteKeysFrom(InTime, bDeleteKeysBefore);
}

void FMovieSceneNewEventChannel::ChangeFrameResolution(FFrameRate SourceRate, FFrameRate DestinationRate)
{
	GetData().ChangeFrameResolution(SourceRate, DestinationRate);
}

TRange<FFrameNumber> FMovieSceneNewEventChannel::ComputeEffectiveRange() const
{
	return GetData().GetTotalRange();
}

int32 FMovieSceneNewEventChannel::GetNumKeys() const
{
	return KeyTimes.Num();
}

void FMovieSceneNewEventChannel::Reset()
{
	KeyTimes.Reset();
	KeyValues.Reset();
	KeyHandles.Reset();
}

void FMovieSceneNewEventChannel::Offset(FFrameNumber DeltaPosition)
{
	GetData().Offset(DeltaPosition);
}
```
{: file='MovieSceneNewEventChannel.cpp'}

According the source code above, this class is only a wrapper class without actual logic. KeyTimes, KeyValues and KeyHandles are the key variables.

#### FMovieSceneNewEvent
Finally, here is the event definition struct. A single **FString** variable is enought for this example.

```
USTRUCT(BlueprintType)
struct FMovieSceneNewEvent
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere, Category="NewEvent")
	FString EventName;
};

```
{: file='MovieSceneNewEvent.h'}

### Editor Module
For event track, it's necessary to register channel interface for new track. Open module source file and change the *StartupModule()* function like following.

```
void FCustomizeTrackEditorModule::StartupModule()
{
	UE_LOG(LogTemp, Log, TEXT("[CustomizeTrackEditor] Startup"));

	ISequencerModule& SequencerModule = FModuleManager::Get().LoadModuleChecked<ISequencerModule>("Sequencer");
	// Register new track editor
	NewTrackCreateEditorHandle = SequencerModule.RegisterTrackEditor(FOnCreateTrackEditor::CreateStatic(&FNewTrackEditor::CreateTrackEditor));
	NewEventTrackCreateEditorHandle = SequencerModule.RegisterTrackEditor(FOnCreateTrackEditor::CreateStatic(&FNewEventTrackEditor::CreateTrackEditor));

	// Register new channel interface
	SequencerModule.RegisterChannelInterface<FMovieSceneNewEventChannel>();
}
```
{: file='CustomizeTrackEditor.cpp'}

> Ensure that header file **"SequencerChannelInterface.h"** is included. Otherwise project will not be compiled.
{: .prompt-tip}

The section renderer and editor menu construtor classes can be added like section track's, no more detail about them.

![UE-New-Event-Track](UE-New-Event-Track.png)
_New Event Track_

## Example Code
The example project shows in this article can be found [**here**](https://github.com/cymhellfire/Blogging/tree/main/Plugins/CustomizeTrack).