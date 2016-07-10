---
layout: post
title: Refactoring: The Case of the Hungry Switch Statement
---

# INTRO

To write new functionality is a lot easier and safer if you write it in blank new file, rather than if you open some existing file, try to find the place where to put the new code and make sure you did not brake the existing functionalities.

# Section one
- what this section is about
- why it matters
- research or examples
- takeaways


# Original code

# Simplify the for loop in Convert method

...

# Support adding new LcdCharacters from outside

I am thinking to extract each LcdChar from the Switch statement and put it in it's own class. 


# Refactoring step 2

# Conclusion 


Let's look at the following code:




This can be achived by applying SRP and OCP principles



  In object-oriented programming, the Open/Close Principle states that the design and writing of the code should be done in a way that new functionality should be added with minimum changes in the existing code. The design should be done in a way to allow the adding of new functionality as new classes, keeping as much as possible existing code unchanged.
  Software entities like classes, modules and functions should be open for extension but closed for modifications.
  ((OOD)[http://www.oodesign.com/open-close-principle.html])

Okay, let's see the existing code that we have here.

OCP and SRP are complementary.

|||

"Define a family of algorithms, encapsulate each one, and make them interchangeable."

This code clearly violates the open/closed principle. Every time we add new parser for we have to change the Mapper code.

# Changing requirements


 A class is open for extension when it does not depend directly on concrete implementations. Instead it depends on abstract base classes or interfaces and remains agnostic about how the dependencies are implemented at runtime.
 
 OCP is about arranging encapsulation in such a way that it’s effective, yet open enough to be extensible. This is a compromise, i.e. “expose only the moving parts that need to change, hide everything else”






  

