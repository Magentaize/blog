---
title: Annoying CollectionViewSource
date: 2019-03-04 13:52:27
categories:
 - Tech
tags:
 - UWP
---

The design of CollectionViewSource is so horrible and if you used a generic collection as its source, you cannot use code-based binding "x:Bind". If you want to make the source implement INotifyCollectionChanged, the mixture of them is so complicate.
<!--more-->

Background story: I'm coding a UWP version of "Dopamine Player", generally speaking, the main UI of it is constituted by three parts, artists' list, albums' list and tracks' list as the following figure shows. However there's a design problem that when I click one of the artists, the two remaining parts should list the corresponding items and how could I handle this?

Before I wrote this post, I have referenced some implementations of other open source music players. Most of them update each view model's data source manually due to the splitted view as MVVM requires. In my opinion, since I've choosed UWP as platform, by using binding and `INotifyCollectionChanged` making all data and views changeable automatically is a more accurate design approach. The more logic I write the more bugs and maintenance cost there are. So I decide to implement it just using built-in types.

During writing XAML, `CollectionViewSource` is the most annoying thing. I bet it has not ever be changed after been created because it already existed in WPF for many years. I try to use it in `SemanticZoom` but the x:bind expression in data template interrupt me because it not support the generic. If you want to use `SemanticZoom`, the document will tell you to use LINQ to create a `IGrouping<TKey, TElement>` variable, and the type of object contains Group is `ICollectionViewGroup`. So, what is that?? Obviously, it's not a generic type, but it's a generic `IGrouping<TKey, TElement>` in actual. Because we cannot make x:DataType to a generic, a x:bind cannot be made on it and the legacy binding is our only choice.

Although the issue of data source has been solved, the data source itself still cannot automatically change. So I need a type of `ObservableCollection<Grouping<TKey, TElement>>` which `Grouping<TKey, TElement>` is based on `ObservableCollection<TElement>` and implements `IGrouping<TKey, TElement>`. And `CollectionChanged` event of original data source collection should be handled inside it so the grouped items can be modified when original data source has changed.
	