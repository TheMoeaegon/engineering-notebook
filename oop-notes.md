# OOP

> **Source / Reference:** [Boot.dev — Learn Object-Oriented Programming](https://www.boot.dev/courses/learn-object-oriented-programming)
> These are my personal notes from the Boot.dev OOP course. All credit for the course material goes to Boot.dev.
>
> **Started:** 2026-10-28
> **Progress so far:** OOP intro, Clean Code, DRY, Classes, Objects, Methods, Self, Constructors, Class vs Instance Variables, Encapsulation, Abstraction, Inheritance, and Polymorphism

## What is OOP?
Object-oriented programming (OOP) organizes data and behavior into objects to (allegedly) make code cleaner and easier to maintain.

## Clean Code
The purpose of programming paradigms like OOP is to make code easier to work with. Code that's easy for us to understand is called "clean" or "maintainable" code. Our main purpose as a developer should able be to write code that human can understand. Anyone can write a code that machine can understand. Clean code doesn't make a program faster or more correct, but it does make bugs easier to find and future changes easier to make.

## DRY Code
One of the most famous rules for maintainable code is "Don't Repeat Yourself" (DRY): avoid writing the same logic twice! Duplicate code causes problems:
- If you need to change it, you have to change it in multiple places
- If you forget to change it in one place, you'll have a bug
- It's more work to write it over and over again

## Classes
A class is a special type in an object-oriented programming language like Python. If you squint really hard, it's kinda like a dictionary in that it usually stores name-value pairs: 
```python
# Defines a new class called "Soldier"
# with three properties: health, armor, damage
class Soldier:
    health: int = 5
    armor: int = 3
    damage: int = 2
```
Just like a string, integer or float, a class is a type, but instead of being a built-in type, classes are custom types that you define.
> In Object-Oriented Programming (OOP), a class is a blueprint, template, or prototype used to create objects. It is a custom data type which store or organize the data in name value pairs and bundle the data (what the object knows) and the actions (what the object can do) or behavior which act on that data into single, reusable package. 

### Objects
So if a class is a new custom type, what's an object? Objects are just instances of a class.
```python
health = 50
# health is an instance of an integer type
aragorn = Soldier()
# aragorn is an instance of the Soldier class type
```
Each new instance of a class is an "object".

**What is the difference between dictionary vs classes?**
First of all, classes provide structure. The class definition says "hey, I have these specific properties". A dictionary is more powerful in the sense that you can store whatever you want in it, but that also makes it less clear what on earth is in there at any given time. Another thing that makes classes useful is that they can have methods!

### Methods
A method is just a function that's tied directly to a class and has access to its properties. See the take_damage method here:
```python
class Soldier:
    health: int = 5

    # This is a method that reduces the
    # health of the soldier
    def take_damage(self, damage: int) -> None:
        self.health -= damage


soldier_one = Soldier()
soldier_one.take_damage(2)
print(soldier_one.health)
# prints "3"

soldier_two = Soldier()
soldier_two.take_damage(1)
print(soldier_two.health)
# prints "4"
```

#### Self
Methods are defined within the class declaration. Their first parameter is always the instance of the class that the method is being called on. By convention, it's called "self", and because self is a reference to the object, you can use it to read and update the properties of the object.

Notice that methods are called directly on an object instance using the dot operator:
```python
my_object.my_method()
```

#### Method Can Return
If a normal function doesn't return anything, it's typically not a very useful function. In contrast, methods often don't return anything because they can mutate (update) the properties of the object instead. That's exactly what we did in the last assignment.

However, they can return values if you want! They're just functions with access to an object, after all. A common use case is a "getter" method that returns a calculated value based on the properties of the object.
```python
class Soldier:
    armor: int = 2
    num_weapons: int = 2

    def get_speed(self) -> int:
        speed = 10
        speed -= self.armor
        speed -= self.num_weapons
        return speed


soldier_one = Soldier()
print(soldier_one.get_speed())
# prints "6"
```

#### Method Vs Functions
You know what a function is, and a method is the exact same thing, it's just tied directly to a class and has access to the properties of the object.

A method auto magically receives the object it was called on as its first parameter:

```python
class Soldier:
    health: int = 100

    def take_damage(self, damage: int, multiplier: int) -> None:
        # "self" is dalinar in the first example
        #
        damage = damage * multiplier
        self.health -= damage


dalinar = Soldier()
# "damage" and "multiplier" are passed explicitly as arguments
# 20 and 2, respectively
# "dalinar" is passed implicitly as the first argument, "self"
dalinar.take_damage(20, 2)
print(dalinar.health)
# 60

adolin = Soldier()
# Again, "adolin" is passed implicitly as the first argument, "self"
# "damage" and "multiplier" are passed explicitly as arguments
adolin.take_damage(10, 3)
print(adolin.health)
# 70
```
A method can operate on data that is contained within the class. In other words, you won't always see all the "outputs" in the return statement because the method might just mutate the object's properties directly.

### Constructors
A constructor is (usually) better way to define properties in a class. It's a specific method on a class called __ init __ that is called automatically when you create a new instance of a class. 
So, using a constructor, the code from above would look like this:
```python
class Soldier:
    def __init__(self) -> None:
        self.name = "Legolas"
        self.armor = 2
        self.num_weapons = 2
```

Not only is this _safer_ (we'll talk about why later), but it also allows us to make the starting property values configurable:
```python
class Soldier:
    def __init__(self, name: str, armor: int, num_weapons: int) -> None:
        self.name = name
        self.armor = armor
        self.num_weapons = num_weapons


soldier_one = Soldier("Legolas", 2, 10)
print(soldier_one.name)
# prints "Legolas"
print(soldier_one.armor)
# prints "2"
print(soldier_one.num_weapons)
# prints "10"

soldier_two = Soldier("Gimli", 5, 1)
print(soldier_two.name)
# prints "Gimli"
print(soldier_two.armor)
# prints "5"
print(soldier_two.num_weapons)
# prints "1"
```

### Class Variable Vs Instance Variable

#### Instance Variable
Instance variables vary from object to object and are declared in the constructor. They're more common:
```python
class Wall:
    def __init__(self) -> None:
        self.height = 10  # instance variable (per object)


south_wall = Wall()
south_wall.height = 20  # only updates this instance of a wall
print(south_wall.height)
# prints "20"

north_wall = Wall()
print(north_wall.height)
# prints "10"
```

#### Class Variable
Class variables are shared between instances of the same class and are declared at the top level of a class definition. They're less common:
```python
class Wall:
    height: int = 10  # class variable (shared across all instances)


south_wall = Wall()
print(south_wall.height)
# prints "10"

Wall.height = 20  # updates all instances of a Wall

print(south_wall.height)
# prints "20"
```

>In other languages these types of variables are often called static variables.

#### Which Should I Use?
Generally speaking, stay away from class variables. Just like global variables, class variables are usually a bad idea because they make it hard to keep track of which parts of your program are making updates. However, it is important to understand how they work because you may see them in the wild.

## Encapsulation
Encapsulation is the practice of hiding complexity inside a "black box" so that it's easier to focus on the problem at hand.
The simplest example of encapsulation is a function. The caller of a function doesn't need to worry too much about what happens inside, they just need to understand the inputs and outputs.
```python
# who even knows how this function works???
# I sure don't, I just call it and assume
# it calculates the acceleration correctly
acceleration: float = calc_acceleration(initial_speed, final_speed, time)
```
To use the calc_acceleration function, we don't need to think about each line of code inside the function definition. We just need to know that if we give it the inputs:
- initial_speed
- final_speed
- time
It will produce the correct acceleration as an output.

### Public and Private
By default, all properties and methods in a class are public. That means that you can access them with the . operator:
```python
wall.height = 10
print(wall.height)
# 10
```
Private data members are a way to encapsulate logic and data within a class definition. To make a property or method private just prefix it with two underscores:
```python
class Wall:
    def __init__(self, armor: int, magic_resistance: int) -> None:
        self.__armor = armor
        self.__magic_resistance = magic_resistance

    def get_defense(self) -> int:
        return self.__armor + self.__magic_resistance


front_wall = Wall(10, 20)

# This results in an error
print(front_wall.__armor)

# This works
print(front_wall.get_defense())
# 30
```
We do this to make it easier to use our class. Now when another developer (or even ourselves) use the Wall class, they don't need to think about how armor and magic_resistance affect the defense of a Wall. In fact, we don't even allow them to access armor and magic_resistance directly by making them private with double underscore.

They simply call the public get_defense() method and know that the correct value will be returned.

### Not Security
Encapsulation is all about making code easy to work with by "hiding" complicated code within a "black box". It makes it so we don't have to worry about the details of how a function or method works... we just use it.

This is kinda like how the inner workings of your car's power steering are hidden from you. You don't need to be an expert on hydraulic systems to turn the wheel from side to side.

**Lane take on encapsulation**:
>Encapsulation and the concepts of private and public members have NOTHING to do with security. This really confused me as a new developer. Just as the casing on your computer hides its inner workings but doesn't stop you from opening the case and looking inside, encapsulation doesn't stop anyone from knowing how your code works; it just puts it all in one easy-to-find place.

Encapsulation is about organization, not security. Making a property "private" tells other developers that it's an internal detail of your class. It doesn't make the value stored in that property secret.

### Encapsulation in Python
Python is a very dynamic language, which makes it difficult for the interpreter to enforce some of the safeguards that languages like Go do. That's why encapsulation in Python is achieved mostly by convention rather than by force.

Prefixing methods and properties with a double underscore is a strong suggestion to the users of your class that they shouldn't be touching that stuff. If a developer wants to break convention, there are ways to get around the double underscore rule.
```python
class Wall:
    def __init__(self, height: int) -> None:
        # the double underscore makes this a private property
        # but it's not strictly enforced, there are hacks to get around it
        self.__height = height

    def get_height(self) -> int:
        return self.__height
```

## Abstraction
Abstraction helps us handle complexity by hiding unnecessary details.which sounds just like encapsulation, right? To be honest, the ideas are so similar that it's almost not worth worrying about the difference...almost.

### Abstraction vs. Encapsulation
- Abstraction is about creating a simple interface for complex behavior. It focuses on what's exposed (public).
- Encapsulation is about hiding internal state. It focuses on tucking away the implementation details (private).
- Abstraction is more about reducing complexity, encapsulation is more about maintaining the integrity of system internals.

The terms "abstraction" and "encapsulation" mostly just emphasize different aspects of the same concept:
- Abstraction focuses on exposing essential features while hiding complexity
- Encapsulation focuses on bundling data with methods and restricting direct access to implementation details

**Video: Encapsulation vs. Abstraction**
![[Encapsulation-Vs-Abstraction.mp4]]

## How OOP Developers Think
Classes in object-oriented programming are all about grouping data and behavior together in one place: an object. Object-oriented programmers tend to think about programming as a modeling problem. They think:

>"How can I write a Human class that holds the data and simulates the behavior of a real human?"

To provide some contrast, when functional programmers aren't busy writing white papers, they tend to think of their code as inputs and outputs, and how those inputs and outputs transition the world from one state to the next:

>"My game has 7 humans in it. When one takes a step, what's the next state of the game?"

![[OOP-Vs-Functional-Programming.png]]

### Both Paradigms Are Useful
OOP isn't the only pattern for organizing code, but it's one of the more popular ones. If you understand multiple ways of thinking about code, you'll be a much better developer overall.

## Inheritance
Inheritance allows a "child" class, to inherit properties and methods from a "parent" class. It's a way to share code between classes. For example, say we have this Aircraft class:
```python
class Aircraft:
    def __init__(self, height: int, speed: int) -> None:
        self.height = height
        self.speed = speed

    def fly_up(self) -> None:
        self.height += self.speed
```
And say we want to also model more specific kinds of aircraft. We could create a more specific Helicopter class like this:
```python
class Helicopter:
    def __init__(self, height: int, speed: int) -> None:
        self.height = height
        self.speed = speed
        self.direction = 0

    def fly_up(self) -> None:
        self.height += self.speed

    def rotate(self) -> None:
        self.direction += 90
```
Trouble is, we've rewritten a lot of the same code twice... wouldn't it be nice if a Helicopter could just take all the behavior from an Aircraft, and then just add its own unique behavior on top of that? Well, it can! We'll just make Helicopter a child class of Aircraft:
```python
class Helicopter(Aircraft):
    def __init__(self, height: int, speed: int) -> None:
        super().__init__(height, speed)
        self.direction = 0

    def rotate(self) -> None:
        self.direction += 90
```
By adding Aircraft in parentheses after Helicopter, we're saying "make Helicopter a child class of Aircraft". Now Helicopter inherits all the properties and methods of Aircraft!

The super() function returns a proxy of the parent class, meaning we can use it to call the parent class's constructor and other methods. So the Helicopter's constructor says "first, call the Aircraft constructor, and then additionally set the direction property".

Now, say we want to create a `Jet` class. Again, because all jets are aircraft, we can inherit from `Aircraft` again. One parent class can have as many child classes as you want.
```python
class Jet(Aircraft):
    def __init__(self, speed: int) -> None:
        # Jets always start on the ground
        super().__init__(0, speed)

    def go_supersonic(self) -> None:
        self.speed *= 2
```

### When to Inherit
Inheritance is a powerful tool, but do not overuse it (looking at you, Java devs). The rule of thumb is:
> A should only inherit from B if A is always a B.

![[Good-Vs-Bad-Inheritance-Hierarchy.png]]
For example:
- Cat should inherit from Animal because a cat is always an animal.
- Truck should inherit from Vehicle because a truck is always a vehicle.
- Lane should inherit from BadDotaPlayers because Lane is always bad at DotA.

But these are not true, and you'll end up with a mess if you try to force them:
- Cat should not inherit from Dog because a cat is not always a dog.
- Animal should not inherit from Cat because an animal is not always a cat.
- Allan should not inherit from Employed because Allan is not always employed.

When a child class inherits from a parent, it inherits everything. If you only want to share some functionality, inheritance should not be the tool you use. Just share some functions, or maybe make a new parent class that both classes can inherit from.

### Wide Not Deep
It's more common for an inheritance tree to be wide than deep.
![[Deep-Vs-Wide-Inheritance-Tree.png]]
In your application code, you likely won't see a `Camera` class inherit from a chain of 13 classes up to `Equipment`. If you do: _run_. There are better organizations to work for, I swear.

#### Why Wide Not Deep?
A child class must fit every parent class "type" above it. That just doesn't happen very often in a deep tree. It's more likely that you'll have a base class with many child classes that are slightly different variations of the base. They're all "siblings" of each other.

## Polymorphism
While inheritance is the most unique trait of object-oriented languages, polymorphism is probably the most powerful. It also is not particularly unique to class-based languages. Polymorphism is the ability of a variable, function or object to take on multiple forms.
- "poly" = "many"
- "morph" = "form"
For example, classes in the same hierarchical tree may have methods with the same name and signature but different implementations. Here's a simple example
```python
class Creature:
    def move(self) -> None:
        print("the creature moves")


class Dragon(Creature):
    def move(self) -> None:
        print("the dragon flies")


class Kraken(Creature):
    def move(self) -> None:
        print("the kraken swims")


creatures: list[Creature] = [Creature(), Dragon(), Kraken()]
for creature in creatures:
    creature.move()
# prints:
# the creature moves
# the dragon flies
# the kraken swims
```
Because all three classes have a .move() method, we can shove the objects into a single list, and call the same method on each of them, even though the implementation (method body) is different.

> This idea is sometimes referred to as "duck typing". If it looks like a duck, swims like a duck, and quacks like a duck, it's a duck. Or, in our example, if it has a .move() method, we can treat it like a Creature.

**Video: Polymorphism**
![[Polymorphism.mp4]]
