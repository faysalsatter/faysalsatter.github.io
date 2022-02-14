---
layout: post
title: "Design Pattern"
subtitle: "This is about how python store certain numbers and strings in its memory."
category: Python
tags: [python, pattern, objects, singletons, interns]
---

In software engineering, the term "Design Pattern" describes an established solution to a commonly occurring problem in software design. These patterns evolve over the time by adapting the experienced software developers best practices. In general, a design pattern describes a problem, a template of how to solve it in different situations, and the consequences it entails.

Based on the complexity, scalability and other factors, design patterns can be  categorized into three main categories: 

1. Creational Design Pattern
2. Structural Design Pattern,
3. Behavioral Design Pattern. 

## Creational Design Pattern

We can create objects by simply calling their constructor. But sometimes we might need more flexibility in how the objects are created. Creational Design Patterns, as the name suggests, deal with object creation process with more control and flexibility. They play important role in reducing software complexities and instability. 

### Singletons

Singleton is one type of creational design patterns that ensures one object exist only once and at one point in memory for the lifetime of the program. It provides a single point of access to it. Every time a singleton object is called, it returns the exact same object from the memory. It will save time and memory costs.

We will see how singletons (and interning) work in Jupyter python notebook in the  [following post]({% post_url 2021-08-14-singletons_interning %}).


