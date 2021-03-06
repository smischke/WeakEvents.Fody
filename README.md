# WeakEvents.Fody: an add-in for [Fody](https://github.com/Fody/Fody/)
## Summary
Automatic publishing of weak events via compile time code weaving and a supporting runtime library.
## The NuGet package
https://www.nuget.org/packages/WeakEvents.Fody/

    PM>  Install-Package WeakEvents.Fody 
## Your Code
    [ImplementWeakEvents]
    public class ClassWithEvent
    {
        public event EventHandler<EventArgs> Example;
    }
## What Gets Compiled
    public class ClassWithEvent
    {
		private EventHandler<EventArgs> _Example;
		
        public event EventHandler<EventArgs> Example
		{
			add
			{
				_Example += WeakEventHandlerExtensions.MakeWeak<EventArgs>(value, weak_handler => this.Example -= weak_handler);
			}
			remove
			{
				_Example -= WeakEventHandlerExtensions.FindWeak<EventArgs>(_Example, value);
			}
		}
	}
(a lot of detail has been excluded for clarity - see below for more information)

Note that a runtime library is required: `WeakEvents.Runtime.dll` must be shipped with your application.
## Supported Event Types
Any event with a delegate signature that is compatible with `(object, EventArgs)` can be automatically made weak. This includes types derived from `EventArgs`. E.g.

    public event EventHandler BasicEvent;
    public event NotifyCollectionChangedEventHandler NonGenericEvent;
    public event EventHandler<AssemblyLoadEventArgs> GenericEvent;
    public static event EventHandler<AssemblyLoadEventArgs> StaticGenericEvent;
## Rational
When object `subscriber` attaches to an event in object  `publisher`, a strong reference is created from `publisher` to  `subscriber`. If `publisher` has an object lifetime that is independent of `subscriber`, this will prevent `subscriber` from being garbage collected. In other words, `subscriber` will be leaked.

The best solution is to add code to `subscriber` to detach from the event in `publisher`. Unfortunately, in many cases it is difficult for `subscriber` to know when to detach.

Weak events fix this problem by inserting a weak reference between `publisher` and  `subscriber`.
## Performance
Since we are adding code to the stock .NET generated event add/remove methods, there is a performance cost.

The results below are for one `publisher`, 2000 `subscriber` objects and 5000 event invocations on the `publisher` for a total of 10,000,000 event calls - an extreme example. Times are in milliseconds.

|Action|Strong Event|Weak Event|Increase|
|------|------------|----------|----------|
|Subscribe|0|28|N/A|
|Invoke|34|380|11x
|Remove|84|130|1.5x

The application used to generate these numbers is included in the source code.
## What *Really* Gets Compiled
Read this section if you are trying to understand the source code.

The `add` and `remove` methods generated by C# 4.0 look like this:

    public event EventHandler BasicEvent
    {
      add
      {
        EventHandler eventHandler = this.BasicEvent;
        EventHandler comparand;
        do
        {
          comparand = eventHandler;
          eventHandler = Interlocked.CompareExchange<EventHandler>(ref this.BasicEvent, (EventHandler) Delegate.Combine((Delegate) comparand, (Delegate) value), comparand);
        }
        while (eventHandler != comparand);
      }
      remove
      {
        EventHandler eventHandler = this.BasicEvent;
        EventHandler comparand;
        do
        {
          comparand = eventHandler;
          eventHandler = Interlocked.CompareExchange<EventHandler>(ref this.BasicEvent, (EventHandler) Delegate.Remove((Delegate) comparand, (Delegate) value), comparand);
        }
        while (eventHandler != comparand);
      }
    }
(the code uses lock free synchronization to make the `add` and `remove` methods thread safe)

The compile time weaver will first add a callback method to unsubscribe a weak handler from the original event delegate. This is required to prevent a leak of weak event handler instances once `subscriber` instances have been garbage collected: 
    
    [CompilerGenerated]
    private void <BasicEvent>_Weak_Unsubscribe([In] EventHandler<EventArgs> obj0)
    {
      this.BasicEvent = (EventHandler)Delegate.Remove((Delegate) this.BasicEvent, (Delegate) obj0);
    }

The weaver then modifies the generated `add` method, injecting a weak event handler instance by calling `WeakEventHandlerExtensions.MakeWeak`:

      add
      {
        EventHandler eventHandler1 = (EventHandler)WeakEventHandlerExtensions.MakeWeak<EventArgs>(value, new Action<EventHandler<EventArgs>>((object) this, <BasicEvent>_Weak_Unsubscribe));
        EventHandler eventHandler2 = this.BasicEvent;
        EventHandler comparand;
        do
        {
          comparand = eventHandler2;
          eventHandler2 = Interlocked.CompareExchange<EventHandler>(ref this.BasicEvent, (EventHandler) Delegate.Combine((Delegate) comparand, (Delegate) eventHandler1), comparand);
        }
        while (eventHandler2 != comparand);
      }
The `remove` method is then modified by calling `WeakEventHandlerExtensions.FindWeak` to map the `value` to the equivalent weak event handler: 

      remove
      {
        EventHandler eventHandler1 = (EventHandler)WeakEventHandlerExtensions.FindWeak<EventArgs>(this.BasicEvent, value);
        EventHandler eventHandler2 = this.BasicEvent;
        EventHandler comparand;
        do
        {
          comparand = eventHandler2;
          eventHandler2 = Interlocked.CompareExchange<EventHandler>(ref this.BasicEvent, (EventHandler) Delegate.Remove((Delegate) comparand, (Delegate) eventHandler1), comparand);
        }
        while (eventHandler2 != comparand);
      }
    }
    
Note that delegate conversion calls have been stripped from this code for clarity: the weaver will include calls to `DelegateConvert.ChangeType` as needed to cast the relevant delegates.
## Acknowledgements
The weak event implementation is based on Dustin Campbells article ["Solving the Problem with Events: Weak Event Handlers"](http://diditwith.net/PermaLink,guid,aacdb8ae-7baa-4423-a953-c18c1c7940ab.aspx).
The delegate conversion implementation was inspired by Ed Balls article ["Casting Delegates"](http://code.logos.com/blog/2008/07/casting_delegates.html)
## License
The MIT License (MIT) 