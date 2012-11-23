This is a real world example of an [integration fault](http://forums.puremvc.org/index.php?topic=308.0) that occurred in the  
well-established open source project [PureMVC](http://puremvc.org/).  
  
### Brief description  
  
System notifications trigger execution of more commands than expected when  
commands are re-registered.  

### The meat  
 
Think of the MVC pattern. This framework hides all the hassle of the component  
relations and communication, and it gives you clear interfaces for distributing  
roles and responsibilities in your system.  
  
`Controller` has a `registerCommand(notification, command)` method for  
registering a particular `Command` as the handler for a particular  
`Notification`.  It's implemented such that when `registerCommand(notification, command)`  
is called a dedicated `Observer` object is created in the `View` component.  
Thus for each unique `Command` and `Notification` known to the  
`Controller` there are corresponding `Observers` in the `View`.   
  
### Failure  
  
The integration fault (<http://forums.puremvc.org/index.php?topic=308.0>) was  
manifested this way:  When the `Controller` component registers the same  
`Command` after having unregistered it, the `View` component notifies the event  
twice and not only once as expected.   
  
### What went wrong  
  
Unregistering the `Command` by `Controller` did not remove  
the corresponding `Observer` from the notification list in the `View`.  So when  
`View` broadcasted some `Notification` to `Observers` there were duplicated  
`Observers` that executed the `Command` twice upon notification.  
  
### Root cause  
  
`View` did not check for the redundancy in the list of its `Observers`. At the  
same time `Controller` could not control the list of `Observers` in the `View`  
because `View` simply did not have `removeObserver()` method or any other means  
to do so.  
  
### Source code before

This is how the code looked like when the system still had a problem:

```java
	/**
	 * Register a particular <code>ICommand</code> class as the handler for a
	 * particular <code>INotification</code>.
	 * 
	 * <P>
	 * If an <code>ICommand</code> has already been registered to handle
	 * <code>INotification</code>s with this name, it is no longer used, the
	 * new <code>ICommand</code> is used instead.
	 * </P>
	 * 
	 * The Observer for the new ICommand is only created if this the 
	 * first time an ICommand has been regisered for this Notification name.
	 */
	public void registerCommand( String notificationName, Class commandClassRef )
	{
		if (null != this.commandMap.put( notificationName, commandClassRef )) return;
		this.view.registerObserver( notificationName, new Observer( new IFunction()
		{
			public void onNotification( INotification notification )
			{
				executeCommand( notification );
			}
		}, this ) );
	}

	/**
	 * Remove a previously registered <code>ICommand</code> to
	 * <code>INotification</code> mapping.
	 * 
	 * @param notificationName
	 *            the name of the <code>INotification</code> to remove the
	 *            <code>ICommand</code> mapping for
	 */
	public void removeCommand( String notificationName )
	{
		this.commandMap.remove( notificationName );
	}
```


### Source code after 

This is how the code looked like when the problem was fixed:

```java
	public void registerCommand( String notificationName, ICommand command )
	{
		if( null != this.commandMap.put( notificationName, command ) )
			return;
			
		view.registerObserver
		(
			notificationName,
			new Observer
			(
				new IFunction()
				{
					public void onNotification( INotification notification )
					{
						executeCommand( notification );
					}
				},
				this
			)
		);
	}

	public void removeCommand( String notificationName )
	{
		// if the Command is registered...
		if ( hasCommand( notificationName ) )
		{
			// remove the observer
			view.removeObserver( notificationName, this );
			commandMap.remove( notificationName );
		}
	}  
```

### Test case

Original unit test cases did not detect the problem, because they were focused  
on individual components and not on their interactions.

This is a test case developers used to verify that problem is gone:

```java
	/**
	 * Tests Removing and Reregistering a Command
	 * 
	 * <P>
	 * Tests that when a Command is re-registered that it isn't fired twice.
	 * This involves, minimally, registration with the controller but
	 * notification via the View, rather than direct execution of
	 * the Controller's executeCommand method as is done above in 
	 * testRegisterAndRemove. The bug under test was fixed in AS3 Standard 
	 * Version 2.0.2. If you run the unit tests with 2.0.1 this
	 * test will fail.</P>
	 */
	@Test
	public void testReregisterAndExecuteCommand() {

		// Fetch the controller, register the ControllerTestCommand2 to handle 'ControllerTest2' notes
		IController controller = Controller.getInstance();
		controller.registerCommand("ControllerTest2", new ControllerTestCommand2());

		// Remove the Command from the Controller
		controller.removeCommand("ControllerTest2");

		// Re-register the Command with the Controller
		controller.registerCommand("ControllerTest2", new ControllerTestCommand2());

		// Create a 'ControllerTest2' note
		ControllerTestVO vo = new ControllerTestVO(12);
		Notification note = new Notification("ControllerTest2", vo, null);

		// retrieve a reference to the View from the same core.
		IView view = View.getInstance();

		// send the Notification
		view.notifyObservers(note);

		// test assertions 
		// if the command is executed once the value will be 24
		Assert.assertEquals("Expecting vo.result == 24", vo.result, 24);

		// Prove that accumulation works in the VO by sending the notification again
		view.notifyObservers(note);

		// if the command is executed twice the value will be 48
		Assert.assertEquals("Expecting vo.result == 48", vo.result, 48);

	} 
```

### FYI  
  
_PureMVC_ is a lightweight framework for creating applications based upon the  
classic Model-View-Controller design pattern.  _PureMVC_ implements a  
publish/subscribe-style observer notification scheme that allows asynchronous  
event-driven communications between the actors in the system.  It is ported to  
ActionScript 2/3, C++, C#, ColdFusion, Dart, Haxe, Java, JavaScript, Objective  
C, Perl, PHP, Python, Ruby, TypeScript!  

---
[back to Wiki](https://github.com/rubinovk/integration-faults/wiki)
