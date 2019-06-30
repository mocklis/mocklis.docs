===============
Getting Started
===============

A first mock
============

Mocking is the art of creating fake but controllable replacements for dependencies to code that we wish
to test. However, for a simple walk-through of the functionality we'll write a normal console application
instead hoping that not too much is lost in the transition.

Let's say we're writing a component that reads numbers from standard input, sends them off to a web
service for some calculation and writes the result to standard output.

The first step is to create two interfaces. One for interacting with the console, and one for the web service:

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
with fields for the dependencies we'll use.

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

Add two new classes, ``MockConsole`` and ``MockService``. Let them implement their corresponding interface but should otherwise
stay empty. Reference the ``Mocklis`` NuGet package from your project and add the ``MocklisClass`` attribute to both classes.

For this walkthrough we need to add the 'Strict' parameter to the attributes - this is purely because we want to get exceptions
for missing configurations to guide us to write more mocks. In your real life cases you may wish to have all mocks return
default values instead of throwing exceptions in which case leave out the `Strict` parameter.

.. sourcecode:: csharp

    [MocklisClass(Strict = true)]
    public class MockConsole : IConsole
    {
    }

    [MocklisClass(Strict = true)]
    public class MockService : IService
    {
    }

Now you can use the `Update Mocklis Class` code fix to create implementations for these interfaces. If you look at the MocklisClass
attribute you'll see that the first characters have an underline. This is Visual Studio hinting that there is a code fix available.
Move your mouse over that area, and Visual Studio will provide you with a light-bulb. Click on the dropdown arrow and you'll see a list of
suggestions. Pick 'Update Mocklis Class' from the list, which will create an implementation of the class. Do this for both classes.

*If you find it a bit tricky to find the `hint` handle, you can change the 'Serevity' of the rule in your project as follows: In the
solution explorer, expand References, Analyzers and Mocklis.MockGenerator. Right-click on the MocklisAnalyzer, and choose `Set Rule Severity`.
Choose `Warning` and your entire MocklisClass will be underlined with a green squiggly line and you can update the class from
anywhere.*

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

Note that you didn't have to cast ``mockConsole`` to ``IConsole``, or ``mockService`` to ``IService``. As long as the parameters accepting the mocked
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

If we try to run this we'll fall over with a ``MockMissingException``:

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
    Returned from '[MockConsole] IConsole.ReadLine' with result: '8'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '13'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '21'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: ''
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
    Returned from '[MockConsole] IConsole.ReadLine' with result: '8'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '13'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '21'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: ''
    Calling '[MockService] IService.Calculate' with parameter: 'System.Int32[]'
    Returned from '[MockService] IService.Calculate' with result: '42'
    Mocklis.Core.MockMissingException: No mock implementation found for Method 'IConsole.WriteLine'. Add one using 'WriteLine' on your 'MockConsole' instance.

Ok - so we're still missing mocking out the ``WriteLine`` method. Let's do so, add logging (as for the other ones) and also recording. A ``Record`` step will
remember everything that was passed through it, and make it available using its out parameter as an IReadOnlyList.

``Record`` steps, like ``Log`` steps, kind of expect you to chain further steps. They don't make any decisions on their own. In this case we didn't provide a further step
so a default behaviour kicks in which is to accept any input and provide default values for any output required. This is normally the default behaviour as
well for mocks that don't have any steps defined at all, but we overrode that behaviour with the `Strict = true` switch in the ``MocklisClass`` attribute. If
we want to throw for an incomplete mock as well (such as only providing a ``Log`` or ``Record`` without a subsequent step) you can set the `VeryStrict = true`
attribute switch. If you'd done so, you would have needed to add a ``Dummy`` step after the ``RecordBeforeCall`` step.

Let's also write out the first recorded value (in fact the only recorded value) to the real console so we can see the full thing end-to-end.

.. sourcecode:: csharp

    public static void Main()
    {
        var mockConsole = new MockConsole();
        var mockService = new MockService();

        mockConsole.ReadLine.Log().ReturnEach("8", "13", "21", string.Empty);
        mockConsole.WriteLine.Log().RecordBeforeCall(out var consoleOut);
        mockService.Calculate.Log().Func(m => m.Sum());

        var program = new Program(mockConsole, mockService);
        program.Run();

        Console.WriteLine("The value 'written' to console was " + consoleOut[0]);
    }

The parameter to ``RecordBeforeCall`` returns a list with the recorded values, which by default is just a list of the values passed to the method. You may want to
store a subset of these or do some calculation on some values (or if they are mutable, get the current values before they're changed) in which case you can add
a selector func as a second parameter.

The program now completes without any exceptions, with the following output:

.. sourcecode:: none

    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '8'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '13'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: '21'
    Calling '[MockConsole] IConsole.ReadLine'
    Returned from '[MockConsole] IConsole.ReadLine' with result: ''
    Calling '[MockService] IService.Calculate' with parameter: 'System.Int32[]'
    Returned from '[MockService] IService.Calculate' with result: '42'
    Calling '[MockConsole] IConsole.WriteLine' with parameter: '42'
    Returned from '[MockConsole] IConsole.WriteLine'
    The value 'written' to console was 42

And with that we have written our first program with mocked interfaces using Mocklis. Of course normally we don't work
with mocking outside of unit tests, so this was for illustration only. But it should have given you some idea of what
you can use Mocklis for.
