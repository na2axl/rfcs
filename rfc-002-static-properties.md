# Static GProperty Descriptors

A detailed technical proposal to integrate static GProperty descriptors into Gir.Core &mdash; 
**See TODO:LINK_TO_ISSUE**

## Summary
- [Static GProperty Descriptors](#static-gproperty-descriptors)
  - [Summary](#summary)
  - [Terminology](#terminology)
  - [Proposal](#proposal)
  - [Goal A : Static Property Descriptor](#goal-a--static-property-descriptor)
  - [Goal B : Know the C# type of the **GProperty**](#goal-b--know-the-c-type-of-the-gproperty)
  - [Goal C : Know the C# type of the **GObject** in which the **GProperty** will be registered](#goal-c--know-the-c-type-of-the-gobject-in-which-the-gproperty-will-be-registered)
  - [Goal D : Know the name of the **GProperty**](#goal-d--know-the-name-of-the-gproperty)
  - [Goal E : High-level API to get/set the **GProperty** value](#goal-e--high-level-api-to-getset-the-gproperty-value)
  - [Goal E : Know if the **GProperty** is write-only, read-only, or read-write](#goal-e--know-if-the-gproperty-is-write-only-read-only-or-read-write)
    - [Explicitly defined](#explicitly-defined)
    - [Implicitly defined](#implicitly-defined)
    - [Pro and Cons of explicit and implicit methods](#pro-and-cons-of-explicit-and-implicit-methods)
  - [Goal F : Allow easy registration and initialization of a **GProperty** into a **GObject**](#goal-f--allow-easy-registration-and-initialization-of-a-gproperty-into-a-gobject)

## Terminology

This RFC will talk about three main subjects:

* **GProperty**: The native C property registered into the **GObject**.
* **Property Descriptor** or **Static Property Descriptor** or simply **Static Property**: The representation/description of the **GProperty** behavior in the C# **Wrapper** or **Subclass**.
* **Property**: Serves as the high-level user API to `get` and `set` the value of the **GProperty**, through his respective **Static Property Descriptor**.

> For the definition of terms **GObject**, **Wrapper**, and **Subclass**, see [RFC001]

## Proposal

To allow Gir.Core be a part of the current C# UI ecosystem, the implementation of **GProperties** can be done using **Static Properties**, to match what it's done now using `DependencyProperty` in other libraries.
For that we have the following goals:

* Declare **GProperties** of current **Wrappers** using **Static Properties**;
* Allow library users to declare **GProperties** of theirs **Subclass** using **Static Properties**;
* An high-level API will provide access to the **GProperty** value, to retrieve and edit it through the corresponding **Property Descriptor**.
* **Property Descriptors** must know the type (the C# type) of the **GProperty**;
* **Property Descriptors** must know the type (the C# type) of the **GObject** in which it will be registered;
* **Property Descriptors** must know the name of the **GProperty**;
* **Property Descriptors** must know if the **GProperty** is write-only, read-only, or read-write;
* **Property Descriptors** must allow an easy registration and initialization of the **GProperty** in the **GObject**;

All of these points must be implemented in an easy to use, `DependencyProperty`-like manner.

## Goal A : Static Property Descriptor

The **Property Descriptor** must be declared statically, why the term **Static Property Descriptor**.

A `Property` class can be created for that. To be `DependencyProperty`-like, the constructor will be made private
and a `Register` static method will be created to actually _register_ the **GProperty**.

```cs
public class Property
{
  private Property();

  public static Property Register();
}

// Example 01
public static readonly Property TextProperty = Property.Register();
```

## Goal B : Know the C# type of the **GProperty**

This can be done by adding a type parameter to the `Property` class. Then other implementations can use this parameter to
know the C# type of the **GProperty**, and maybe do a mapping from a _type dictionary_ to guess the native C type if necessary.

```cs
public class Property<T>
{
  private Property();

  public static Property<T> Register();
}

// Example 01
public static readonly Property<string> TextProperty = Property<string>.Register();
```

## Goal C : Know the C# type of the **GObject** in which the **GProperty** will be registered

Since this goal can be mainly for registration purposes, we can add to the actual `Register` static method a type parameter.
Using a _type dictionary_ it will be easy to retrieve the corresponding GLib type from that C# type for further processing.

```cs
public class Property<T>
{
  private Property();

  public static Property<T> Register<TObject>() where TObject : GObject.Object;
}

// Example 01
public static readonly Property<string> TextProperty = Property<string>.Register<Label>();
```

We can also add this type parameter directly to the `Property` class. This will be helpful if in that class, a field/property
have to know the type of the **Wrapper**/**Subclass**.

```cs
public class Property<TObject, T>
   where TObject : GObject.Object
{
  private Property();

  public static Property<TObject, T> Register();
}

// Example 01
public static readonly Property<Label, string> TextProperty = Property<Label, string>.Register();
```

## Goal D : Know the name of the **GProperty**

For this we can simply add a read-only property into the **Property Descriptor** to store the **GProperty** name,
which will be filled at the registration time, so the `Register` static method will also take this name as a parameter.

```cs
public class Property<T>
{
  /// <summary>
  /// GProperty name.
  /// </summary>
  public string Name { get; private set; }

  private Property();

  public static Property<T> Register<TObject>(string name) where TObject : GObject.Object;
}

// Example of Property Descriptor declaration
public static readonly Property<string> TextProperty = Property<string>.Register<Label>("text");
```

## Goal E : High-level API to get/set the **GProperty** value

We can tweak the existing `GetProperty` and `SetProperty` methods on the **GObject** to take as parameter a **Property Descriptor**.
So we will know exactly on which **GProperty** we have to get/set the value.

```cs
public partial class Object
{
  /// <summary>
  /// Returns the value of the GProperty described by <paramref name="prop"/>.
  /// </summary>
  public T GetProperty<T>(Property<T> prop);

  /// <summary>
  /// Sets the <paramref name="value"/> of the GProperty described by <paramref name="prop"/>.
  /// </summary>
  public void SetProperty<T>(Property<T> prop, T value);
}
```

Then in user code, to retrieve the **GProperty** value we just use standard properties, which will act as proxies for the corresponding
**Property Descriptor**.

```cs
public static readonly Property<string> TextProperty = Property<string>.Register<Label>("text");

public string Text
{
  get => GetProperty(TextProperty);
  set => SetProperty(TextProperty, value);
}
```

## Goal E : Know if the **GProperty** is write-only, read-only, or read-write

There are two ways to do that, but only one can be chosen.

### Explicitly defined

Here the user is the one who define if the **Property Descriptor** is write-only, read-only, or read-write. Everything
is defined inside the `Property<T>.Register()` static method using parameters.

```cs
public class Property<T>
{
  /// <summary>
  /// GProperty name.
  /// </summary>
  public string Name { get; private set; }

  /// <summary>
  /// Property name.
  /// </summary>
  public string PropertyName { get; private set; }

  /// <summary>
  /// Is GProperty readable?
  /// </summary>
  public bool IsReadable { get; private set; }

  /// <summary>
  /// Is GProperty writeable?
  /// </summary>
  public bool IsWriteable { get; private set; }

  private Property();

  public static Property<T> Register<TObject>(string name, string propName, bool write = true, bool read = true) where TObject : GObject.Object;
}

// Example 01
public static readonly Property<string> TextProperty = Property<string>.Register<Label>("text", nameof(Text), write: true, read: true);
public string Text
{
  get => GetProperty(TextProperty);
  set => SetProperty(TextProperty, value);
}

// Example 02
public static readonly Property<Widget> ChildProperty = Property<Widget>.Register<Container>("child", nameof(Child), write: true, read: false);
public Widget Child
{
  set => SetProperty(ChildProperty, value);
}
```

With this method, the user has the ability to define exactly if the **GProperty** is readable or writeable, without doing nothing more.

### Implicitly defined

Here the readability and writeability of the **GProperty** is defined implicitly by the code, by checking what get/set callback is
null when the user register his **GProperty**.

```cs
public class Property<T>
{
  /// <summary>
  /// Property getter.
  /// </summary>
  private Func<GObject.Object, T> _get;

  /// <summary>
  /// Property setter.
  /// </summary>
  private Action<GObject.Object, T> _set;

  /// <summary>
  /// GProperty name.
  /// </summary>
  public string Name { get; private set; }

  /// <summary>
  /// Is GProperty readable.
  /// </summary>
  public bool IsReadable => _get != null;

  /// <summary>
  /// Is GProperty writeable.
  /// </summary>
  public bool IsWriteable => _set != null;

  private Property();

  /// <summary>
  /// Get the value of this property in the given object.
  /// </summary>
  public T Get(GObject.Object o) => _get(o);

  /// <summary>
  /// Set the value of this property in the given object
  /// using the given value.
  /// </summary>
  public void Set(GObject.Object o, T v) => _set(o, v);

  public static Property<T> Register<TObject>(string name, Func<TObject, T> get = null, Action<TObject, T> set = null) where TObject: GObject.Object;
}

// Example of use 1
public static readonly Property<string> TextProperty = Property<string>.Register<Label>("text", get: (o) => o.Text, set: (o, v) => o.Text = v);
public string Text
{
  get => GetProperty(TextProperty);
  set => SetProperty(TextProperty, value);
}

// Example of use 2
public static readonly Property<Widget> ChildProperty = Property<Widget>.Register<Container>("child", set: (o, v) => o.Child = v);
public Widget Child
{
  set => SetProperty(ChildProperty, value);
}
```

With this method the user don't only care about read-write mode of the **GProperty**, but also it define what should exactly happen
when Gir.Core is processing [bindings] on that property.

### Pro and Cons of explicit and implicit methods

The only advantage of explicit method is to be... explicit. With that, the name of the **Property** have to be registered too,
because Gir.Core will use this name with the [reflection API](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection)
to be able to properly process [bindings]. This will lead to more usage of reflection (performance hit), and limit the number of customizations the
user can do when getting/setting their property values from [bindings].

This is where the implicit method rocks. With it we can totally avoid the use of reflection, and directly call `Get` and `Set` from
the **Property Descriptor** (eg. `TextProperty.Get(myTextWidget)`). The only drawback is that the user can write repetitive code
(`Register("name", get: (o) => o.PropertyName, set: (o, v) => o.PropertyName = v)`) for `get` and `set` parameters.
But this is already something that other libraries are doing, like [Avalonia](https://avaloniaui.net), which is highly used now.

## Goal F : Allow easy registration and initialization of a **GProperty** into a **GObject**

> This goal depend on [RFC001]

A **Static Property Descriptor**, like his name said, is a static property. So it will be registered prior to the **GObject** initialization,
only once, to _describe_ the **GProperty** to initialize to the **GObject** it depend on. Then this property could be used with functions
like `class_init`, `g_object_class_install_properties`, or `g_object_new_with_properties` to create the proper **GObject**.

> See [RFC001] to learn more about **GObject** type initialization.

[bindings]: ./rfc-003-mvvm-binding.md
[RFC001]: ./rfc-001-type-integration.md