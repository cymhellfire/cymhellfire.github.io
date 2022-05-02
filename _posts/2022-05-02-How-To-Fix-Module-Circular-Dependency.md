---
title: How to Fix Module Circular Dependency
date: 2022-05-02 10:56:25 +0800
categories: [Unreal Engine, Programming]
tags: [how to]
img_path: /img/2022-05-02-How-To-Fix-Module-Circular-Dependency/
---

Circular Dependency is a common issue when developping with multiple modules. It will block the compile process of the project. This article will introduce two different ways to solve this problem.

## Problem Reproduce
In general, this issue is happened when classes in different modules dependent on each other. Following is a UML diagram that shows the situation.

![UE-Circular-Dependency-UML](UE-Circle-Dependency-UML.jpg)
_Circular Dependency_

According to the diagram, class **A** belongs to *Module1* and class **B**/C belongs to *Module2*. Class **A** holds a reference of class **B** instance, and at the same time, function of class **A** is invoked by wrapper function of class C. To compile the code successfully, it's necessary to add *Module1* in public (or private) dependency module name list of *Module2*'s Build.cs and vise versa. In this situation, a circular dependency is established between *Module1* and *Module2*. 

![Rider-Compile-Error](Rider-Compile-Error.png)
_Compile Error of Circular Dependency_

## Static Delegate

One of the solution is using delegate instead of directly function call. Following is the new UML diagram.

![UE-Static-Delegate](UE-Static-Delegate-UML.jpg)
_Static Delegate_

As the diagram shows, a static delegate is involved as the data bridge between class **A** and **C**. Class **C** register the callback function to delegate and it will be triggered when class **A** broadcast the event. In this situation, class **C** no longer dependent on class **A** and *Module2* no longer dependent on *Module1*. The circular dependency is fixed.

## Dervided Class Override

Another solution is based on dervided class. The overview UML diagram is like this.

![UE-Dervided-Override](UE-Dervided-Override-UML.jpg)
_Dervided Class Override_

In the diagram, a new *Abstract Class* is added. This class does not provide any functionality but a framework. Class **B** dervided from the *Abstract Class* and override the *FunctionA*. The key is set the instance of class **B** to the **Instance** variable of *Abstract Class*. After that, class **C** can access the **Instance** of *Abstract Class* without dependency to *Module1*. When invoke the *FunctionA*, the overridden version from class **A** will be triggered. The circular dependency problem is no longer exists.