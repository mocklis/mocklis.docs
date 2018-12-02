===============
Getting Started
===============


My first mock
=============


Normally the use of mocking is to create fake but controllable replacements for dependencies to the
code that we wish to test, but for a simple walk-through of the functionality we're probably better
off with a normal console application, and just work with the mock instance itself.

So - create a new console application, and add an interface to it.





Common use-cases
================

Sharing setup logic
-------------------

It's a simple thing, but one that is easy to overlook. Since your mock classes are available as source code,
you can write methods that operate on them. So if you have a similar mock setup needed for a number of your
tests, you can refactor that logic into a method of its own.

Inheritance
-----------

The Mocklis code generator will not impose a base class for your classes, nor will it enforce that your
classes are sealed. That is to say you can do it yourself if you want to but there is no forcing.

(The only real restriction is that the mocklis classes must not be partial - as that introduces a whole
new level of corner case cacaphonies.)

If you do derive the mocklis class from one of your own classes, it must have a default constructor, as the
generated code will only create one of those. Apart from this restriction, the class hierarchy is yours for
making the most of. If you want to create a common ancestor for all your mocks you can, and if you want to
override a mocklis class (to create common behaviour or make individual steps available through new properties)
please go ahead.

If you use the MocklisClass attribute at more than one level of the class hierarchy you need to generate the
code in the right order, from base class to derived class. Since the commmand line implementation of Mocklis
currently doesn't know how to figure this out for itself, you may need to run it more than once to get all
the classes correctly generated.

Type Parameters
---------------

Roslyn, the code analysis and compilation framework that the Mocklis code generator uses, makes some things
that look simple very difficult. Fine-tuning layout of code springs to mind. It also makes some things that
seem insanely difficult almost trivial. Using type parameters is one such case.

Mocklis will very happily let you declare mock classes with open type parameters, or with some open and some
closed, in any (valid) combination. And Roslyn somehow sorts it out. Try for instance this:

.. sourcecode:: csharp
    
    [MocklisClass]
    public class Blah<TBlah> : IDictionary<TBlah, string>
    {
    }

It will happily expand out all the interfaces necessary for the implementation (such as `ICollection<KeyValuePair<TBlah, string>>`,
and leave you with a mock class you can fully close with different key types for your tests.

*Now there's one mock class you didn't want to write by hand...*

Invoking Mocks
--------------

The mock properties that are added to your mocklis classes will let you make the same calls to them
as the mocked-out interface members would.

The different `MethodMock` classes (`ActionMethodMock` and `FuncMethodMock`) expose a `Call` method. The `PropertyMock`
gives you access to a `Value` property, and the `IndexerMock` has an indexer defined so you can use it directly as an indexer.

It would be nice if the `EventMock` could have an event, but it seems it is not possible to declare an interface with a type
from a type variable, regardless of whether it's restricted to a `Delegate` type. However we have an `Add` and a `Remove` method
that will let you do the same thing.

This can be particularly useful when unit testing steps themselves.


What Mocklis can't do
=====================

As with any framework, there have been trade-offs in the design process. Therefore there are a number
of things that just cannot be done with the framework, and there are a number of things that are not
yet possible to do with the framework.

Firstly: Mocklis deals with interfaces only, the reason being that mocked interface members can be 
explicitly implemented. This makes things quite a bit easier for us - we don't need to worry too much
about naming clashes (that is to say the code generator worries greatly about exactly this, but the resulting
code will be much less likely to have them). Then it may be that we want to use the same mocked class
for more than one interface, and have the mock handle identical members on different interfaces in
different ways.

So if you want to mock members of an abstract base class you can't - unless you're happy to manually
write code to create mock properties and call them from your overridden memebers, and either do away
with the ability to call 'base' or pass on the base call as another property as a lambda.


Then there are methods whose parameters or return values we cannot turn into a TParam/TResult pair of
Value tuples, and indeed there are interface members that cannot be explicitly implemented at all.

The full known list of 'problematic' members are shown by the following interface declaration. Mocklis will
not try to implement them, insted leaving you with a comment in the class of what it couldn't do. It
is not particularly satisfying, but ways of dealing with them are being investigated.

.. sourcecode:: csharp

    public interface IProblematic
    {
        ref int MyProperty { get; }
        ref readonly int MyReadonlyProperty { get; }

        ref string this[int i] { get; }
        ref readonly int this[string i] { get; }

        ref int RefMethod();
        ref readonly int RefReadonlyMethod();

        string GenericMethod<T>(T data);

        bool ArgListMethod(__arglist);
    }

... which will just leave you with the following - non compiling - mock class:

.. sourcecode:: csharp

    [MocklisClass]
    public class Problematic : IProblematic
    {
        // Could not create mocks for the following members:
        // * IProblematic.MyProperty (returns by reference)
        // * IProblematic.MyReadonlyProperty (returns by readonly reference)
        // * IProblematic.this[] (returns by reference)
        // * IProblematic.this[] (returns by readonly reference)
        // * IProblematic.RefMethod (returns by reference)
        // * IProblematic.RefReadonlyMethod (returns by readonly reference)
        // * IProblematic.GenericMethod (introduces new type parameter)
        // * IProblematic.ArgListMethod (uses __arglist)
        //
        // Future version of Mocklis will handle these by introducing virtual members
        // that can be given a 'mock' implementation in a derived class.

        public Problematic()
        {
        }
    }

If you come up with other ways of foiling the code generator, please flag this up so it can be
dealt with.

