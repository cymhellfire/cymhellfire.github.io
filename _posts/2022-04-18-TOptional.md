---
title: TOptional Nullable Struct Variable
date: 2022-04-18 22:54:15 +0800
categories: [Unreal Engine, C++]
tags: [learning ue, utils]
img_path: /img/2022-04-18-TOptional/
---

Struct is widely used in Unreal C++ programming. But whenever you want to distinguish an invalid struct value, a special struct instance is always the best choose. With *TOptional*[^OfficialDoc] template, there is a more elegant solution for this case.

## Example

First, here is a simple struct with two float value inside.
```
struct FMyCharacterData
{
	float Health;
	float Mana;
};
```
{: file='Sample Struct'}

To declare a nullable struct variable, just add a line like follows.
```
TOptional<FMyCharacterData> CharacterData;
``` 
{: file='Variable Declaration'}

Use member function *GetValue()* to access the underlying value.

> When any logic try to access its value before assign any meaningful value, it will trigger a breakpoint(when debugger attached).
{: .prompt-warning }

```
void ATestActor::BeginPlay()
{
	Super::BeginPlay();

	FMyCharacterData NewData = CharacterData.GetValue();
	UE_LOG(LogTemp, Log, TEXT("NewData Health: %f"), NewData.Health);
}
```
{: file='Access Variable without Assignment'}

![Exception without Checking](Exception-Access-No-Check.png)
_Exception thrown when accessing_

To avoid the situation above, check the return value of *IsSet()* function before access the value.

```
void ATestActor::BeginPlay()
{
	Super::BeginPlay();

	if (CharacterData.IsSet())
	{
		FMyCharacterData NewData = CharacterData.GetValue();
		UE_LOG(LogTemp, Log, TEXT("NewData Health: %f"), NewData.Health);
	}
}
```
{: file='Check Logic'}

With *TOptional* template, no need to create special instance for every struct type.

## How it works

*TOptional* struct is defined in file \Engine\Source\Runtime\Core\Public\Misc\Optional.h. *TOptional* has lots of different constructors, just take one of them for example.
```
/** Construct an OptionaType with a valid value. */
TOptional(const OptionalType& InValue)
{
	new(&Value) OptionalType(InValue);
	bIsSet = true;
}
```
{: file='First Constructor'}

According to the code above, there is a variable named **Value** in *TOptional* struct for storing underlying struct. When construct with a valid value, the constructor will create a new struct instance at the address pointed by **Value** with given value. And set **bIsSet** to true to indicate this *TOptional* instance is meaningful.

```
~TOptional()
{
	Reset();
}

...

void Reset()
{
	if (bIsSet)
	{
		bIsSet = false;

		// We need a typedef here because VC won't compile the destructor call below if OptionalType itself has a member called OptionalType
		typedef OptionalType OptionalDestructOptionalType;
		((OptionalType*)&Value)->OptionalDestructOptionalType::~OptionalDestructOptionalType();
	}
}
```
{: file='Destructor'}

In destrutor, *TOptional* invoke *Reset()* function to release the underlying value. As the above code, it call the destrcutor of underlying struct type.

```
/** @return The optional value; undefined when IsSet() returns false. */
const OptionalType& GetValue() const { checkf(IsSet(), TEXT("It is an error to call GetValue() on an unset TOptional. Please either check IsSet() or use Get(DefaultValue) instead.")); return *(OptionalType*)&Value; }
		OptionalType& GetValue()		 { checkf(IsSet(), TEXT("It is an error to call GetValue() on an unset TOptional. Please either check IsSet() or use Get(DefaultValue) instead.")); return *(OptionalType*)&Value; }
```
{: file='Getter Function'}

The getter function will check **bIsSet** variable internal. It will fail to pass the check if no valid value has been set.

```
/** @return The optional value when set; DefaultValue otherwise. */
const OptionalType& Get(const OptionalType& DefaultValue) const { return IsSet() ? *(OptionalType*)&Value : DefaultValue; }
```
{: file='Get with Fallback'}

*TOptional* also provides a variant getter with default value as fallback. When no valid value set, it will return the passed in default value.

```
TTypeCompatibleBytes<OptionalType> Value;
```
{: file='Data Storage'}

At the end, let's check about the variable **Value**. To hold the actual data from different types, it's declared as a *TTypeCompatibleBytes* struct.

```
/** An untyped array of data with compile-time alignment and size derived from another type. */
template<typename ElementType>
struct TTypeCompatibleBytes :
	public TAlignedBytes<
		sizeof(ElementType),
		alignof(ElementType)
		>
{
	ElementType*		GetTypedPtr()		{ return (ElementType*)this;  }
	const ElementType*	GetTypedPtr() const	{ return (const ElementType*)this; }
};
```
{: file='TTypeCompatibleBytes'}

*TTypeCompatibleBytes* is a simple struct that derived from *TAlignedBytes*. In TypeCompatibleBytes.h, there are four  implementations based on different alignment.
```
/** A macro that implements TAlignedBytes for a specific alignment. */
#define IMPLEMENT_ALIGNED_STORAGE(Align) \
	template<int32 Size>        \
	struct TAlignedBytes<Size,Align> \
	{ \
		struct MS_ALIGN(Align) TPadding \
		{ \
			uint8 Pad[Size]; \
		} GCC_ALIGN(Align); \
		TPadding Padding; \
	};
#endif

// Implement TAlignedBytes for these alignments.
IMPLEMENT_ALIGNED_STORAGE(16);
IMPLEMENT_ALIGNED_STORAGE(8);
IMPLEMENT_ALIGNED_STORAGE(4);
IMPLEMENT_ALIGNED_STORAGE(2);
```
{: file='Implementations'}

The compatible one will be chosen based on given **ElementType**. The essential is, the data is stored in a **uint8** array and the array has the same size as the **ElementType** to ensure whole element is stored.

## Reference
[^OfficialDoc]: [**Official documentation**](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Core/Misc/TOptional/)