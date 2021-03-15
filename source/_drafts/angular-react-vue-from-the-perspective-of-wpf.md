---
title: Angular, React, Vue From the Perspective of WPF
data: 2020-08-06 16:00:00
categories:
 - Tech
---

I am well aware that various new concepts have emerged in the front-end field in recent years, and the tool chain is constantly maturing. But in this field, traditional GUI programmers are rare, and these people have a little bias towards desktop development. Therefore, today I do not want to talk about JavaScript, but concepts in GUI application field.
<!--more-->

I am not a full-time front-end developer. There may be errors or improprieties in this article. If any, please point out.

# Angular
Perhaps Angular is the most controversial of these three development libraries. According to most people’s evaluations, there are two main types of attitudes towards Angular. One is that the Angular is too complicated and rigid, just like JavaScript without script. Those who used to use JavaScript freely are now restricted by various requirements. The code structure must meet the specification of Angular. The other is that Angular is a modern front-end engineering template, which raises the lower limit of the overall code quality of the project and greatly improves maintainability.

Since I have no experience in front-end programming in the old age, I can't appreciate the freedom of JavaScript, but I can understand these people, because compared to C#, Java is indeed very rigid. As for others, they think that the learning curve of Angular is so steep that no one can get start in 10 minutes. I think this is due to their lack of modern desktop programming experience. Note that the attribute "modern" here is for the front-end field not for desktop field. In fact, these concepts are old things, and desktop programming has not improved for a long time.

Following the order of the Angular tutorial documentation, each concept will be compared with the one in wpf.

## Component
Components are very similar to controls in WPF. They encapsulate logic and are separated from templates and styles. There is a similar concept in React and they think React is relatively simple. Because the complexity of Angular is not in the concept of component, but the elements inside it.

## Interpolation
The concept of interpolation is equivalent to binding in WPF, but there are two differences here, which are their advantages respectively. First, binding is not only used for interpolation, the structural directive ngFor can also be implemented using binding, and there is no need to declare a loop variable. The second is that interpolation in Angular supports pipes. In some scenarios, pipes can significantly reduce the amount of code, but in WPF you have to use nested custom converters to achieve the same function. Due to the syntax limitation of the Markup Extension, this will make the code become lengthy.

## ngSwitch
With ngSwitch, the rendered template can be dynamically determined at runtime. In WPF, DataTemplate behaviors as the same as ngSwitch, but because XAML is a strongly typed markup language, you can enjoy the benefits of IntelliSense when writing DataTemplate.

## Event Binding
In Angular and WPF, the same term is used in how to handle events: binding, and they use the same programming model. I think that binding is a new concept for Angular, and in WPF, binding here is the same thing as binding in interpolation.

## Observable
Perhaps for some JavaScript programmers, Observable and RxJS are already indispensable things, while for others, they may have heard of RxJS and think it is a complex and difficult to use library (compared to jQuery and lodash). But in fact, the concept of Reactive Extension was originally proposed by Microsoft and implemented on the .Net platform, and RxJS was originally developed by Microsoft.

## Dependency Injection
Dependency injection is probably one of the most confusing terms for front-end programmers, because it is a back-end concept. Shouldn't it appear in the front end because this is a commonly used concept in the back end? The answer is no. The various best practices that have been summarized back-end programmers over the years must have reasons to use them, not for the sake of proposing concepts.

The core idea of ​​dependency injection is inversion of control. Every part of the program should not directly depend on others, but the process of constructing dependencies should be external. It's a bit like using different components to piece a completed program together, and the process of piecing (injection) is done by the programmer.

Although WPF is not a back-end framework, many MVVM frameworks (such as Prism) have absorbed the idea of ​​back-end programming and implemented the dependency injection function a long time ago, which can better separate presentation layer, service layer and even persistence layer when developing complex client applications. So when I saw that Angular chose dependency injection, what I felt was not unfamiliar, but comprehensively engineering.

# React

# Vue

