---
title: TGuardValue Recoverable Value by Scope
date: 2022-04-17 19:27:10 +0800
categories: [Unreal Engine, C++]
tags: [learning ue, utils]
---

TGuardValue[^OfficialDoc] is a helper struct that can ensure a variable restores its previous value when leaving current scope.

## Implementation
TGuardValue is simply backup the original value of input variable, and set this value back when destructor execute. 
Following shows the source code:
```
 @line:279
 /** 
 * exception-safe guard around saving/restoring a value.
 * Commonly used to make sure a value is restored 
 * even if the code early outs in the future.
 * Usage:
 *  	TGuardValue<bool> GuardSomeBool(bSomeBool, false); // Sets bSomeBool to false, and restores it in dtor.
 */
template <typename RefType, typename AssignedType = RefType>
struct TGuardValue : private FNoncopyable
{
	TGuardValue(RefType& ReferenceValue, const AssignedType& NewValue)
	: RefValue(ReferenceValue), OldValue(ReferenceValue)
	{
		RefValue = NewValue;
	}
	~TGuardValue()
	{
		RefValue = OldValue;
	}

	/**
	 * Overloaded dereference operator.
	 * Provides read-only access to the original value of the data being tracked by this struct
	 *
	 * @return	a const reference to the original data value
	 */
	FORCEINLINE const AssignedType& operator*() const
	{
		return OldValue;
	}

private:
	RefType& RefValue;
	AssignedType OldValue;
};
```
{: file='UE_4.27\Engine\Source\Runtime\Core\Public\Templates\UnrealTemplate.h'}

## Usage
This struct is useful when some variables are neccessary to restore original value even if there may be exception occurred and interrupted normal executing path.

Here is a example:
```
void ATestCharacter::BeginPlay()
{
	Super::BeginPlay();

	bFinished = false;
	UE_LOG(LogTemp, Log, TEXT("[BeginPlay] bFinish = %d"), bFinished);
	TestFunction();
	UE_LOG(LogTemp, Log, TEXT("[BeginPlay] bFinish = %d"), bFinished);
}

void ATestCharacter::TestFunction()
{
	TGuardValue<bool> FinishGuard(bFinished, true);
	UE_LOG(LogTemp, Log, TEXT("[TestFunction] bFinish = %d"), bFinished);

	// Interrupt executing path
	return;

	bFinished = false;
	UE_LOG(LogTemp, Log, TEXT("[TestFunction] bFinish = %d"), bFinished);
}
```
{: file='TestCharacter.cpp'}

Run this code in editor can get following logs:
```
LogTemp: [BeginPlay] bFinish = 0
LogTemp: [TestFunction] bFinish = 1
LogTemp: [BeginPlay] bFinish = 0
```
The value of variable *bFinished* has been restored to the original value set in **BeginPlay** function without executing the restore code in **TestFunction**.

## References
[^OfficialDoc]: [**Official documentation**](https://docs.unrealengine.com/4.26/en-US/API/Runtime/Core/Templates/TGuardValue/)