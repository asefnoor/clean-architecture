# Clean Architecture for iOS

This is an example of how to architect an iOS application by decoupling completely the UI and the business logic.

This example uses UIKit for the UI but it should be easy to replace this UI with a CLI or any other delivery mechanishm.

### Basic concepts
#### Entities
You should be familiar with this type of objects. An entity represents a domain-specific concept. Department, Employee, Sale, Return, Flight, etc.

#### Implementation specific code
Not domain-related code that is required in order to make the system work as expected. Modules that abstract persistance, networking, etc will fall in this category

#### Interactors
An interactor represents a Use Case of the app. It is responsible for:
* Managing the interaction between multiple Entities
* Keep the application state, given that iOS apps are not service oriented.
* Define a strict boundary around the business logic.
* Deal with other implementation specific details like calling the persistance layer, networking to third-party systems, etc.

#### Delivery mechanism
The specific implementation that the user will use to perform actions on the system and to comunicate the state of the system back to the user. Call it GUI, Web, JSON API, CLI, multitouch screen, Rails, etc.

#### Boundaries
Interactors define a strict boundary to separate the delivery mechanism from the business logic. This boundaries represent some request to the interactor and some response from it.

In this example, ViewControllers capture a request from the user via the UI, interpret what the user wants and request the interactor to perform that action via the designated boundary. The interactor will then return a response with the result of that action. The ViewController will unpack that response and present it back to the user.

We can find boundaries of multiple types in this example:

- Method call to the interactor to query the application state. 
  - A simple property getter in the interactor.
- Method call to an interactor to mutate the application state.
  - A method call.
  - A property setter (yeah, method call).
  - A class representing a request that will be passed as an argument to a method of the interactor.
  
The interactor can respond in different ways, depending on the needs:

- It can return void in a method call. You should use a query mechanism to find out the result, if you care at all.
- It can return the result synchronously. The getter of property for instance.
- It can return synchronously a class representing the response, giving you access to the result of the request, status, errors. 

This last approach is specially interesting because you can include future-like properties in this response. For example, you request to the interactor to login a user, given the username and password comming from the UI using a method call. 

```objc
LSLoginResponse  = [interactor loginWithUsername:username password:password];
```

This operation is asynchrnous in nature because it makes an HTTP request to a backend. In order to avoid passing blocks as arguments for dealing with callbacks, the method call will return synchronously a response object. The asynchronous operation will start right away but you don't know when it will finish. This response object will give you different options:


##### Asynchronously
The most common way will be to pass a callback after the fact, to get notified of the completion

```objc
response.onCompletion = ^{
  // Process response
}
```
Same with errors
```objc
response.onFailure = ^{
  // Process errors
}
```

##### Future
You can get the result synchronously, potentially blocking the main thread. This is specally useful if you make the request and you are not going to use the response for a while. In that time the underlaying asynchronous request will finish and you can avoid callbacks. If the request is not done when asking for the result, it will block until it gets the result. Another good use of this is for testing!

```objc
User *user = response.loggedInUser;
```

Same applies to errors;

```objc
User *user = response.error;
```

##### Query the state of the request
- Check whether the request is done:
```objc
BOOL isDone = response.isDone
```

- Check whether the request was successful:
```objc
BOOL isSuccessful = response.isSuccessful;
```

#### Service
A service is a process that runs for the lifetime of the app. Think of analytics frameworks, crash report tools, etc. You set them up at startup time and you forget about them. You create one subclass of LSService for each service and register an instance of each service at startup time. This way the code related to this specific things won't be scatered in your AppDelegate, each one will have its own place to leave.

A service will get notified for changed in the state of the app (most of the methods of the UIApplication Delegate protocol), in order to act consequently.