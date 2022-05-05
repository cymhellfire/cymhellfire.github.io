---
title: How to Add Customized Sequence Track
date: 2022-05-04 14:41:02 +0800
categories: [Unreal Engine, Editor]
tags: [how to]
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