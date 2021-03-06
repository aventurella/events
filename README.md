# Events

Represents an attempt to bring backbone style events to Swift.

Delegates are fine, and they are probably more performanent, but
it is so much faster to just keep coding than to:

1. Stop coding
2. Create a new protocol
3. Create an implementation of that protocol
4. Add the delegate to the object that needs it
5. Alter your code to use the delegate


Events let you just keep going. Need to communicate between N
Child ViewControllers to their parent ViewController, no problem with
events:

**(Lots more example in the unit tests).**

```swift
import Events

enum ColorEvents: String{
    case Blue = "some:blue:event"
    case Red = "some:red:event"
}

// EventManagerHost is a protocol, stick it on anything you want.
// it only requires you add an appropriately accessible instance variable
class ParentViewController: EventManagerHost {
    let eventManager = EventManager() // <--- the only thing you need. Welcome to using events.

    func viewDidLoad(){
        super.viewDidLoad()
        let blue = BlueViewController(nibName: "Blue", bundle: nil)
        let red = RedViewController(nibName: "Red", bundle: nil)

        // see how we define these handlers below, it's important!
        listenTo(blue, event: ColorEvents.Blue, callback: onBlueEvent)
        listenTo(red, event: ColorEvents.Red,  callback: onRedEvent)

        // String based Enums are now the preferred method to register
        // events. Why? One of the problems with the event approach is
        // you end up litering your code with strings. Capturing the
        // the events in an Enum provides some in-code documentation
        // about what events are available from both a reading the source
        // perspective and a from an IDE code completion perspective.
        // you can still use regular strings as before, but from a
        // maintainability and ease-of-use perspective their use
        // is discouraged. For reference using plain strings looks
        // like this:
        // listenTo(blue, event: "some:blue:event", callback: onBlueEvent)
        // listenTo(red, event: "some:red:event",  callback: onRedEvent)

        addChildViewController(blue)
        addChildViewController(red)

        view.addSubView(blue.view)
        view.addSubView(red.view)
    }

    // Note that it casts the values back to what you expect
    // vs NSNotification which requires manual casting at runtime
    // also note the specific construction of this handler. lazy var
    // with [unowned self] capture list
    //
    // see: https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AutomaticReferenceCounting.html#//apple_ref/doc/uid/TP40014097-CH20-ID57
    // Resolving Stong Reference Cycles for Closures
    // for why we do this.
    //
    // TL;DR: using an instance method like: func onBlueEvent(...)
    // strongly captures self.
    lazy var onBlueEvent: (BlueViewController, Int) -> () = { [unowned self]
        (sender, data) in
        print("\(sender): \(data)")
    }

    // Note that it casts the values back to what you expect
    // vs NSNotification which requires manual casting at runtime
    lazy var onRedEvent: (BlueViewController, Bool) -> () = { [unowned self]
        (sender, data) in
        print("\(sender): \(data)")
    }

    func viewDidDisappear(animated: Bool){
        super.viewDidDisappear(animated)

        // free the listeners
        stopListening()

        // you can also use String Enums to stopListening
        // stopListening(blue, ColorEvents.Blue)
        // stopListening(red, ColorEvents.Red)
    }
}

class BlueViewController: EventManagerHost{
    let eventManager = EventManager()

    func viewDidLoad(){
        super.viewDidLoad()
        doSomething()
    }

    func doSomething(){
        trigger(ColorEvents.Blue, data: 36)

        // Using the strings (discouraged)
        // trigger("some:blue:event", data: 36)
    }
}

class RedViewController : EventManagerHost {
    let eventManager = EventManager()

    func viewDidLoad(){
        super.viewDidLoad()
        doSomething()
    }

    func doSomething(){
        trigger(ColorEvents.Red, data: true)

        // Using the strings (discouraged)
        // trigger("some:red:event", data: true)
    }
}
```

### EventBus

This feature is directly inspired by Backbone.Radio

https://github.com/marionettejs/backbone.radio

It exposes namespaced channels/event system and a request and response system.
Lets take a look at how you might use it.

First of all, when you register channels on the EventBus, they stick around
which means you can get a handle on them from anythign that might need them.

On the flip side, they stick around so put that in your
"things to remember re: memory" back pocket.

```swift
import Events

enum MyEvents: String {
    case Action
}


class Foo: EventManagerHost{
    let eventManager = EventManager()
    let statusChannel = EventBus.localChannel.get("status")

    func doSomething(){
        statusChannel.trigger(MyEvents.Action, data: "lucy")
    }
}

class Bar: EventManagerHost{
    let eventManager = EventManager()
    let statusChannel = EventBus.localChannel.get("status") // it's the same channel as above!
                                                            // You could also initialize it with
                                                            // with an RawRepresentable string enum
                                                            // to alleviate the polluting your
                                                            // app with "random strings"

    init(){
        listenTo(statusChannel, event: MyEvents.Action, callback: onAction)
    }

    lazy var onAction: (LocalChannel, String) = {
        (sender, value) in
        print("got value: \(value)")
    }
}

```


Channels support the same API as any `EventManagerHost` in fact they are an
`EventManagerHost`. They do come with some extras built in: Request/Reply.


### Channel Request/Reply

Request/Reply basically lets you call a function somewhere else and get back
it's result. It sort of lets your turn your app into an API unto itself allowing
you to keep things separate.

Let see how that looks.

In the above example, if you wanted to pass some information around, you would
have to do some extra typing:

```swift
import Events

enum PetType: String {
    case Dog
}

enum Request: String{
    case WantsType
}

enum Reply: String{
    case ReceivedType
}


class Foo: EventManagerHost{
    let eventManager = EventManager()
    let statusChannel = EventBus.localChannel.get("status")

    func getType(){
        listenTo(statusChannel, event: Reply.ReceivedType, callback: onReceivedType)
        statusChannel.trigger(Request.WantsType, data: "lucy")
    }

    lazy var onReceivedType: (LocalChannel, PetType?) = {
        (sender, value) in
        print("got reply: \(value)")
        stopListening(Reply.ReceivedType)
    }
}

class Bar: EventManagerHost{
    let eventManager = EventManager()
    let statusChannel = EventBus.localChannel.get("status")
    let types = ["lucy": PetType.Dog]

    init(){
        listenTo(statusChannel, event: Request.WantsType, callback: onWantsType)
    }

    lazy var onWantsType: (LocalChannel, String) -> () = {[unowned self]
        (sender, value) in
        print("got request: \(value)")

        let petType: PetType?

        if let type = self.types[value.lowercaseString]{
            petType = type
        }

        statusChannel.trigger(Reply.ReceivedType, data: petType)
    }
}
```

Yikes, quite a bit in play there!

- Our classes have to be `EventManagerHost`s.
- We have to retrigger on the channel once our event handler fires.
- We need a request and reply event type

We can make this a lot shorter with a channel's built in Request/Reply.
Same example as above, but now done with Request/Reply on the channel.

```swift
import Events

enum PetType: String {
    case Dog
}

enum Request: String{
    case WantsType
}


class Foo{
    let statusChannel = EventBus.localChannel.get("status")

    func getType(){
        // Here we send a request of type 'Request.WantsType'
        // with 1 argument (you can send more than 1 argument).
        // We are then expecting that once the reply handler does
        // it's work, our callback will receive a value of 'PetType?'

        statusChannel.request(Request.WantsType, "lucy"){
            (value: PetType?) in
            print(value)
        }
    }
}

class Bar{
    let statusChannel = EventBus.localChannel.get("status")
    let types = ["lucy": PetType.Dog]

    init(){
        // the first argument is the type name of the event
        // we want to reply to. The Callback is the function we would
        // like to be invoked, when we receive a request.
        // In this case Any 'request' of type 'Request.WantsType'
        // will cause 'onWantsType' to be invoked and it's return value
        // will be passed back to whoever initiated the request.
        statusChannel.reply(Request.WantsType, callback: onWantsType)
    }

    lazy var onWantsType: (String) -> PetType? = {[unowned self]
        (arg) in
        print("got request with arg: \(arg)")

        let petType: PetType?

        if let type = self.types[value.lowercaseString]{
            petType = type
        }

        return petType
    }
}
```

It's quite a bit simpler, not so much house keeping to do anymore.

So that's pretty cool right? Why `localChannel` and not just `channel`?
Turns out we can adapt this to also cover things like WatchOS communication
using the same API. The primary difference being that **watchOS**.


#### Watch Request/Reply
**watchOS 2 Extension**
```swift
import Events

enum Request: String{
    case Add
}

class InterfaceController: WKInterfaceController{
    let phoneChannel = EventBus.watchChannel.get() // no value the channel is "default"

    @IBAction func onPress(){
        // this will send the request to the phone.
        // if you wanted channel communication on between
        // WKInterfaceController you would use a LocalChannel
        // not a WatchChannel.
        phoneChannel.request(Request.Add, 8){ (value: Int) in
            print("Got Value: \(value)")
        }
    }
}
```


**iOS App**
```swift
import Events

enum Request: String{
    case Add
}

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
        let watchChannel = EventBus.watchChannel.get() // again, no explicit channel
                                                       // name provided then "default" is
                                                       // assumed.

        watchChannel.reply(Request.Add){
            (value: Int) -> Int in
            return 22 + value
        }

        return true
    }
}
```

#### Watch trigger/listenTo

It should be noted, that not only do request/reply work over the watch channel,
but so does `trigger` / `listenTo` events: (request/reply uses the same system):


**watchOS 2 Extension**
```swift
import Events

enum MyEvents: String{
    case GotoIndex
}

class InterfaceController: WKInterfaceController{
    let phoneChannel = EventBus.watchChannel.get() // no value the channel key is "default"

    @IBAction func onPress(){
        phoneChannel.trigger(MyEvents.GotoNext, data: 2)
    }
}
```


**iOS App**
```swift
import Events

enum MyEvents: String{
    case GotoNext
}

class MyViewController: UIViewController, EventManagerHost {
    let eventManager = EventManager()
    let watchChannel = EventBus.watchChannel.get() // no value the channel key is "default"

    override func viewWillAppear(animated: Bool) {
        super.viewWillAppear(animated)
        listenTo(watchChannel, MyEvents.GotoNext, callback: onGotoNext)
    }

    override func viewWillDisappear(animated: Bool) {
        super.viewWillDisappear(true)
        stopListening(MyEvents.GotoNext)
    }

    lazy var onGotoNext: (Int) -> () = {[unowned self]
        (value: Int) in

        // maybe load a different storyboard based on the index
        let sb = UIStoryboard(name: "SomeStoryboard", bundle: nil)
        let viewController = sb.instantiateInitialViewController()!
        self.navigationController?.pushViewController(viewController, animated: true)
    }
}
```

This example works both ways. You could also trigger from the phone to
the watch in the same way.



