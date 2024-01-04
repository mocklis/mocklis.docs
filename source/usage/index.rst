=============
Using Mocklis
=============

Generating Mocklis classes
==========================

The Mocklis code generator takes a class which implements one or more interfaces and replaces the contents of that class
with valid implementations of the interface memebers along with hooks through which you can control the behaviour of those
members on a case by case basis. It does this by means of an analyzer/code-fix pair. Any class that is decorated with the
``MocklisClass`` attribute will make the code-fix available.

Note: From version 1.4.0 (currently in alpha), Mocklis has a source generator that can create the boilerplate code for you and
keep it updated. In order to allow it to do this, you add a ``MocklisClass`` attribute as shown below, and make the class
'partial'. It does not need to have a body, so the code will normally look something like this:

.. sourcecode:: csharp

    [MocklisClass]
    public partial class MockMyViewModel : IMyViewModel;

The code fix appears in Visual Studio as an underline of the first couple of characters in the attribute. Hover over these
with the mouse, click on the arrow next to the light-bulb that appears and select 'Update Mocklis Class'.

.. image:: generate.png

*If you find it a bit tricky to find the right spot to hover over, you can change the 'Severity' of the rule in your project as follows: In the
solution explorer, expand References, Analyzers and Mocklis.MockGenerator. Right-click on the MocklisAnalyzer, and choose `Set Rule Severity`.
Choose `Warning` and your entire MocklisClass will be underlined with a green squiggly line and you can update the class from
anywhere.*

The code generator will make use of any using statements it can find to storten names of types. It will not add any of its own, so if you
generate a class and find that you end up with a lot of fully qualified types you can just add the relevant using statements, and then
re-generate the class.

The details of the code generation can be found in the :doc:`../reference/index` section.

Attribute parameters
--------------------

The generated code can be customised to a degree using properties on the ``MocklisClass`` attribute. The first two we'll look at
deal with how a mock should act when it's got an incomplete or missing configuration, and the other two deal with generating code
for methods returning values by reference.

One caveat: you can only set these properties to `true` or `false`. Anything else, like expressions, references to constants etc. will
be interpreted as the default value for the parameter. Mocklis reads the parameters from code that has been parsed and analyzed, but
it's not running code. While it may be theoretically possible to do the required evaluations from the semantic model this has not been
attempted.

Strictness parameters
'''''''''''''''''''''

Normally when you call a member on a mock that doesn't have any steps assigned to it, it will do the least amount of work possible
and return the least it can to the caller. In other words do nothing and return `default` where needed.

This may or may not be what you want. Another approach is that you should assign steps for all members that you expect to be used;
if any of your uncofigured members are being called that's a sign that your code is behaving in ways that you didn't expect. To get
mocks that throw an exception whenever this happens you add a ``Strict=true`` parameter to the ``MocklisClass`` attribute.

However we can be more draconian than that. Some steps will forward to further steps. If you want to throw not only on a completely
unconfigured step, but also when a step is missing a step to forward to, you can add the ``VeryStrict=true`` parameter to the
``MocklisClass`` attribute.

The exception thrown is a ``MockMissingException``, and you can ask to have it thrown regardless of strictness level by adding
a ``Missing`` step. Likewise if you want to mimic the lenient behaviour, you can add a ``Dummy`` step.

For an example where the ``Strict=true`` parameter is used, see the :doc:`../getting-started/index` section.

Return by reference
'''''''''''''''''''

C# 7 supports returning values by reference. However the ``ref`` keyword is not part of the returned type, and we cannot therefore
just introduce a Mocklis `mock`  where the return type is ``ref int`` rather than ``int``. So we can cheat by pretending that the call
is not by reference, and then we wrap it in an object at the last minute to fulfil the contract specified with the ``ref`` keyword in
the interface.

Now, there are (at leaste) two reasons for returning values by reference. One is that we wish to modify the original value through the
reference, the other is to improve performance where the returned type is larger than the size of a pointer. In the latter case we
normally mark them as ``ref readonly`` to signal that you cannot make modifications through the reference. At least since C# 7.2 when
this feature was introduced.

In the former case we usually need the reference to point to something known - so that the update has an effect that is visible to
other code. In the second it doesn't matter if the reference is to a newly created wrapper object that's only there to fulfil a
contract. Therefore Mocklis' default behaviour differs between ``ref`` and ``ref readonly``; for ``ref readonly`` we cheerfully wrap
and for ``ref`` we use a `virtual method` fallback: Instead of normal mocks, we define virtual methods for all accessors of the
mocked member and instead of adding steps you need to subclass the ``MocklisClass`` class and add bespoke implementations there.
The base implementation for these methods is to always throw a ``MockMissingException`` regardless of strictness level. This is for
technical reasons, the fallback is used in other cases where it turns out to be remarkably tricky to produce code that does nothing,
returns default values and at the same time compiles...

But anyway.

You may want to use the fallback for ``ref readonly`` returns, and you might want to use the object wrapper solution for ``ref``
returns. The ``MockReturnsByRefReadonly`` parameter defaults to true (meaning to use `mock properties` and the wrapper solution),
but you can set it to false if you like, which means to use virtual methods to control the behaviour of ``ref readonly`` returns.
Likewise the ``MockReturnsByRef`` parameter defaults to false (meaning to use the fallback), but you can set it to true if you like,
changing the code created to deal with ``ref`` returns to use mocks and the wrapper solution.

In the current version the parameters will affect all members in the class and there is no means by which we can chose what members
to use what strategy for. There are no plans to change this but if you really need it please raise an issue on GitHub.

Adding steps
============

If you just need an object that implements an interface to pass to a constructor or method then you don't need to add any steps
at all - just an instance will do.

If the instance is used, but you don't really care about what it does or returns you will get away with not doing any configuration,
as long as you've created a lenient mocklis class, which is after all the default.

In other cases you'll need to add configuration via steps. To add steps you can get a lot of help from the intellisense feature of
your code editor. Given a variable that contains an instance of a `Mocklis class`, you can type that variable and a dot to get a list
of mock properties to choose from. Select one, and type another dot and it will give you a list of all the valid steps you can add at this point.

Steps can be also chained together for more advanced cases. If a step can forward on call to other steps, type that dot directly after it
(without ending the expression with a semi-colon) and intellisense will give you a list of valid steps to choose from.

Let's say that you have mocked an ``int`` property, where the first time you call it expect the value 120, the
second time you expect the value 210, and for any calls after that it should throw a ``FileNotFoundException``.
The following would do the trick:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .ReturnOnce(120)
        .ReturnOnce(210)
        .Throw(() => new FileNotFoundException());

The ``ReturnOnce`` steps can forward on calls, while the ``Throw`` step will always throw an exception and as such
cannot chain in a further step. The extension methods used to add steps to mocks are written in such a way
that you will get full intellisense and the ability to add steps to ``TotalLinesOfCode`` and ``ReturnOnce``, but
will not allow you to add anytihng to ``Throw`` (that is to say the ``Throw`` extension method returns ``void``).

Since all steps are added through extension methods on the step type interface, any steps that you create yourself will automatically
be available through intellisense.

More details can be found in the :doc:`../reference/index` section.

Work with type parameters
=========================

Roslyn, the code analysis and compilation framework that the Mocklis code generator uses, makes some things
that look simple very difficult. Fine-tuning layout of code springs to mind. It also makes some things that
seem insanely difficult almost trivial. Using type parameters is one such case.

There are two places where you can declare new type parameters, one is in the declaration of a class, struct
or interface, and the other is when defining a method.

Type parameters on interfaces and classes
-----------------------------------------

Mocklis can mock interfaces with type parameters, and indeed `Mocklis classes` can themselves be generic. You just need to make sure all
types are closed when instantiating the class.

.. sourcecode:: csharp

    public interface IValueReader<out T>
    {
        T Value { get; }
    }

    [MocklisClass]
    public class MockValueReader<T> : IValueReader<T>
    {
         // implementation removed for brevity
    }

    // usage:
    var mock = new MockValueReader<string>();
    mock.Value.Return("Hello world!");

Note that the steps remain strongly typed to the choice of type parameters; 'mock.Value.Return(15)' wouldn't have compiled.

If a ``MocklisClass`` implements more than one interface, either directly or through other interfaces, each will be given its own implementation.
A ``MocklisClass`` implementing ``IEnumerable<string>``, which in turn extends ``IEnumerable``, will have two different ``GetEnumerator`` methods; one
from each interface, and they will need to be mocked out separately.

For a more extreme example run the code generator on the following class:

.. sourcecode:: csharp

    [MocklisClass]
    public class MyDictionary<TKey> : IDictionary<TKey, string>
    {
    }

It will happily expand out all the interfaces necessary for the implementation (such as ``ICollection<KeyValuePair<TKey, string>>``,
and leave you with a `Mocklis class` you can instantiate with any key type you wish in your tests.

A corner case to be aware of is that you cannot implement interfaces that could unify for some combinations of actual
types but not others. The following is an invalid declaration, but not because IEnumerable is declaced twice That is perfectly ok. The issue
is that if T is substituted with 'int' then the interface declarations would unify, and otherwise they would remain separate. *That* is invalid.

.. sourcecode:: csharp

    public class Incompatible<T> : IEnumerable<T>, IEnumerable<int>
    {
    }

Type parameters on methods
--------------------------

For type parameters introduced on methods, Mocklis generates code with a slightly different syntax. Let's say
you have the following in your interface:

.. sourcecode:: csharp

    public interface ITypeParameters
    {
        TOut Test<TIn, TOut>(TIn input) where TOut : struct;
    }

Now Mocklis will generate a bit more code than normally:

.. sourcecode:: csharp

    [MocklisClass]
    public class TypeParameters : ITypeParameters
    {
        // The contents of this class were created by the Mocklis code-generator.
        // Any changes you make will be overwritten if the contents are re-generated.

        private readonly TypedMockProvider _test = new TypedMockProvider();

        public FuncMethodMock<TIn, TOut> Test<TIn, TOut>() where TOut : struct
        {
            var key = new[] { typeof(TIn), typeof(TOut) };
            return (FuncMethodMock<TIn, TOut>)_test.GetOrAdd(key, keyString => new FuncMethodMock<TIn, TOut>(this, "TypeParameters", "ITypeParameters", "Test" + keyString, "Test" + keyString + "()", Strictness.Lenient));
        }

        TOut ITypeParameters.Test<TIn, TOut>(TIn input) => Test<TIn, TOut>().Call(input);
    }

The difference is that the `mock property` has been replaced with a generic `mock factory method`, and this in turn requires a slightly different syntax
when adding steps; where your 'normal' tests used to look like this:

.. sourcecode:: csharp

    var t = new TypeParameters;
    t.Test.Return(15); // mock property

You'll now write:

.. sourcecode:: csharp

    var t = new TypeParameters;
    t.Test<string, int>().Func(int.Parse); // mock factory method
    t.Test<int, int>().Func(a => a*2);     // mock factory method

Your mocks are made 'per type combination', and if you're trying to use the mock with an un-mocked set of type parameters the result depends on the strictness
level of your mock. There is no
easy way to define a mock 'for all possible combinations of types', so Mocklis doesn't support this. Note however that Mocklis passed on the type constraints
to your factory method so you won't be able to add steps to an invalid type combination.

Debugging tests
===============

There are two approaches to debugging tests that have been taken into account for Mocklis. One is something akin to good old `print`-style debugging where
a print statement would log any calls to a specific piece of code. Mocklis has a specific ``Log`` step that does roughly this.

Then, given that Mocklis generates source code you can easily set breakpoints as you would in any other code, step through code and
watch variables as you normally do. The Mocklis libraries themselves are source linked debug builds, which means that you can get
Visual Studio to let you step into the Mocklis source code and set breakpoints in the Mocklis sources almost as easily as you can do
that in your own code.

To make this work you'll need to switch off `Just My Code` and enable `Source Links`. Both from the `Options\Debugging` section in Visual Studio. It's also useful to
clear `Step over properties and operators`.

.. image:: debugging-options.png

Now you can set breakpoints in your tests and use `step into` (F11) to drill into the Mocklis source code once you're in debug mode. You can set breakpoints in any source file
opened in this fashion. If you need to set a breakpoint in a file you don't have opened but you know the name of the member, you can add this with the
`New` dropdown in the Breakpoints window, or `New Breakpoint` from the Debug menu.

Refactoring tests
=================

As the number of tests in your solution grows, it becomes increasingly important to make the test code itself streamlined and easy
to work with. The classes you write with Mocklis are just normal code so most of the techniques you use for your normal development
work equally well. In some cases you use patterns in your code base that would benefit from having steps tailored for them, we'll go
over how to do this in the section on :doc:`../extensions/index`. Find a few other techniques listed below.

Sharing setup logic
-------------------

It's a simple thing, but one that is easy to overlook. Since your `Mocklis classes` are just normal classes with source code
you can write methods that operate on them. If you have a similar mock setup needed for a number of your tests, you can
refactor that logic into a method of its own, or define extension methods on the `Mocklis class`.

Inheritance
-----------

The Mocklis code generator will not impose a base class for your `Mocklis classes`, nor will it prevent you from inheriting from them.

The only real restriction is that the `Mocklis classes` must not be partial (as that introduces a whole new level of corner
case cacaphony), or static (as you cannot implement an interface 'statically' on a class).

But in short the class hierarchy is yours for making the most of; if you want to create a common ancestor for all your mocks you can
certainly do so, and if you want to override a `Mocklis class` to centralise configuration or add new functionality just go ahead.
Mocklis will create constructors as necessary, all of which will be protected if the `Mocklis class` is abstract and public otherwise.

You can also have `Mocklis classes` inherit from other `Mocklis classes` which lets you mock new interfaces for an existing `Mocklis class`.
This could be useful if some of your tests require the mocked out dependency to also be disposable for instance...
If you do use the ``MocklisClass`` attribute at more than one level of the class hierarchy you need to generate the code in the
right order, from base class to derived class, otherwise you could get unresolved name clashes.

Invoking Mocks directly
-----------------------

Strictly speaking not something that helps you refactor tests, but still a technique that is useful to know when writing code that
interacts with `Mocklis classes`: The `mock properties` that are added to your `Mocklis classes` will let you make the same calls to them
as the explicitly implemented interface members would.

The different `MethodMock` classes (`ActionMethodMock` and `FuncMethodMock`) expose a `Call` method. The `PropertyMock`
gives you access to a `Value` property, and the `IndexerMock` has an indexer defined so you can use it directly as an indexer.

*It would be nice if the `EventMock` could have an event, but it seems it is not possible to declare an interface with a type
from a type variable, regardless of whether it's restricted to a `Delegate` type. However we have an `Add` and a `Remove` method
that will let you do the same thing.*

This can be particularly useful when unit testing steps themselves, but it can come in handy for writing normal tests as well.

.. sourcecode:: csharp

    [Fact]
    public void SetThroughMock()
    {
        var mock = new MockSample();
        var stored = mock.TotalLinesOfCode.Stored(0);

        // Write through the mock property
        mock.TotalLinesOfCode.Value = 99;

        // Assert through the stored step
        Assert.Equal(99, stored.Value);
    }

What Mocklis can't do
=====================

As with any framework, there have been trade-offs in the design.

Let's start with the biggest one: Mocklis deals with interfaces only, the reason being that only interface members can be
explicitly implemented. This makes things quite a bit easier for us - we don't need to worry too much
about naming clashes (that is to say the code generator does worry greatly about this, but the resulting
code will be much less likely to have them). Then it may be that we want to use the same mocked class
for more than one interface, and have the mock handle identical members on different interfaces in
different ways.

So if you want to mock members of an abstract base class you can't - unless you're happy to manually
write code to create `mock properties` and call them from your overridden memebers, and either do away
with the ability to call 'base' or pass on the base call as another property as a lambda.

Then there are the so-called restricted types, comprised of a handful of core .net classes
and ref structs. (The handful of classes are ``System.RuntimeArgumentHandle``, ``System.ArgIterator``,
and ``System.TypedReference``, and your ref structs are things like ``Span<T>``.) These cannot be cast
to object, and cannot be used as type parameters. As Mocklis uses type parameters to fit interface
members into one of the four standard forms, these types can not be used by normal Mocklis mocks.

Mocklis will still implement these interface members explicitly, but instead of forwarding calls
on to a `mock property` (or `mock factory method`) it will create `virtual methods` whose
default implementation is to throw a ``MockMissingException``. If you want to create bespoke behaviour you'll
have to subclass, and override. This is exactly the same trick as is used by default for some of the
`ref returns` cases mentioned earlier.

Having said all of this, Mocklis should be able to provide something that compiles from any interface or
(valid combination of) interfaces. In most cases this should should result in `mock properties` that you can use
steps with. It should also avoid any name clashes, be it clashes with the name of the `Mocklis class` itself,
any members defined in base classes, or clashes in type parameter names. If you do come up with a way of tripping
up the code generator, please flag this on GitHub so it can be dealt with.
