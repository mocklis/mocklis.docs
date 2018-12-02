===============
Getting Started
===============


My first mock
=============

Normally the use of mocking is to create fake but controllable replacements for dependencies to the
code that we wish to test, but for a simple walk-through of the functionality we're probably better
off with a normal console application.

Let's say we're writing a component that reads numbers from standard input, sends them off to a web
service for some calculation and writes the result to standard output.

The first step is to create two interfaces. One for reading/writing and one for the 'service':

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

And then we have our code that uses these two interfaces. Let's add a constructor to our Program class.

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

Add two new classes, MockConsole and MockService. Let them implement their corresponding interface, reference Mocklis, and 
add the MocklisClass attribute to both classes.

.. sourcecode:: csharp

    [MocklisClass]
    public class MockConsole : IConsole
    {
    }

    [MocklisClass]
    public class MockService : IService
    {
    }

Now you can use the 'Update Mocklis Class' refactoring action to create implementations for these interfaces. Right-click on MockConsole
and chose this refactoring from the context menu. Then do the same for MockService.

*Note: The rest of this section relies on you having created mock implementations using Mocklis.*

Instantiate these mocks in the static Main. Pass the instances to the constructor, create a non-static Run method and call it once the program
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

Note that you didn't have to cast mockConsole to IConsole, or MockService to IService. As long as the parameters accepting the mocked
instances are of interface type, c# will perform an implicit cast.

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

If we try to run this we'll fall over with a `MockMissingException` at _console.ReadLine:

.. sourcecode:: none

    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IConsole.ReadLine'. Add one using 'ReadLine' on the 'MockConsole' class.

Let's fix this with some mocking. First we want to return some strings from the mocked console. Let's say the strings "8", "13", "21", and an empty string.
We should also add logging so we can follow what's going on. Update Main() as follows:

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        mockConsole.ReadLine.Log().ReturnEach("8", "13", "21", string.Empty);

        var program = new Program(mockConsole, mockService);
        program.Run();
    }

Running the program now should give us the following output, most of it coming from the Log call.

.. sourcecode:: none

    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 8
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 13
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 21
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: 
    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IService.Calculate'. Add one using 'Calculate' on the 'MockService' class.

Apparently we're missing a mock for the IService.Calculate interface member. Let's add that. In fact, let's just pretend that the service adds up anything that is sent to it.

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
    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IConsole.WriteLine'. Add one using 'WriteLine' on the 'MockConsole' class.

Ok - so we're still missing mocking out the WriteLine method. Let's do so, add logging (as for the other ones) and also recording. Other than recording the
call we don't care about what happens, so we're chaining in the Dummy step at the end. Currently Mocklis doesn't special-case simple collections when writing
out parameters, just as it will not write out tuple names in a value tuple. In basically does what `ToString()` does...

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

It's a simple thing, but one that is easy to overlook. Since your mock classes are available as source code,
you can write methods that operate on them. So if you have a similar mock setup needed for a number of your
tests, you can refactor that logic into a method of its own.

Inheritance
-----------

The Mocklis code generator will not impose a base class for your classes, nor will it enforce that your
classes are sealed. That is to say you can do it yourself if you want to but this is not a requirement.

(The only real restriction is that the mocklis classes must not be partial - as that introduces a whole
new level of corner case cacaphony.)

If you do derive the mocklis class from one of your own classes, it must have a default constructor, as the
generated code will only create one of those. Apart from this restriction, the class hierarchy is yours for
making the most of; if you want to create a common ancestor for all your mocks you can, and if you want to
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

