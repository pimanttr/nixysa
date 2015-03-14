# Nixysa Intro #

Nixysa is a lot like swig to create Javascript bindings (via NPAPI) to C++. Given C++ (and some glue metacode), it'll write all the NPAPI for you.


## The simple example ##

First start with our HelloWorldWalkThru. It's only a few lines of code and will explain each step and get you familiar with the Nixysa basics.


## The slightly more complex example ##

A simple example that exports Complex number math is in the Nixysa source under `examples/Complex` - after the above intro, it should be easy to follow it.


## Returning Objects ##

If you want to return an to Javascript, you have to make a class with setters and getters. Those setters and getters need to by types that Nixysa understands, or other classes which you've defined.

Lets's say you want to return a simple object that has a bool and a string. How would you do that? You'd define a class like this, lets say in `types.h`:

```
class MyObj {
 public:
  MyObj()
       : is_good_(false) {
  }

  bool is_good() const {
    return is_good_;
  }

  bool set_is_good (bool is_good) {
    is_good_ = is_god;
  }

  const std::string& output() const {
    return output_;
  }

  void set_output(const std::string& output) {
    output_ = output;
  }

 private:
  bool is_good_;
  std::string output_;
};
```

**NOTE**: If you use the defaluts, the getter function must match this naming convention - the private member variable without the trailing underscore, and the setter must match the convention of `set_` followed by the name of the getter. However it is possible to specify the name of the getter and setter (see below).

And then you'd write the IDL for it, let's say in `types.h`.

```
[binding_model=by_value, nocpp, include="types.h"] class MyObj {
  [getter] bool is_good_;
  [getter] std::string output_;
};
```

In this case I don't want JS to be able to change things, so I've set only `getter`... you could use `[getter, setter]` to export both read and write operations to Javascript.

If you want to change the name of the getter or setter in your C++ class, you can specify it in the IDL:

```
[binding_model=by_value, nocpp, include="types.h"] class MyObj {
  [getter=Value, setter=SetValue] bool value_;
};
```

That will work with a C++ class defined as follow:

```
class MyObj {
 public:
  bool Value() { return value_; }
  void SetValue(bool value) { value_ = value; }
 private:
  bool value_;
};
```


Now you can return MyObj from classes - but you'll have to write a small bit of glue code for functions that return this. For example, lets say you have this function in `helloworld.cc`:

```
MyObj HelloWorld::Cool() {
  MyObj ret;
  ret.set_is_good(true);
  ret.set_output("Some wonky stuff");
  return ret;
}
```

The IDL looks the same...

```
  [const] MyObj Cool();
```

Don't forget to add your new idl file to `IDL_SOURCES` in `SConstruct` and include your new header file in the relevant C++ files.

To use this from JS, it would look liek this:

```
var foo = hw.cool()
if (foo.isGood) {
  alert(foo.output);
}
```

**NOTE**: That the namespace has again been mangled to match standard javascript naming.

## Arrays ##

Nixysa supports arrays. The idl for the javascript is simply `std::string[] bar` and this will get converted to a `std::vector<std::string> bar`.

## Passing By Reference ##

The IDL for reference pass is the same. For example, your C++ might be:

```
void foo(const std::vector<std::string> &bla);
```

The IDL is simply:

```
void foo(std::string[] bla);
```


## Slightly more complicated method calls ##

So what if the C++ function you want to call takes the object by reference instead of returning it? You can do this with a bit of IDL glue. We'll assume here the C++ prototype is `void Cool(MyObj *foo)`

```
  [const, userglue] MyObj Cool();

  [verbatim=cpp_glue] %{
    MyObj userglue_method_Cool(HelloWorld* self) {
      MyObj ret;
      self->Cool(&ret);
      return ret;
    }
  %}
```

Not here that the prototype we export to javascript is still the same - it returns the object and takes no arguments. We then wrap the C++ to create our object and pass in a pointer to it.

You can add as many glue methods inside of the `verbatim` block as needed. This takes the pointer to object, calls it in C++ and returns the value to the NPAPI glue which will then due the necessary conversions to JS.

Obviously you could write a more interesting C++ class which would be more dynamic if you prefer.

## Class Inheritance ##

Lets say you like your class `MyObj`, but want another class `MyBiggerObj` which inherits from `MyObj`. In C++ this is easy:

```
class MyBiggerObj : public MyObj {
...
}
```

And in IDL, the same thing is supported:

```
[binding_model=by_value, nocpp, include="types.h"] class MyBiggerObj : MyObj {
...
}
```

Of course, you have to have defined the base class in IDL - and there's no "public" keyword here: since IDL only deals with interfaces, public inheritance is assumed.

## Exceptions ##

There is currently no support for throwing JS exceptions, you should use properties of the JS objects you export to do this.

## Binding Models ##

TODO(piman)

## More Complicated Stuff ##

Until more documentation is written, see the [http://src.chromium.org/svn/trunk/src/o3d/plugin/idl/[o3d](o3d.md)] code for examples of more complicated stuff. Since Nixysa was written for o3d, it exercises pretty much every part of Nixysa.