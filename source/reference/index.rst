=========
Reference
=========

Standard Steps
==============

Steps are the building blocks of mock behaviours, and they can be chained together for more advanced cases,
that is to say that some steps will let you add on further steps that they can optionally forward on calls to.

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
will not allow you to add anytihng to ``Throw`` (that is to say the ``Throw`` extension method returns ``void``). We
sometimes refer to these steps as 'final'.


Conditional steps
-----------------

The conditional steps are steps that either branch out or cut short the invocation of a mocked member
based on some condition.

The ``If`` steps branch to a different chain of steps if a given condition holds, with the option of
joining the original remaining steps. In their basic form the decision is based on just information passed
to the step, parameters to a method or key to an indexer. The ``InstanceIf`` step lets you make the decision
based on the state of the whole mocked instance, and the ``IfAdd``, ``IfRemove``, ``IfGet`` and ``IfSet`` versions branch
only for those actions of an event, property or indexer.

Here's a mock setup for an indexer. Note the underlying name used for the indexer, which is used since it would
not be well-formed C# to name a property `this[]`. The underlying name is usually `Item` - which is the reason
why it's not possible (unless the indexer name has been changed via the ``IndexerNameAttribute``) to have
a method named ``Item`` in a class with an indexer.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.Item
        .If(g => g % 2 == 0, null, b => b.Return("Even"))
        .Return("Odd");

``If`` steps provide you with a `joinpoint` representing the non-if branch (called 'ElseBranch'). In the following
sample we have a property, where both the getter and setter are connected to the same ``Stored`` step. Only calls
to the setter are logged to the console, however.

.. sourcecode:: csharp

    var mock = new MockSample();
    mock.TotalLinesOfCode
        .IfSet(b => b.Log().Join(b.ElseBranch))
        .Stored(200);

The last sample for a conditional step uses the ``OnlySetIfChanged`` conditional which only exists for properties
and indexers. When an attempt to 'set' a value is made, the step will first try to 'get' the value, check if it's
actually changed, and only 'set' the new value if it has.

.. sourcecode:: csharp

    var mock = new MockSampleWithNotifyPropertyChanged();
    mock.PropertyChanged
        .Stored(out var pch);
    mock.TotalLinesOfCode
        .OnlySetIfChanged()
        .RaisePropertyChangedEvent(pch)
        .Stored();

The combination of the ``OnlySetIfChanged``, ``RaisePropertyChangedEvent`` and ``Stored`` steps is so common that there
is a shorthand: ``StoredWithChangeNotification``

.. sourcecode:: csharp

    var mock = new MockSampleWithNotifyPropertyChanged();
    mock.PropertyChanged
        .Stored(out var pch);
    mock.TotalLinesOfCode
        .StoredWithChangeNotification(pch);


Dummy steps
-----------

The ``Dummy`` steps will do as little as possible without throwing an exception. For a property or indexer, the
step will do nothing for a setter, and return a default value for a getter. For an event, adding or removing
an event handler do absolutely nothing, and for a method, it will not do anything with the parameters, and return
default values for anything that needs returning, including out and ref parameters.

Note also that ``Dummy`` steps are final - you cannot add anything to follow them.


Join steps
----------

We've already met the ``Join`` step in the sample code for ``If`` above, where it allows us to take any step (with
the right form - that is member type and type parameters) and use as the next step. The missing piece is a method
to designate a step as such a target, which is where the ``JoinPoint`` comes in.

Let's say that we want to connect two properties to the same ``Stored`` step. The solution is to add a ``JoinPoint``
step just before the ``Stored`` step.

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.JoinPoint(out var soup).Stored();
    mockDishes.Revenge.Join(soup);

    IDishes dishes = mockDishes;

    dishes.Vichyssoise = "Best served cold";
    Console.WriteLine(dishes.Revenge);

Note that any step would do for a ``Join``, as long as we can get hold of it. The following would work equally well, taking
the ``Stored`` step itself and using that as a join point:

.. sourcecode:: csharp

    var mockDishes = new MockDishes();
    mockDishes.Vichyssoise.JoinPoint.Stored(out var soup);
    mockDishes.Revenge.Join(soup);

Lambda steps
------------

These steps are constructed with either an ``Action`` or a ``Func``, and when they are called the ``Action`` or ``Func`` will be
run. In the case of ``Func`` the result of the call will be returned.

The names always contain the word ``Action`` or the word ``Func``, but they are further qualified for non-method steps. Property
and indexer steps are called ``GetFunc`` and ``SetAction`` while event steps are called ``AddAction`` and ``RemoveAction``.

The lambda steps (and some of the other steps) have 'instance' versions where the current instance of the mock
is passed as an additional parameter. This parameter is always untyped (well, passed as object), so you'll need
to cast it to one of the mocked interfaces (or the mocking class itself) for it to be of any use. These steps have the names of
their non-instance counterparts prefixed with the word ``Instance`` (so that ``InstanceSetAction`` would exist as a
property step to give an example).

Here's an example where a ``Send`` method takes a message of some reference type and returns a ``Task``:

.. sourcecode:: csharp

    var mockConnection = new MockConnection();
    mockConnection.Send.Func(m => m == null
        ? Task.FromException(new ArgumentNullException())
        : Task.CompletedTask);

Log steps
---------

``Log`` steps are your quintessential debugging steps. They won't do anything except write out anything that
passes through them to the console (or any other TextWriter) in some detail.

Therefore you can just add in a ``.Log()`` if you need to figure out what happens with a given mock. Note that they are best
added early in a mock step chain if you want to get a faithful representation of what's being called from the code you
are testing, as steps can short-circuit calls or make calls of their own down the chain.

See Conditional steps above for an example.

Miscellaneous steps
-------------------

Stuff that couldn't really be placed in an existing category, and would have constituted a 'one-step-only' category if
pushed...

Currently this (possibly expanding) category contains just the ``RaisePropertyChangedEvent`` step you saw in the last example
of the Conditional steps category.

Missing steps
-------------

When one of these steps is invoked, it will throw a ``MockMissingException`` with information about the `mock property` itself.

The exception thrown could look something like this:

    *Mocklis.Core.MockMissingException: No mock implementation found for getting value of Property 'ISample.TotalLinesOfCode'. Add one using 'TotalLinesOfCode' on your 'MockSample' instance.*

Record steps
------------

These steps will keep track of all the calls that have been made to them, so that you can assert in your tests that the
right interactions have happened.

Each of the record steps will cater for one type of interaction only (method call, indexer get, indexer set, property
get, property set, event add or event remove), and it will take a ``Func`` that transforms whatever is seen by the step
to something that you want to store. They also provide the 'ledger' with recorded data as an out parameter.

There is currently no mechanism for letting record steps share these 'ledgers' with one another.

.. sourcecode:: csharp

    [Fact]
    public void RecordAddedEventHandlers()
    {
        // Arrange
        var mockSamples = new MockSampleWithNotifyPropertyChanged();
        mockSamples.PropertyChanged.RecordBeforeAdd(out var handlingTypes, h => h.Target?.GetType());

        // Act
        ((INotifyPropertyChanged)mockSamples).PropertyChanged += OnPropertyChanged;

        // Assert
        Assert.Equal(new[] { typeof(RecordSamples) }, handlingTypes);
    }

Repetition steps
----------------

The ``Times`` steps look a little like conditional steps in that they add a separate step chain that can be taken. They
differ from the if-step in that they cannot join back to the normal path, and that the separate path will only be used
a given number of times.

In the current version a get or a set both count as a usage from the same pool for property and indexer mocks, as do
adds and removes for an event mock.

For a sample see the next section, return steps.

Return steps
------------

Arguably the most important step of them all. The ``Return`` step, only useable in cases where some sort of return value is
expected, will simply return a value.

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

If the ``Return`` steps are the most used steps, the ``Stored`` steps are definitely the first runners up. These steps are defined
for properties, playing backing field to the mocked property. They are also defined for indexers, where the backing structure
is a dictionary which has the default return value for all non-set keys.

When creating a ``Stored`` step for a property you can give it an initial value, and for both properties and indexers you can use
verifications to check that the stored value has been set correctly by the components that are under test.

``Stored`` steps are also used with events where the steps act as storage for added event handlers. If you have a reference to the
``Stored`` step you can raise events on these handlers, simply by calling ``Invoke`` on the stored value. Alternatively,
if your handler type is a generic ``EventHandler<>`` or one of a handful of very common event
handler types including ``PropertyChangedEventHandler`` and the basic ``EventHandler``, the Mocklis library
provides you with ``Raise`` extension methods. These can be found in the ``Mocklis.Verification`` namespace.

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

For indexers the step is called ``StoredAsDictionary`` as it holds different values for different indexes. It will return a default
value rather than throw if an empty slot is read from.

Throw steps
-----------

Super easy - with these steps you provide a ``Func`` that creates an exception. When called, the step will call
this method and throw the exception it returns.


Verification steps
------------------

Verification steps are steps that track some condition that can be checked and asserted against.

``ExpectedUsage`` steps take a verification group as a parameter, along with the number of time they
expect the mocked member to be called (which are tracked individually for getters, setters, adds, removes and plain method calls).

To get access to all steps and checks (see next section) for verifications you need to have the namespace ``Mocklis.Verification``
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
        .ExpectedUsage(vg, "DoStuff",  1);

... and also as 'checks' that verify some condition of an existing step:

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

To check that verifications have been met, call ``Assert`` on the top-most verification group, as done in the last example.

Experimental Stuff
==================

Mocklis has a project & associated NuGet package for experimental things: ``Mocklis.Experimental``. It is meant for things that are
in a bit of flux and may either graduate to the main ``Mocklis`` package, or be found wanting and deleted.

Gate steps
----------

The idea behind the ``Gate`` step is that it will complete a ``Task`` (as in Task Parallel Library), when the step is called. The ``Task`` can then be used to
drive other things happening in the step, effectively forcing a strict ordering of events in the face of many threads running.

The syntax is still very experimental - it currently only exists for 'Method' mocks, and might well be killed off altogether...

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
        await sendGate;
        messageReceive.Raise(mockConnection, new MessageEventArgs(new Message("PingResponse")));
        var pingResult = await ping;

        // Assert
        Assert.True(pingResult);
    }

*Yes - kind of screams 'design phase not completed to our satisfaction', doesn't it?*
