---
layout: post
title:  "Self-registering factory pattern and how we get there"
date:   2026-08-04
categories:
  - cpp
tags:
   - design
---

Consider a codebase that has a number of different types of derived objects (for the lack of a better example, say "shapes"). We want to be able to construct the objects based on user input.

```c++
  class Circle : class Shape
  {
      ...
    private:
      int _radius;
  }

  std::unique_ptr<Shape> myShape = createShape("Circle", /* params= */ "radius=5"); // the argument comes via a user-input so we could not have known that we needed to call createCircle() for instance.
```

We require myShape to point to a `Circle`. Let's try to imagine what createShape would look like -

```c++
  std::unique_ptr<Shape> createShape( std::string_view shape_name, std::string_view params )
  {
    if (shape_name == "Circle" )
    {
      return makeCircle(params);
    }
    else if ( shape_name == "Rectangle" )
    {
      return makeRectangle(params);
    }
    ...
  }
```

Everytime a new Shape is added, one would need to remember to update createShape function, which is artificially adding a coupling where all shapes need to be aware of each other. This does not follow OCP well.

Perhaps this can be made simpler if we have a registry of the factories that stores these function pointers?

```c++
  using ShapePtr = std::unique_ptr<Shape>;
  using ShapeFactoryFunction = ShapePtr (*)(std::string_view params);

  class Registry{
      public:
        static Registry& instance();

        add_factory( std::string name, ShapeFactoryFunction fnPtr );
        ShapePtr create( std::string_view, std::string_view );
      private:
        std::map<std::string, ShapeFactoryFunction> _map{};
  };

  int main()
  {
      // somewhere in the beginning of your code-path, before any call to call_factory
      Registry::instance().add_factory("Circle", createCircle);
      Registry::instance().add_factory("Rectangle", createRectangle);

      auto shape = Registry::instance().create("Circle", "radius=5");
  }
  
```
We have found ourselves the registration pattern. However, this is still very much manual.
We need to manually update an initialisation section every time we add a new factory. What we require is to be able to do this initialisation _early enough_ and ideally automatically. 

The problem of registering _early enough_ can be solved with the help of **static initialisation**.

## Static storage duration and initialization

The key idea is that objects with [_static storage duration_](https://www.learncpp.com/cpp-tutorial/scope-duration-and-linkage-summary/) are created before `main()` begins.

For example:

```cpp
class Logger
{
public:
    Logger()
    {
        // setup work
    }
};
//local scope
{
  static Logger logger; // global variable => static duration
}
```

The object logger is not created when execution reaches a line inside main(). Instead, its lifetime begins before main() is called.

We can use this behaviour for registration.

```c++
class ShapeRegistrar
{
public:
    ShapeRegistrar(std::string name, ShapeFactoryFunction fn)
    {
        Registry::instance().add_factory(name, fn);
    }
};

static ShapeRegistrar registrar("Circle", createCircle);
```

Here registrar is a global object with static duration. Its constructor is executed during program startup[^1]. Instead of having a central initialization section, each translation unit that implements a shape can register itself.

For example, `Circle.cpp` can contain:

```cpp
static ShapeRegistrar registrar("Circle", createCircle);
```
Now the existence of the Circle type and its registration live together.

[^1]: Static registration usually works because implementations initialize namespace-scope (global) objects before entering main(). However, the C++ standard has subtle rules around deferred dynamic initialization, so code should avoid depending on initialization order between unrelated translation units.


