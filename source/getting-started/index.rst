===============
Getting Started
===============


My first mock
=============

Mocking is the art of creating fake but controllable replacements for dependencies to code that we wish
to test. However, for a simple walk-through of the functionality we'll write a normal console application
instead hoping that not too much is lost in the transition.

Let's say we're writing a component that reads numbers from standard input, sends them off to a web
service for some calculation and writes the result to standard output.

The first step is to create two interfaces. One for reading/writing and one for the web service:

.. sourcecode:: csharp

    public interface IConsole
    {
        string ReadLine();
        void WriteLine(string s);
    }


    public interface IService
    {
        int Calculate(params int[] values);
    }

And then we have our code that uses these two interfaces. Let's add a constructor to our ``Program`` class, along
with fields for the services we'll use.

.. sourcecode:: csharp

    public class Program
    {
        public static void Main()
        {
        }

        private readonly IConsole _console;
        private readonly IService _service;

        public Program(IConsole console, IService service)
        {
            _console = console;
            _service = service;
        }
    }

We will at some point write proper implementations of these interfaces, but for now we want to just mock them out.

Add two new classes, ``MockConsole`` and ``MockService``. Let them implement their corresponding interface, reference ``Mocklis.Analyzer``
and ``Mocklis`` (which will in turn bring in ``Mocklis.Core``), and add the ``MocklisClass`` attribute to both classes.

.. sourcecode:: csharp

    [MocklisClass]
    public class MockConsole : IConsole
    {
    }

    [MocklisClass]
    public class MockService : IService
    {
    }

Now you can use the 'Update Mocklis Class' code fix to create implementations for these interfaces. Right-click on ``MockConsole``
and chose the fix from the context menu. Then do the same for ``MockService``. Your program should now compile.

*Note: The rest of this section relies on you having created mock implementations using Mocklis.*

Instantiate these mocks in the static ``Main``. Pass the instances to the constructor, create a non-static ``Run`` method and call it once the program
instance has been created:

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        var program = new Program(mockConsole, mockService);
        program.Run();
    }

    public void Run()
    {
    }

Note that you didn't have to cast ``mockConsole`` to ``IConsole``, or ``MockService`` to ``IService``. As long as the parameters accepting the mocked
instances are of an implemented interface type, C# will perform an implicit cast.

Now we want to have a play with the interfaces. Let's say we read numbers off standard input until we get an empty string, pass them
all to the service, and then write the return value back to the console.

.. sourcecode:: csharp

    public void Run()
    {
        var values = new List<int>();
        for (;;)
        {
            string s = _console.ReadLine();
            if (string.IsNullOrEmpty(s))
            {
                break;
            }
            values.Add(int.Parse(s));
        }

        var result = _service.Calculate(values.ToArray());
        _console.WriteLine(result.ToString());
    }

If we try to run this we'll fall over with a ``MockMissingException`` at ``_console.ReadLine``:

.. sourcecode:: none

    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IConsole.ReadLine'. Add one using 'ReadLine' on your 'MockConsole' instance.

Let's fix this with some mocking. First we want to return some strings from the mocked console. Let's say the strings "8", "13", "21", and an empty string.
We should also add logging so we can follow what's going on. Update ``Main`` as follows:

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        mockConsole.ReadLine.Log().ReturnEach("8", "13", "21", string.Empty);

        var program = new Program(mockConsole, mockService);
        program.Run();
    }

Running the program now should give us the following output, most of it coming from the ``Log`` step.

.. sourcecode:: none

    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 8
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 13
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 21
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result:
    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IService.Calculate'. Add one using 'Calculate' on your 'MockService' instance.

Apparently we're missing a mock for the ``IService.Calculate`` interface member. Let's add that. In fact, let's just pretend that the service adds up anything that is sent to it.

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        mockConsole.ReadLine.Log().ReturnEach("8", "13", "21", string.Empty);
        mockService.Calculate.Log().Func(m => m.Sum());

        var program = new Program(mockConsole, mockService);
        program.Run();
    }

Which should now give us the following when we run the program:

.. sourcecode:: none

    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 8
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 13
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 21
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result:
    Calling '[MockService] IService.Calculate' with parameter: System.Int32[]
    Returned from '[MockService] IService.Calculate' with result: 42
    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IConsole.WriteLine'. Add one using 'WriteLine' on your 'MockConsole' instance.

Ok - so we're still missing mocking out the ``WriteLine`` method. Let's do so, add logging (as for the other ones) and also recording. Other than recording the
call we don't care about what happens, so we're chaining in a ``Dummy`` step at the end. Currently Mocklis doesn't special-case simple collections when writing
out parameters, just as it will not write out tuple names in a value tuple. It basically does what ``ToString()`` does...

Let's also write out the first recorded value (in fact the only recorded value) to the real console so we can see the full thing end-to-end.

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        mockConsole.ReadLine.Log().ReturnEach("8", "13", "21", string.Empty);
        mockConsole.WriteLine.Log().RecordBeforeCall(out var consoleOut, a => a).Dummy();
        mockService.Calculate.Log().Func(m => m.Sum());

        var program = new Program(mockConsole, mockService);
        program.Run();

        Console.WriteLine("The value 'written' to console was " + consoleOut[0]);
    }

The first parameter to ``RecordBeforeCall`` returns a list with the recorded values, and the second is a selector lambda. This is used because you may not want to record
all of the data passed around, and furthermore if any of the parameters is mutable you may want to capture the current state at the time of recording. In
this particular case we want to keep the whole thing, hence ``a => a``.

The program now completes without any exceptions, with the following output:

.. sourcecode:: none

    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 8
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 13
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 21
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result:
    Calling '[MockService] IService.Calculate' with parameter: System.Int32[]
    Returned from '[MockService] IService.Calculate' with result: 42
    Calling '[MockConsole] IConsole.WriteLine' with parameter: 42
    Returned from '[MockConsole] IConsole.WriteLine'
    The value 'written' to console was 42

And with that we have written our first program with mocked interfaces using Mocklis. Of course normally we don't work
with mocking outside of unit tests, so this was for illustration only. But it should have given you some idea of what
you can use Mocklis for.

Common use-cases
================

Apart from the very basic mocking out of individual members we saw in the 'my first mock' above, there are
some tricks of the trade that can be very useful. Find below a couple of our favourites:

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
certainly do so, and if you want to override a `Mocklis class`
(to create common behaviour or make individual steps available through new properties) please go ahead. Mocklis will
create constructors as necessary, all of which will be protected if the `Mocklis class` is abstract and public otherwise.

You can also have `Mocklis classes` inherit from other `Mocklis classes` which lets you mock new interfaces for an existing `Mocklis class`.
This could be useful if some of your tests require the mocked out dependency to also be disposable for instance...
If you do use the ``MocklisClass`` attribute at more than one level of the class hierarchy you need to generate the code in the
right order, from base class to derived class, otherwise you could get unresolved name clashes.

Type Parameters
---------------

Roslyn, the code analysis and compilation framework that the Mocklis code generator uses, makes some things
that look simple very difficult. Fine-tuning layout of code springs to mind. It also makes some things that
seem insanely difficult almost trivial. Using type parameters is one such case.

Mocklis will very happily let you declare `Mock classes` with open type parameters, or with some open and some
closed, in any (valid) combination. And Roslyn somehow sorts it out. Try for instance this:

.. sourcecode:: csharp

    [MocklisClass]
    public class Blah<TBlah> : IDictionary<TBlah, string>
    {
    }

It will happily expand out all the interfaces necessary for the implementation (such as ``ICollection<KeyValuePair<TBlah, string>>``,
and leave you with a `Mocklis class` you can instantiate with proper types in your tests.

*Now there's one mock class you didn't want to write by hand...*

Mocklis will also allow member methods that introduce new type parameters, but they require a slightly different syntax. Let's say
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
        private readonly TypedMockProvider _test = new TypedMockProvider();

        public FuncMethodMock<TIn, TOut> Test<TIn, TOut>() where TOut : struct
        {
            var key = new[] { typeof(TIn), typeof(TOut) };
            return (FuncMethodMock<TIn, TOut>)_test.GetOrAdd(key, keyString => new FuncMethodMock<TIn, TOut>(this, "TypeParameters", "ITypeParameters", "Test" + keyString, "Test" + keyString + "()"));
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

Your mocks are made 'per type combination', and if you're trying to use the mock with an un-mocked set of type parameters you'll get a ``MockMissingException``. There is no
easy way to define a mock 'for all possible combinations of types', so Mocklis doesn't support this. Note however that Mocklis passed on the type constraints
to your factory method so you won't be able to add steps to an invalid type combination.

Invoking Mocks
--------------

The `mock properties` that are added to your `Mocklis classes` will let you make the same calls to them
as the explicitly implemented interface members would.

The different `MethodMock` classes (`ActionMethodMock` and `FuncMethodMock`) expose a `Call` method. The `PropertyMock`
gives you access to a `Value` property, and the `IndexerMock` has an indexer defined so you can use it directly as an indexer.

It would be nice if the `EventMock` could have an event, but it seems it is not possible to declare an interface with a type
from a type variable, regardless of whether it's restricted to a `Delegate` type. However we have an `Add` and a `Remove` method
that will let you do the same thing.

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

Firstly: Mocklis deals with interfaces only, the reason being that only interface members can be
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
on to a `mock property` (or `mock factory method`) it will create a `virtual method` whose
default implementation is to throw a ``MockMissingException``. If you want to create bespoke behaviour you'll
have to subclass, and override.

Mocklis uses this trick for another set of interface members, namely those returning values by ref. While
these can be fit into the four standard forms by wrapping the return value into a reference and returning
that, the default behaviour for Mocklis is to create `virtual methods` for these members. The reasoning
is that returning by ref is really useful when the returned value is something that we want to
observe the change in - otherwise you would surely have used ref readonly instead. For instance the following method
gives us a reference to one of the entries in the array which can be used to change the value of that
particular array entry:

.. sourcecode:: csharp

    public ref double GetAtIndex(double[] array, int index)
    {
        return ref array[index];
    }

Since Mocklis' normal approach would be to wrap the resulting value in a new ref just before returning it, we would
not be able to add behaviour that mimicks this.

On the other hand when the reference is returned 'readonly' we expect the usage of ref to simply be a performance
improvement - we won't need to be able to observe the change since there cannot be one. In this case the
default behaviour is to mock the member out as if the value was returned normally without any ref or readonly,
and then we wrap it up in a reference and return that. There will be a small performance penalty, but at least
we can use the normal steps we have in our mocking arsenal.

The choice to use `virtual methods` for return 'by ref', and `mock properties` for return 'by ref readonly' is
made without knowing exactly how Mocklis will be used. The ``MocklissClass`` attribute defines two properties
(``MockReturnsByRef`` which defaults to ``false``, and ``MockRetursByRefReadonly`` which defaults to ``true``)
that control which method is used by each of these cases. It's not currently possible to use different approaches for
different mocked-out members in the same interface.

Mocklis should be able to provide something that compiles from any interface or (valid combination of) interfaces.
In most cases this should be a `mock property`, that you can use steps with. It should also avoid
any name clashes, be it clashes with the name of the `Mocklis class` itself, any members defined in base classes,
or clashes in type parameter names. If you do come up with a way of foiling the code generator, please flag this
up so it can be dealt with.
