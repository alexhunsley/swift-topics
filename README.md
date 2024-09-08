# swift-topics

# Introduction

It's not unusual to see this kind of code in a project, which represents an action on an app screen:

```swift
enum SettingsScreenAction {
    case screenViewed
    case screenDismissed
    case settingChanged(String, Int)
}
```

Enums are good: they are convenient, expressive value types with nice type safety and exhaustiveness checks; no problem here.

If our app generated analyics, we might also have this:

```swift
enum SettingsScreenAnalytics {
    case screenViewed
    case screenDismissed
    case settingChanged(String, Int)
}
```

Hmm. We have a seperate type just for the analytics, which is probably appropriate, but that repetition of every case is annoying.

And then we might also have something like this:

```swift
extension SettingsScreenAnalytics {
    func analytics(forAction action: SettingsScreenAction) -> SettingsScreenAnalytics {
	switch action {
        case .screenViewed:
           return .screenViewed
        case .screenDismissed:
           return .screenDismissed)
        case let .settingsChanged(key, value):
           return .settingsChanged(key, value)
        }
    }
}
```

Ugh! This repetition seems very tedious. And what's worse, we've coupled our analytics type to our events type. This isn't ideal; imagine if we wanted to break our app into modules and minimise cross-talk.

And suppose that alongside `Action` and `Analytics` we had another aspect such as `Logging`. You now have three very similar enums, and the amount of code mapping these objects to each other could balloon (3 x 2 = 6 possible conversion routines).

So what can we do to make this nicer?

# Some analysis

We want to be able to maintain the type-safety of using something like `enum`, while teasing apart the coupling between `Action`, `Event`, `Analyics`. If we can avoid repeating cases across enums, even better!

It's helpful to recognise that we have two orthogonal concerns here: 

* **topics**, a single example of which is the Settings screen and its things of interest (`screenViewed`, `screenDismissed`, `settingsChanged` -- the "topic values"). 
* **contexts**, which are aspects such as `Action`, `Event`, `Logging`.

# Our example expressed in Swift Topics

If we express our example above using the idea of topics, the setup code for our topic (settings screen) and contexts (action, event) is like this:

```swift
// a topic for our settings screen
enum SettingsTopic: Topic {
    case screenViewed
    case screenDismissed
    case settingChanged(String, Int)
}

// The Action context for SettingsTopic
struct SettingsAction: TopicRepresentable {
    let topic: SettingsTopic
}

// The Event context for SettingsTopic
struct SettingsEvent: TopicRepresentable {
    let topic: SettingsTopic
}
```

And here's how we'd use the topics in this scenario:

```swift
// make a settings screen action and event
let settingsAction = SettingsAction(topic: .screenViewed)
let settingsEvent = SettingsEvent(topic: .screenDismissed)

// OR we can use the `build` helper via the topic to do the same thing:
let settingsAction: SettingsAction = SettingsTopic.screenViewed.build()
let settingsEvent: SettingsEvent = SettingsTopic.screenDismissed.build()
```

Note that in the last two lines above, the `build()` mechanism knows what to make given the type to the left of the `=`: it senses the type of the var we're assigning to and builds the correct thing. This can be handier in some places, e.g. when returning a context from a func (it saves you making a temporary var).

And what about that laborious switch statement for converting an `Action` into an `Event`? Now it's just this single line:

```swift
let settingsEvent = SettingsEvent(mirroring: settingsAction)
```

This `mirroring` init is automatically provided for all applicable contexts. No laborious switch, and no ballooning Nx(N-1) possible code possibilities for N contexts.

What if you don't *want* this automatic conversion across between all possible contexts for some topics? That's something we can do too. More on this later.

# State re-use

We can do enum state re-use in the usual way, using enum associated values.

Here's an example that reuses an `enum ScreenLifecycle` to reduce repetition:

```swift
// a plain enum for the state we want to re-use
enum ScreenLifecycle: Equatable {
    case screenViewed
    case screenDismissed
}

// our topics
enum LoginScreenTopic: Topic, Equatable {
    	// re-use:
    case lifecycle(ScreenLifecycle)
    case usernameFieldUpdated(username: String)
    case logInTapped
}

enum SettingsScreenTopic: Topic, Equatable {
    	// re-use:
    case lifecycle(ScreenLifecycle)
    case settingChanged(String, Int)
}

// our contexts
struct LoginScreenAction: TopicRepresentable {
    let topic: LoginScreenTopic
}

struct LoginScreenEvent: TopicRepresentable {
    let topic: LoginScreenTopic
}

// Usage examples

// we can still use .build() on a LoginScreenTopic
let loginScreenActionViaBuild: LoginScreenAction
    	= LoginScreenTopic.lifecycle(.screenViewed).build()

// and we can still use .mirror() to convert one context to another
// (with the same topic value)
let loginScreenEventViaMirror: LoginScreenEvent
    	= loginScreenActionViaBuild.mirror()
```

# (Afterword)

I thought of the word `aspect` instead of `context`, but aspect oriented programming is a distinct thing and using that word might be confusing.

I did wonder about forking PointFree's `Tagged` library to approach this subject, but it's not quite the right fit. In particular, I don't think Tagged's transparent wrapping of types fits here -- having types to represent contexts like Action or Analytics seems appropriate (it's a place to put helper code, for a start).
And obviously Tagged is about not confusing types in contexts and so translating between them might not fit with the whole raison d'être of the library.
