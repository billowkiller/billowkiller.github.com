---
layout: post
title: "Design Pattern--Principle"
date: 2013-05-14 01:09
comments: true
categories: book
tags: [book, programming, Design Patterns]
---

Head First Design Patterns

* * * * *

 

​1. Identify the aspects of your application that vary and separate them
from what stays the same.

　　Take the parts that vary and encapsulate them, so that later you can
alter or extend the parts that vary without affecting those that don't.

 

​2. Program to an interface, not an implementation.

　　Our intuition to solve problems is relying on an implementation,
either in the superclass or in the subclass itself. We were locked into
using that specific implementation and there was no room for changing
out the behavior other than writing more code.

　　With a new design, the subclasses will use a behavior represented by
an interface, so that the actual implementation of the behavior won't be
locked.

　　ps: *program to an interface *really means *program to a
supertype. *In JAVA, it means interface or abstract class.

　　　　Even better, you can assign the concrete implementation object
at runtime.

 <!--more-->

​3. Favor composition over inheritance.

　　Creating systems using composition gives you a lot more flexibility.
Not only does it let you encapsulate a family of algorithms into their
own set of classes, but it also lets you change behaviror at runtime.

 

​4. Strive for loosely coupled designs between objects that interact.

　　When two objects are loosely coupled, they can interact, but have
very little knowledge of each other.

　　Loosely coupled designs allow us to build flexible OO systems that
can handle change because they minimize the interdependency between
objects.

 

 

 
