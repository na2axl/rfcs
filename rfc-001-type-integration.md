# GObject Type Integration
A detailed technical proposal for integrating Gir.Core with the GObject type system.

***NOTE:** This document is incomplete. I will try to add to it as more features/issues are discussed/discovered.*

## Table of Contents
- [GObject Type Integration](#gobject-type-integration)
  - [Table of Contents](#table-of-contents)
  - [Key Terms](#key-terms)
  - [Proposal](#proposal)
  - [Issue A: Object Lifetimes](#issue-a-object-lifetimes)
    - [Wrapper Types](#wrapper-types)
    - [Subclass Types](#subclass-types)
      - [Toggle References](#toggle-references)
  - [Issue B: Object Creation for Subclasses](#issue-b-object-creation-for-subclasses)
    - [Subclass Constructors](#subclass-constructors)
    - [Factory Methods](#factory-methods)
      - [Constructor](#constructor)
      - [Option B: Factory Pattern](#option-b-factory-pattern)
      - [Extra: Autoconnecting Properties](#extra-autoconnecting-properties)
  - [Issue C: GtkBuilder and Composite Templates](#issue-c-gtkbuilder-and-composite-templates)
  - [Issue D: Validation](#issue-d-validation)

## Key Terms
A GObject in gir.core consists of two parts:
 * A ***Native GObject***, which refers to the type provided by GLib in C.
 * A ***Proxy Object***, which is the C# representation of the GObject. The proxy object 'owns' the native GObject.

A Proxy Object can exist in two forms:
 * A ***Wrapper*** over a type defined in another language (GtkWindow, HdyPaginator, etc)
 * A ***Subclass*** representing a C# type that inherits from a Wrapper type (MyWindow, MyPaginator, etc).

## Proposal
For all user-defined C# classes that derive from `GObject.Object`, the library will automatically create a corresponding GObject behind the scenes. This enables us to implement the following features:

 * **GObject Properties** - Subclasses can expose properties to the gobject type system, allowing for them to be viewable/changeable in GtkInspector, etc.
 * **Overriding Virtual Functions** - Subclasses can override virtual functions ([See GtkWidgetClass](https://developer.gnome.org/gtk4/unstable/GtkWidget.html#GtkWidgetClass)). For example, creating a custom container widget in Gtk with full control over sizing and layout.
 * **Composite Template Support** - Supports binding a subclass to a composite template defined in an XML file. Particularly useful for complex custom widgets.
 * **Bidirectional Bindings** - The *possibility* of using C# defined types from C and/or other language bindings.

In order to implement this, there are a few areas to address. These will need to be solved to achieve the above.

## Issue A: Object Lifetimes
We will recieve pointers to abitrary GObject-based types from functions. For example, `gtk_widget_get_toplevel()` will likely return a pointer to a `GtkWindow` or window subclass `MyWindow`. As we are supporting GtkBuilder, we cannot assume that *we* directly created this object. Thus we need a way to wrap a pointer to a GObject in a type we can return to the user.

### Wrapper Types
***Note:** We already do this!*

This is straightforward for wrapper types. We simply define a `Wrapper(IntPtr handle)` constructor or `NewFromPtr(IntPtr handle)` static method which creates a new instance of the wrapper bound to the given handle. In the `GObject` class, we already allow for GObjects to be created from `IntPtr`.


### Subclass Types
For subclasses, we *could* simply require a constructor/method with the same signature. However, this implies that a user-defined subclass (e.g. `class MyWindow : Gtk.Window`) can be destroyed while the underlying GObject remains active, such that we could recieve a pointer to it in the future.

This is problematic as if we have ***Custom State*** on the C# type (e.g. we have set some member variable `myBool` to `false`), this will be lost upon re-creating the type. Additionally, this negatively impacts *Object Creation* (see later).

A better solution which avoids this problem entirely is to simply require that any Subclass we create will **always** outlive the underlying GObject. Therefore, if we recieve a pointer to a Subclass, we simply retrieve the Subclass by querying the object dictionary with the pointer.

#### Toggle References
***Note:** This is an explanation of ToggleRefs to the best of my knowledge. No guarantees this is correct!*

Toggle references (`ToggleRef`) are a GObject construct that can toggle between being a strong and weak reference. When the refcount is greater than one, the reference to the proxy object is strong. When the refcount is *exactly* one, the proxy object is held by a weak reference. Effectively, they are a way to let our Subclass know whether it can be garbage collected.

**Scenario:** Take the subclass proxy object `MyButton` (in C#) that maps to the native GObject `cs_mybutton` (in C; name for illustrative purposes). `cs_mybutton` is referenced by two objects, one of which is `MyButton`, and the other is the button's parent widget.

The ToggleRef ensures that `cs_mybutton` holds a strong reference to the proxy object `MyButton`. Even if no other C# object references `MyButton`, it cannot be garbage collected because `cs_mybutton` forces it to remain alive.

Now, the widget's parent is destroyed, and consequently it releases its reference to `cs_mybutton`. `cs_mybutton` is now only referenced by `MyButton`; this is where the 'Toggle' part comes in. The ToggleRef toggles the reference from strong to weak, meaning that if no other object in C# holds a reference to `MyButton`, the type can now be safely garbage collected.

Upon garbage collection, `MyButton` is destroyed and releases it's strong reference to `cs_mybutton`. `cs_mybutton` is then finalised as zero objects reference it.

As we can see, ToggleRefs will allow us to ensure that our wrapper lifetime matches that of the object it references.

## Issue B: Object Creation for Subclasses
As soon as we define a custom `GType` for subclasses, we need some way of creating objects with the generic `g_object_new(params)` method. The solution to this would be to have all GObject-based types define a ‘Factory Method’. This takes in properties and returns an instance of the object.

### Subclass Constructors
The constructor of a subclass usually serves to set up important state, like connecting signals. For this reason, we cannot bypass the C# constructor for subclasses.

My proposed solution is to override the `constructor` method of `GObjectClass`, which is called before the object instance is created. We will 'hijack' it to call the C# type's factory method. If the object is being created from C#, indicated by a special "inhibit" property, we simply chain up to the original constructor and return.

**Fun Fact:** This is how singletons can be implemented with GLib.

```cs
public static IntPtr ConstructorOverride<T>(GType type, uint n_props,
                                            IntPtr construct_properties)
{
    // Detect the caller
    if (ContainsProp(construct_properties, "inhibit"))
    {
        // We are calling from C#, create object as normal
        return constructor(type, n_props, construct_properties);
    }
    else
    {
        // We are calling from outside
        // Create new C# object using the factory method
        var obj = new T.FactoryMethod(
            new PropertySet(
                n_props,
                construct_properties
            )
        );

        // Return the gobject pointer transparently
        return obj.Handle;
    }
}
```

I believe this is similar to what GtkSharp does, but differs in some areas with significantly less legacy/compatibility code.

### Factory Methods
The issue now is not how to ensure constructors are called, but rather how to create a unified interface for creating GObject-derived classes. Using the Factory pattern as inspiration, here are some possible solutions:

#### Constructor
The user must either define:
 1. An empty constructor
 2. A constructor with the signature `Constructor(params Property[] properties)`

```cs
public MyWindow(params Property[] properties)
{
    // Allow the user to process properties
    foreach (var prop in properties) { ... }
}
```

This is the easiest to implement for the user, but we have no way of enforcing this at compile time (Unless we run a validation tool?).

Optionally this could be a static function instead, although the difference
is minimal.

#### Option B: Factory Pattern
We implement a `GObjectFactory` class/interface which can create any GObject given its type. Even if we use the constructor pattern above, it might be worth creating this anyway to abstract over GObject creation.

#### Extra: Autoconnecting Properties
Using reflection, we may be able to automatically assign properties for the user, making the constructor more ergonomic to write:

```cs
public MyWindow(params Property[] properties) : this(...)
{
    Property.Autoconnect(this, properties);
}
```

## Issue C: GtkBuilder and Composite Templates
Gtk supports two types of Builder UI files:
 - ***Normal Templates*** which define a series of objects in an XML-based format.
 - ***Composite Templates*** which define an individual widget class's components. Unlike normal templates, composite templates are tied to an object's class.

Currently, we support only "Normal Templates". This is perfectly fine for the normal intended usage of Builder (e.g. create a Builder object and retrieve a fully constructed object from it), however we are using it to implement a form of half-baked composite templates instead.

We pass in a builder object or ui file to the constructor of a subclass as follows:
```cs
public class DemoWindow : Gtk.ApplicationWindow
{
    public DemoWindow(Application application) : base(application, "demo_window.glade")
    {
        // Setup
    }
}
```
This is not ideal as firstly templates are tied to the instance instead of the class; and secondly, it creates "constructor soup", where we have a mess of different constructors for wrapper types.

See: `Gtk3/ApplicationWindow.cs`:
```cs
public ApplicationWindow (Application application, string template, string obj = "root") : base(template, obj, Assembly.GetCallingAssembly())
{
    // Wrapper Init
}
```

Instead, my proposal is to properly implement composite types using attributes instead (only possible with type integration &mdash; we need `class_init()` to bind the template):
```cs
[Gtk.Template("ui.glade")]
public class DemoWindow : Gtk.ApplicationWindow
{
    public DemoWindow(Application application)
    {
        // Setup
        this.Application = application;
    }
}
```

This way, composite templates are correctly tied to the class rather than the instance. Gtk.Builder still functions in the procedural style:

```cs
// Procedural Style
var builder = new Gtk.Builder("window.ui");
var Window = builder.GetObject();
...
```

## Issue D: Validation
The bindings (and all code for that matter) should be verifiable at compile time. There should be no “Unexpected” behaviour, whether that be exceptions for ill-defined types or otherwise. We could possibly have a validation task that runs as part of the build process and verifies that all objects defined in the assembly satisfy the requirements. Better yet, try avoiding this entirely.
