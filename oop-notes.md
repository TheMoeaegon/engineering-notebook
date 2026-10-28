##### OOP

> **Source / Reference:** [Boot.dev — Learn Object-Oriented Programming](https://www.boot.dev/courses/learn-object-oriented-programming)
> These are my personal notes from the Boot.dev OOP course. All credit for the course material goes to Boot.dev.
>
> **Started:** 2026-10-28
> **Progress so far:** OOP intro, Clean Code, DRY, Classes, Objects, Methods, Self, Constructors, and Class vs Instance Variables

**What is OOP?**
Object-oriented programming (OOP) organizes data and behavior into objects to (allegedly) make code cleaner and easier to maintain.

**Clean Code**
The purpose of programming paradigms like OOP is to make code easier to work with. Code that's easy for us to understand is called "clean" or "maintainable" code. Our main purpose as a developer should able be to write code that human can understand. Anyone can write a code that machine can understand. Clean code doesn't make a program faster or more correct, but it does make bugs easier to find and future changes easier to make.

**DRY Code**
One of the most famous rules for maintainable code is "Don't Repeat Yourself" (DRY): avoid writing the same logic twice! Duplicate code causes problems:
- If you need to change it, you have to change it in multiple places
- If you forget to change it in one place, you'll have a bug
- It's more work to write it over and over again
##### Classes
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
###### **Objects**
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
##### Methods
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
###### Self
Methods are defined within the class declaration. Their first parameter is always the instance of the class that the method is being called on. By convention, it's called "self", and because self is a reference to the object, you can use it to read and update the properties of the object.

Notice that methods are called directly on an object instance using the dot operator:
```python
my_object.my_method()
```
###### Method Can Return 
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
###### Method Vs Functions
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
##### Constructors
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
##### Class Variable Vs Instance Variable
###### Instance Variable
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
###### Class Variable
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
###### Which Should I Use?
Generally speaking, stay away from class variables. Just like global variables, class variables are usually a bad idea because they make it hard to keep track of which parts of your program are making updates. However, it is important to understand how they work because you may see them in the wild.