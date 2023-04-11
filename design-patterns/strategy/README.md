# Strategy

## Definition
Strategy is a behavioral design pattern that lets you define a family of algorithms, put each of them into a separate class, and make their objects interchangeable so that the algorithm
varies independently from clients that use it.

## Pain Points
- **different implementation logic for different concrete classes**

    For example, Different animal species have different `bark()` behavior. You might want to solve this by using *inheritance* - creating an abstract class called *Animal* then having concrete animal subclasses implement their `bark()` method.

    Now, some animal species may have the same `bark()` behavior but we have no choice but to duplicate that code in each concrete class. This will lead to a maintenance nightmare. 

    As a software engineer, you should know the next step is to move the implementation somewhere else and call it whenever needed.

- **more behaviors need to be added**

    We need to add a behavior called `eat()` and now again we need to add an implementation to all concrete animal classes. 