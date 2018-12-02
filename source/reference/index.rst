=========
Reference
=========

Steps
=====

Steps are the building blocks of mock behaviours, and they can be chained together for more advanced cases.
As such, steps are either `final` meaning they will handle any calls passed on to them, or `medial` meaning
they may pass on calls to other steps.

Let's say that you have mocked an int property, where the first time you call it expect the value 120, the
second time you expect the value 210, and for any calls after that it should throw a FileNotFoundException.
The following would do the trick:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .ReturnOnce(120)
        .ReturnOnce(210)
        .Throw(() => new FileNotFoundException());

Here, the ReturnOnce steps are `medial`, and the throw step is `final`.


Conditional steps
-----------------

The conditional steps are steps that either branch out or cut short the invocation of a mocked member
based on some condition. 

The `If` steps branch to a different chain of steps if a given condition holds, with the option of
joining the original remaining steps. In their basic form the decision is based on just information passed
to the step, parameters to a method or key to an indexer. The `InstanceIf` step lets you make the decision
based on the state of the whole mocked instance, and the `IfAdd`, `IfRemove`, `IfGet` and `IfSet` versions branch
only for those actions of an event, property or indexer.

The other set of conditional steps are the `OnlySetIfChanged` steps which only exist for properties and
indexers. When called for a `Set` action they will first `Get` the current value, and only progress the
`Set` if that value differs from the current one.

Here's a mock setup for an indexer. Note that the underlying name is used for the indexer, since it would
not be well-formed c# to name a property `this[]`. The underlying name is usually `Item` - which is the reason
why it's not possible (unless the indexer name has been changed via the ``IndexerNameAttribute``) to have
a method named `Item` in a class with an indexer.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.Item
        .If(g => g % 2 == 0, null, b => b.Return("Even"))
        .Return("Odd");

'If' steps provide you with a `joinpoint` representing the non-if branch (called 'ElseBranch'). In the following
sample we have a property, where both the getter and setter are connected to the same `Stored` step. Only calls
to the setter are logged to the console, however.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .IfSet(b => b.Log().Join(b.ElseBranch))
        .Stored(200);

The last sample for a conditional step uses the `OnlySetIfChanged` conditional. When an attempt to `set` a value,
it will first try to `get` the value, check if it's actually changed, and only forward on the `get` if it has. The
combination in the sample of the `OnlySetIfChanged`, `RaisePropertyChangedEvent` and `Stored` is so common that there
is a shorthand: `StoredWithChangeNotification`.

.. sourcecode:: csharp

    var mock = new MockSampleWithNotifyPropertyChanged();
    mock.PropertyChanged
        .Stored(out var pch);
    mock.TotalLinesOfCode
        .OnlySetIfChanged()
        .RaisePropertyChangedEvent(pch)
        .Stored();


Dummy steps
-----------

The `Dummy` steps will do as little as possible without throwing an exception. For a property or indexer, the
step will do nothing for a setter, and return a default value for a getter. For an event, adding and removing
an event handler will both do nothing. For a method, it will not do anything with parameters, and return
default values for anything that needs returning, including out and ref parameters.

Note also that `Dummy` steps are `final` - you cannot add anything to follow them.

Gate steps
----------

There is somthing slightly unfortunate with this alphabetic ordering, in that the most complex and most experimental
steps (`Gate`, `Join`, and to some extent `If`) appear before the more mundane ones (`Return` and `Stored` spring
to mind).

The idea behind a `Gate` is that it will complete a Task (as in the TPL), when the step is called. The Task can then be used to
drive other things happening in the step, effectively forcing a strict ordering.

*Syntax very experimental - only exists for `Method` mocks currently - might be killed off altogether...*

.. sourcecode:: csharp

    public async Task SuccessfulPing()
    {
        // Arrange
        var mockConnection = new MockConnection();
        mockConnection.Send
            .Gate(out var sendGate)
            .Return(Task.CompletedTask);
        mockConnection.Receive
            .Stored<MessageEventArgs>(out var messageReceive);
        var pingService = new PingService(mockConnection);

        // Act
        var ping = pingService.Ping();
        await sendGate.GatePassed;
        messageReceive.Raise(mockConnection, new MessageEventArgs(new Message("PingResponse")));
        var pingResult = await ping;

        // Assert
        Assert.True(pingResult);
    }

*Yes - kind of screams 'design phase not completed to our satisfaction', doesn't it?*

Join steps
----------

We've already met the `Join` step in the sample code for `If` above, where it allows us to take any step (with
the right form - that is member type and type parameters) and use as the next step. The missing piece is a method
to designate a step as such a target, which is where the `JoinPoint` comes in.

Let's say that we want to connect two properties to the same stored step.

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.JoinPoint(out var soup).Stored();
    mockDishes.Revenge.Join(soup);

    IDishes dishes = mockDishes;

    dishes.Vichyssoise = "Best served cold";
    Console.WriteLine(dishes.Revenge);

Note that any step would do for a `Join`, as long as we can get hold of it. The following would work equally well, taking
the `stored` step and using that as a join point.:

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.JoinPoint.Stored(out var soup);
    mockDishes.Revenge.Join(soup);

Lambda steps
------------

These steps are costructed with either an Action or a Func, and when they are called the Action or Func will be
run, and the result (in the case of the Func) will be returned.

In the current version of the code they only exist for methods, and for property and indexer getters, where in the
latter case the indexer key is passed to the func as a parameter.

The lambda steps (and some of the other steps) have 'instance' versions where the current instance of the mock
is passed as an additional parameter. This parameter is always untyped (well, passed as object), so you'll need
to cast it to one of the mocked interfaces (or the mocking class itself) for it to be of any use.

Here's an example where a `Send` method takes a message of some reference type and returns a Task:

.. sourcecode:: csharp

    var mockConnection = new MockConnection();
    mockConnection.Send.Func(m => m == null
        ? Task.FromException(new ArgumentNullException())
        : Task.CompletedTask);

Log steps
---------

`Log` steps are essentially your quintessential debugging step. They won't do anything except write out anything that
passes through them to the console (or any other TextWriter) in some detail.

Therefore you can just add in a `.Log()` if you need to figure out what happens with a given mock. Note that they are best
added early in a mock step chain if you want to get a faithful representation of what's being called from the code you
are testing, as steps can short-circuit calls or make calls of their own down the chain.

See `Conditional steps` above for an example.

Miscellaneous steps
-------------------

Stuff that couldn't really be placed in an existing category, and would have constituted a 'one-step-only' category if
pushed...

Currently this (possibly expanding) category contains just the `RaisePropertyChangedEvent` step you saw in the last example
of the `Conditional` steps category.

Missing steps
-------------

When one of these steps is invoked, it will throw a `MockMissingException` with information about the mock property itself.

Part of the contract for writing steps that can chain on to further steps, is that if no other step has been added, we should
proceed as if a `Missing` step was chained instead. You can happily think of `Missing` steps as the 'null object' for
steps.

The exception thrown could look something like this:

    *Mocklis.Core.MockMissingException: No mock implementation found for getting value of Property 'ISample.TotalLinesOfCode'. Add one using 'TotalLinesOfCode' on the 'MockSample' class.*

You won't normally need to add these yourself to your code, as they are in essence default values, but if you ever need to
the syntax is simply:

.. sourcecode:: csharp

    mockSample.DoStuff.Missing();

Record steps
------------

These steps will keep track of all the calls that have been made to them, so that you can assert in your tests that the
right interactions have happened.

Each of the record versions will cater for one type of interaction only (method call, indexer get, indexer set, property
get, property set, event add or event remove), and it will take a Func from the inforamtion passed to or returned from
these calls to something that you want to store. They also provide the ledger with recorded data as an out parameter.

There is currently no mechanism for letting record steps share ledgers with one another.

.. sourcecode:: csharp

    [Fact]
    public void RecordAddedEventHandlers()
    {
        // Arrange
        var mockSamples = new MockSampleWithNotifyPropertyChanged();
        mockSamples.PropertyChanged.RecordBeforeAdd(out var handlingTypes, h => h.Target?.GetType()).Dummy();

        // Act
        ((INotifyPropertyChanged)mockSamples).PropertyChanged += OnPropertyChanged;

        // Assert
        Assert.Equal(new[] { typeof(RecordSamples) }, handlingTypes);
    }

Repetition steps
----------------

The `Times` step look a little like a conditional step in that it adds a separate step chain that can be taken. They
differ from the if-step in that they cannot join back to the normal path, and that the separate path will only be used
a given number of times.

In the current version a get or a set both count as a usage from the same pool for property and indexer mocks, as do
adds and removes for an event mock.

For a sample see the next section, return steps.

Return steps
------------

Arguably the most important step of them all. The return step, only useable in cases where some sort of return value is
expected, will return a value.

There are three versions, one that just returns a given value once, and passes calls on to subsequent steps on later calls,
one that returns items from a list one by one, and one that returns the same value over and over.

Here's code that shows how to use these, and the repetition step:

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.GuessTheSequence
        .Times(2, m => m.Return(1))
        .ReturnOnce(int.MaxValue) // should really be infinity for this sequence
        .ReturnEach(5, 6)
        .Return(3);

    var systemUnderTest = (ISample)mock;

    Assert.Equal(1, systemUnderTest.GuessTheSequence);
    Assert.Equal(1, systemUnderTest.GuessTheSequence);
    Assert.Equal(int.MaxValue, systemUnderTest.GuessTheSequence);
    Assert.Equal(5, systemUnderTest.GuessTheSequence);
    Assert.Equal(6, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);
    Assert.Equal(3, systemUnderTest.GuessTheSequence);

Stored steps
------------

If the return steps are the most used steps, the `stored` steps are definitely the first runners up. These steps are defined
for properties, playing backing field to the mocked property. They are also defined for indexers, where the backing structure
is a dictionary which has the default return value for all non-set keys.

When creating a stored step you can give it an initial value, and you can use verifications to check that the stored value
has been set correctly by the components that are under test.

Stored steps are also used with events, and is currently the only way in Mocklis to actually invoke events. You can either
do this by invoking the stored handler, or if you use the generic EventHandler there is a version that actually gives you
a `Raise` method.

.. sourcecode:: csharp

    [Fact]
    public void RaiseEvent()
    {
        var mock = new MockSample();
        mock.MyEvent.Stored<EventArgs>(out var eventStep);
        bool hasBeenCalled = false;

        ISample sample = mock;
        sample.MyEvent += (s, e) => hasBeenCalled = true;
            
        eventStep.Raise(null, EventArgs.Empty);
        // equivalent: eventStep.EventHandler?.Invoke(null, EventArgs.Empty);
        Assert.True(hasBeenCalled);
    }


Throw steps
-----------

Super easy - with these steps you provide a factory method that creates an exception. When called, the step will call
this method and throw the exception it returns.


Verification steps
------------------

Verification steps are steps that track some condition that can be checked and asserted against. The only verification steps
currently check that interface members have been called the right number of times.

These steps take a verification group as a parameter, along with the number of time they expect the mocked member to be called,
which are tracked individually for getters, setters, adds and removes (and plain method calls).

To get access to all `steps` and `checks` (see next section) for verifications you need to have the namespace `Mocklis.Verification`
in scope via a using statement at the top of your file.



Verifications
=============

If steps provide a means of creating behaviour for the system under test, verifications provide a means of checking that those
behaviours have been used in the right way by the system under test.

Verifications come in two flavours. As normal steps they check data as it passes through them:

.. sourcecode:: csharp

    var vg = new VerificationGroup();
    var mock = new MockSample();
    mock.DoStuff
        .ExpectedUsage(vg, "DoStuff",  1)
        .Dummy();

There are also verifications that check some condition of an existing step (unimaginatively, just called 'checks'):

.. sourcecode:: csharp

    [Fact]
    public void JustChecks()
    {
        var vg = new VerificationGroup();
        var mock = new MockSample();
        mock.TotalLinesOfCode
            .Stored(50)
            .CurrentValueCheck(vg, "TLC", 60);

        ISample sample = mock;
        sample.TotalLinesOfCode = 60;

        vg.Assert();
    }

These are the only verifications in the framework at the moment. The expected usage steps work for all different member types,
and track the different access methods independently. The current value checks exist for properties and indexers only, where
the latter takes a list of key-value pairs to check.

To check that verifications have been met, call `Assert` on the top-most verification group, as done in the last example.

