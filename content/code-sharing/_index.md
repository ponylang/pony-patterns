---
title: "Overview"
section: "Code Sharing"
layout: overview
menu:
  toc:
    identifier: "code-sharing-overview"
    parent: "code-sharing"
    weight: 1
---

Pony is an object-oriented language. Data and behavior are co-located classes and actors. However, Pony's OO is different than the OO you might be used to.
Pony favors composition over inheritance.

What does it mean to favor composition over inheritance? Well, for starters, Pony has no inheritance. When you favor composition over inheritance, you favor "has-a" relationships over "is-a" relationships.

"is-a" relationships are the common starting point for teaching object-orientation. If you are reading this after writing some Pony, you know that Pony makes expressing such relationships more difficult. We'll leave the discussion of why to another time.

In this chapter, we are going to explore ways to "share" and "reuse" code in Pony. We are also going to look at patterns you can follow to implement reuse and code organization methods that you've used in other languages.

Some of those patterns are ones that you commonly see to allow code reuse via "is-a" relationships. Just because Pony favor composition, doesn't mean that some of those inheritance tricks that you might know and love are unavailable to you. You will, however, need to keep an open mind and approach them a little differently than you are used to.

In this chapter we'll cover...

## Notifier

A callback based approach that allows you to create mini-frameworks allowing for specialization of your code. The notifier pattern is used throughout the Pony standard library.

## Inheritance

Pony doesn't do inheritance, but with default implementations for traits and interfaces you can get pretty close.

## Mixin

The mixin pattern is like the inheritance pattern but "even more so". If you are looking for a full-on Java-style "class A extends class B" type experience, then the mixin pattern is what you are looking for.

## Global Function

Does your favorite language have bare functions of some sort? Do you wish Pony had bare functions or perhaps... global functions? If so, the global function pattern is for you.
