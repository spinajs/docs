# Dependency injection

## Basics

SpinaJS DI container is standalone npm package that can be uses separately from main SpinaJS framework.  Within framework you just import from `@spinajs/core` if used standalone add npm package `@spinajs/di` and import from it.

Module exports main DI container as `DI` object, and `Inject`,`LazyInject`, `Autoinject` decorators. For more details see rest of documentation.

### Basic usage

Dependency injection in SpinaJS is done by decorators - ES Next feature supported by Typescript. Lets go straight to example.

Lets say we have class `UserController` that wants some data from database. It depends on some kind of database object that is used all across application, lets call it `Orm`. We don't want to hard code this dependency, neither create new database object every time. We want to get it from DI container that handle it's creation and lifetime.

```typescript
import { Inject } from "@spinajs/core";
import { Orm } from "service/database";

@Inject(Orm)
class UserController{

    private Database : Orm;

    constructor(database : Orm){

        //  database object is avaible and ready to use
        this.Database = database;
    }
}
```

In example code above we used special decorator `@Inject` that tells DI container that class `UserController` depends on `Orm` class, and every time `UserController` is created it must first obtain instance of `Orm` and pass it to constructor. Notice how constructor signature matches list of dependencies in `@Inject` decorator. You can also have as many injected dependencies as you need. Here's a quick example:

```typescript
import { Inject } from "@spinajs/core";
import { Orm } from "service/database";
import { EmailSender } from "service/email";

@Inject(Orm, EmailSender)
class UserController{

    private Database : Orm;
    private Email : EmailSender;

    constructor(database : Orm, email : EmailSender){
        this.Database = database;
        this.Email = email;
    }

    public async registerUser(/** some parameters */){
        await this.Database.insert(/* some data*/);
        await this.Email.sendEmail(/* some data*/);
    }
}
```

{% hint style="info" %}
SpinaJS heavily depends on decorators. To use it you must enable `"experimentalDecorators" : "true"` setting to the `compilerOptions` section of `tsconfig.json`
{% endhint %}

### Auto-inject feature

Also with typescript having information about types it is possible to skip explicit type declaration and constructor initialization. We can tell DI container to automatically inject dependencies based on variable type. It uses Typescript experimental feature that allows to extract variable type by transpiler.  Example below:

```typescript
import { Inject } from "@spinajs/core";
import { Orm } from "service/database";
import { EmailSender } from "service/email";


class UserController{

    @Autoinject
    private Database : Orm;

    @Autoinject
    private Email : EmailSender;

    public async registerUser(/** some parameters */){
        await this.Database.insert(/* some data*/);
        await this.Email.sendEmail(/* some data*/);
    }
}
```

Notice how we use `@Autoinject` decorator with class properties so we get rid of constructor and avoid write tedious code. DI container will automatically extract property type and resolve it before `UserController` is created.

{% hint style="warning" %}
SpinaJS DI container can handle both asynchronous and synchronous object creation. Depending on use case \( and resolve strategies \) auto injected properties should be accessed with or without`async` keyword. Handling such cases are described in [Synchronous and asynchronous resolving](https://app.gitbook.com/@spinajs/s/spinajs/~/edit/drafts/-LlCNE7RtEj9Cl32BJnB/fundamentals/dependency-injection#synchronous-and-asynchronous-resolving)
{% endhint %}

### Lazy inject properties

Sometimes we want to declare dependency without resolving it, we can do this by using `@LazyInject` decorator when declaring class property, like this:

```typescript
class LazyInjectExample {

    @LazyInject(LazyInjectDependency)
    public Instance: LazyInjectDependency;
    
    public DoSomething(){
        this.Instance.SendEmail(); // dependency is created & resolved at usage time
    }
}

```

Instance property is not resolved when `LazyInjectExample` class is created via container. Instead proxy function is created and assigned to `Instance` property that when called for first time resolves target dependency.

{% hint style="warning" %}
SpinaJS DI container can handle both asynchronous and synchronous object creation. Depending on use case \( and resolve strategies \) lazy inject property should be accessed with or without `async` keyword. Handling such cases are described in [Synchronous and asynchronous resolving](https://app.gitbook.com/@spinajs/s/spinajs/~/edit/drafts/-LlCNE7RtEj9Cl32BJnB/fundamentals/dependency-injection#synchronous-and-asynchronous-resolving)
{% endhint %}

### Resolving multiple instances at once

There can be registered multiple implementations of same base class in container. For example we have multiple notification implementation for users lets say email and slack message. First thing to do is to register implementations of interface to container:

```typescript
class EmailNotification extends Notification
{
 // ....
}

class SlackNotification extends Notification
{
 // ....
}

DI.register(EmailNotification).as(Notification);
DI.register(SlackNotification).as(Notification);
```

Then later we can resolve all implementation simply by using extended array type like this:

```typescript

// sending notification to all channels

const notification = await DI.resolve(Array.ofType(Notification));
for(const n of notification){
    await n.send( /** some data **/);
}
```

{% hint style="info" %}
 Notice usage of `Array.ofType` function. It simply tells container to resolve all instances of given type. If you use simple type instead, only first type that was registered will be resolved, in our example it would be `EmailNotification`
{% endhint %}

## Manual resolving and retrieving already created instances

If for some reason you don't want to use DI for heavy work you can resolve instances and their dependencies manually. Lets say you have function in some place in your code that is called by external tool or application, like this:

```typescript
import { DI } from "@spinajs/core";
import { Orm } from "service/database";

async function cleanUserHistory()
{
    const database = await DI.resolve<Orm>(Orm);

    await database.UsersHistory.select().olderThan(5, "days").delete();
}

cleanUserHistory();
```

We use `resolve` function of DI container to resolve dependency.

Di container expose more useful functions for manual usage:

* `has` to check if type is already resolved and exists in container
* `get` to return resolved object if exists in container, if not returns nothing

Examples below:

```typescript
import { DI } from "@spinajs/core";
import { Orm } from "service/database";

async function cleanUserHistory()
{
    if(DI.has(Orm)){

        // Orm instance already exists in container, we can obtain it
        const database = DI.get(Orm);
    }
}

cleanUserHistory();
```

#### Manual resolving and parent container lookup

When using child container we can tell `get` and `has` function to not to check in parent container. To do this simply set second parameter to false \( default is true\)

```typescript
// do not check parent container for Orm instance
if(childContainer.has(Orm, false)) { }

// same for get, if Orm not exists in child but exists in parent it still return null
if(childContainer.get(Orm, false)) { }
```

{% hint style="info" %}
Use `@PerChildInstance` if you want to ensure that child container will not check if object exists in parent before creating new instance.
{% endhint %}

## Root container, child containers and DI namespace

Root container is app-wise main DI container that holds all references for resolved objects. It's default container, accessed via DI namespace as shown in previous examples. All we do by methods exposed in the `DI` namespace use root container.

```typescript
import { DI } from "@spinajs\di"

DI.resolve(SomeType);
// .. etc

```

SpinaJS dependency injection implementation also have concept of child containers. We can create child containers from main `root` container that exists in SpinaJS. Child containers are mainly used to overriding DI configuration \( if you want change default services for different implementations or inject mocked objects in tests \). Created child containers inherits all data from parent container. 

```typescript
import { DI } from "@spinajs\di";

const child = DI.child();

// child container is avaible now
child.resolve(SomeType);
```

#### Child containers with override 

To override already existing type in container simply use `register` function on it. Lets say we want to override basic implementation 

## Resolve strategies

By default our container only calls class constructor when resolving dependencies. This behavior is simple, but sometimes we must do more work at initialization or some asynchronous code, for example connecting to database. To make this possible we can define our custom resolve strategies for DI container. To do this you must implement `IStrategy` interface and register it to container.

```typescript
import { IStrategy } from "@spinajs/di";

/**
 * Resolve strategy to initialize framework internal modules.
 */
export class FrameworkModuleStrategy implements IStrategy {
    
    /**
     * resolve function is where we make our initialization
     */
    public resolve(target: any): Promise<void> | void {
    
        // check if initialize function extists on target & call it 
        if (target && target.initialize && _.isFunction(target.initialize)) {
            return new Promise((res, _) => {
                res(target.initialize());
            });
        }
    }
}



```

In example above we implement this interface, `target` parameter passed to `resolve` function is newly created instance of our object, then we perform custom check \( in this example we check if specific function exists, but it could be anything like `instanceof` check\), and call specific function.

{% hint style="info" %}
Notice that `resolve` function could return either `Promise` or `void`. If resolve returns promise then you should use `await` keyword when resolving object using this strategy. More on this can be found in [Synchronous and asynchronous resolving](https://app.gitbook.com/@spinajs/s/spinajs/~/edit/drafts/-LlCNE7RtEj9Cl32BJnB/fundamentals/dependency-injection#synchronous-and-asynchronous-resolving). This method is very useful for example when connecting to database or external service, checking resource availability etc.
{% endhint %}

This method is used to initialize internal spinajs framework modules. 

{% hint style="info" %}
Strategies can be replaced by factories. Use strategies for initializing large group of similar objects to avoid writing repeating code and improve readability.
{% endhint %}

## Synchronous and asynchronous resolving

Primary job of `resolve` function in DI container is to call object constructor and resolve dependencies same way. It's done in synchronous way - if we know that object is created only like that we can skip async / await and for that matter promise handling.

{% hint style="info" %}
`resolve` method can return `Promiset<T>` or `T` depending on used resolve strategy.
{% endhint %}

Most basic usage is like this:

```typescript
import { DI } from "@spinajs/di";

class Foo{ }

@Inject(Foo)
class Bar {
  constructor(public f : Foo){ }
}

const instance = DI.resolve(Foo);

```

`Bar` instance is resolved synchronously.

## Object lifetime

Objects in container can be instantiated in those three ways:

* As singleton - only one instance exists whole application
* As per child instance - only one instance exists in container. If we create child DI container, new instance will be resolved at that child instance
* As new instance - always create new instance, container will not store it.

To tell DI container which way to use you must decorate class in way described below:

* For singleton there is no need to do this, its default behavior, to mark singleton explicit use `@Singleton` decorator
* For per child instance use `@PerChildInstance` decorator
* For new instance use `@NewInstance` decorator

```typescript
import { Singletom NewInstance, PerChildInstance } from '@spinajs/di';

@Singleton()
class SingletonClass{}

@PerChildInstance()
class ClassPerChild{ }

@NewInstance()
class AlwaysCreatedClass{ }
```

## Overriding objects and explicit container configuration

Until now we just resolved objects without any configuration. It just works, we did not set any special decorators, or code to set up which class can be resolved and how. But for more advanced usage such option is a must.

For example we want to override basic implementation with new one, or configure container using configuration file. We can do this by telling container which class resolves to which implementation. its done by `register` function.

```typescript
import { DI } from '@spinajs/di'

// we tell containet to register Foo class as implementation of FooBasic class
DI.register(Foo).as(FooBasic);

// when we want to resolve FooBasic class
// container looks up in registration table for entries that match FooBasic
const instance = DI.resolve(FooBasic); // Foo instance is created
```

We can also register multiple implementation under one basic class \( for example multiple notification services under one\) and resolve all at once.

```typescript
import { DI } from '@spinajs/di';

DI.register(SmsNotification).as(Notification);
DI.register(SlackNotification).as(Notification);

// all implementations are resolved
const instances = DI.resolve(Array.ofType(Notification); 
```

### Registering as self

We can also register service / class as self. It mean that it will be available under registered name.

```typescript
import { DI } from '@spinajs/di';

DI.register(SmsNotification).asSelf();

const instance = DI.resolve(SmsNotification);
```

{% hint style="info" %}
Registering as self is not needed in most cases. We can skip this and just resolve object

```typescript
import { DI } from '@spinajs/di'

// sms notification is resolved and registered at container
const instance = DI.resolve(SmsNotification);
```
{% endhint %}

### Factory functions

If we want to resolve thid-party services or add some extra initialization code we can register factory function that will be called when class is resolved. To do this we use `register` function and pass function to it.

```typescript
abstract class Database { }
class DatabaseImpl implements Database { }

// register factory function for `Database` abstract class
DI.register((container: Container, connString: string) => {

   // consString is passed when resolving
   return new DatabaseImpl(connString);
}).as(Database);

const instance = await DI.resolve(Database, ["root@localhost"]);
```

Notice how we pass additional parameters to factory function just by passing array to `resolve`. Parameter list match order in that array. Also always first argument of factory function is reference to current  DI container \( to allow resolve other deps, configure container etc. \)

## Passing parameters to object constructor

To pass additional parameters to class when resolving, just add array with parameters to `resolve` function

```typescript
class Database
{
    constructor(connectionString : string, debug : boolean, someVar : any){}
}

DI.resolve(Database, ["root@localhost", true, someVariable]);
```

{% hint style="info" %}
Notice how order in passed table match parameters in constructor. 
{% endhint %}

Same goes for passing variables to factory functions, find example at [Factory functions](https://app.gitbook.com/@spinajs/s/spinajs/~/edit/drafts/-LlCNE7RtEj9Cl32BJnB/fundamentals/dependency-injection#factory-functions)

If for some reason you want pass variables to class dependencies, best way to do this is to implement factory function, and resolve manually needed dependencies.

```typescript
class SqlConnection {
    constructor( connectionString : string){}
}

class Orm { 
    constructor(modelFolder: string, connection : SqlConnection) {} 
}

DI.register((container : Container, modelFolder : string, connstring : string) =>{
    const connection = container.resolve(SqlConnection, [connstring]);
    return new Orm(modelFolder, connection);
});

const orm = DI.resolve(Orm, ["\home\spina\models", "root@localhost"]);
```

## How SpinaJS uses DI internally

