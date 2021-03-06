# PHP Dependency Injector #
A dependency injection container capable of:

1. Auto-wiring.
2. Delegating object creation to factories.
3. Dealing with multiple instances of the same class.
4. Calling methods after instantiation.
5. Mapping interfaces and abstract classes to concrete implementations.
6. Circular dependency detection.
7. And more.

## Table of Contents ##
##### [- Requirements](#requirements) #####
##### [- Installation](#installation) #####
##### [- Instantiating an Object](#instantiating-an-object) #####
##### [- Unresolvable Arguments](#unresolvable-arguments) #####
##### [- Setting Arguments](#setting-arguments) #####
##### [- Mapping to Concrete Implementations](#mapping-to-concrete-implementations) #####
##### [- Calling a Method After Instantiation](#calling-a-method-after-instantiation) #####
##### [- Using Factories (Delegating Instantiation)](#using-factories-delegating-instantiation) #####
##### [- Multiple Instances of the Same Class](#multiple-instances-of-the-same-class) #####
##### [- Sharing Instances](#sharing-instances) #####
##### [- Invoking a Method or Function](#invoking-a-method-or-function) #####

## Requirements ##

Syringe requires PHP 5.4 or higher.

## Installation ##

#### Composer ####

```
composer require david2m/syringe
````

## Instantiating an Object ##

For the sake of simplicity (this being the first example and all), say we want to instantiate an object which has no dependencies:

```php
class UserMapper
{
}
```

```php
$mapper = $injector->make('UserMapper');
```

### Recursively Resolving Parameters ###
```php
class PdoAdapter
{
}
```

The `UserMapper` now depends on a `PdoAdapter`

```php
// UserMapper.php
public function __construct(PdoAdapter $pdoAdapter)
{
}
```

```php
$mapper = $injector->make('UserMapper');
```
When making a `UserMapper` the injector discovers that it depends on a `PdoAdapter` so it creates a `PdoAdapter` object first and injects it into the `UserMapper`.

## Unresolvable Arguments ##
Two scenarios exist where it is impossible to automatically resolve an argument:

1. No type-hint exists - In this situation you must explicitly tell the injector what the argument is. See [setting arguments](#setting-arguments).
2. The type-hint is an interface or abstract class - See [mapping to concrete implementations](#mapping-to-concrete-implementations).

## Setting Arguments ##

```php
class PdoAdapter
{
  public function __construct($host, $user, $password, $schema)
  {
  }
}
```

```php
$constructor = $injector->getConstructor('PdoAdapter');

// Setting one argument
$constructor->setArgument('host', 'localhost');

// Setting multiple arguments
$constructor->addArguments([
  'host' => 'localhost',
  'user' => 'david'
]);
```

You can set the arguments of any method, not just the constructor.

```php
$injector
  ->getMethod('PdoAdapter', 'someMethod')
  ->setArgument('name', 'value');
```

You can also set arguments on the fly:

```php
$injector->make('PdoAdapter', [
  'host' => 'localhost',
  'user' => 'david'
]);
```

Arguments set on the fly always take precedence over arguments set prior to calling `make()`.

This is useful when [adding calls](#calling-a-method-after-instantiation) to a method or using the [invoke()](#invoking-a-method-or-function) method.

### Callable Arguments ###

Callable arguments get invoked and their return value gets passed into the method.

```php
$injector
  ->getConstructor('PdoAdapter')
  ->setArgument('user', function()
  {
    return 'bob';
  });
```

**If the type-hint of the parameter is callable and the set argument is callable then the argument will NOT be invoked before passing it into the method.**

### String Argument Resolved to an Object ###
You can set a class name argument for a parameter which has a class or interface type-hint.

```php
class UserMapper
{
  public function __construct(PdoAdapter $pdoAdapter)
  {
  }
}
```

```php
$injector
  ->getConstructor('UserMapper')
  ->setArgument('pdoAdapter', 'PdoAdapter');
```

When the injector is resolving the argument it will notice it is a string but the type-hint is a class. It will then resolve the string argument (class name) into an object.

This technique is especially powerful when you need to use [multiple instances of the same class](#multiple-instances-of-the-same-class). Say you have two databases, a local and a remote one and both obviously have different connection details. The `UserMapper` connects to the local database and the `Logger` connects to a remote database.

```php
class UserMapper
{
  public function __construct(PdoAdapter $pdoAdapter)
  {
  }
}
```

```php
class Logger
{
  public function __construct(PdoAdapter $pdoAdapter)
  {
  }
}
```

Both objects need a different instance of a `PdoAdapter`, one which can connect to the local database and another which can connect to the remote database.

```php
$injector
  ->getConstructor('PdoAdapter')
  ->addArguments([
      'host' => 'localhost',
      'user' => 'local_user',
      'password' => 'local_password',
      'schema' => 'local_database_name'
    ]);

$injector
  ->getConstructor('PdoAdapter#remote')
  ->addArguments([
      'host' => '103.243.0.78',
      'user' => 'remote_user',
      'password' => 'remote_password',
      'schema' => 'remote_database_name'
    ]);

$injector
  ->getConstructor('Logger')
  ->setArgument('pdoAdapter', 'PdoAdapter#remote');

$mapper = $injector->make('UserMapper');
$logger = $injector->make('Logger');
```

The `UserMapper` will get the `#default` instance of the `PdoAdapter` and the `Logger` will get the `#remote` instance of the `PdoAdapter`. For more information about dealing with multiple instances click [here](#multiple-instances-of-the-same-class).

## Mapping to Concrete Implementations ##

```php
interface DatabaseAdapterInterface
{
}
```

```php
class PdoAdapter implements DatabaseAdapterInterface
{
}
```

```php
// UserMapper.php
public function __construct(DatabaseAdapterInterface $dbAdapter)
{
}
```

When trying to make a `UserMapper` the injector will throw an `InjectorException` because it cannot resolve `DatabaseAdapterInterface` to a concrete implementation. To prevent this you simply map the interface to a concrete implementation i.e. a class which implements the interface.

**You can also map abstract classes to concrete implementations.**

```php
$injector->setMapping('DatabaseAdapterInterface', 'PdoAdapter');

// You can also add multiple mappings at once
$injector->addMappings([
  'DatabaseAdapterInterface' => 'PdoAdapter',
  'AbstractClassName' => 'AnotherConcreteImplementation'
]);
```

## Calling a Method After Instantiation ##
```php
// PdoAdapter.php
public function connect()
{
}
```

```php
$injector
  ->getMethod('PdoAdapter', 'connect')
  ->addCall();

$pdoAdapter = $injector->make('PdoAdapter');
```

Once the `PdoAdapter` has been instantiated the `connect()` method will be called before the object is returned. If the method being called has any parameters they will be resolved and passed into the method. If the method has any [unresolvable arguments](#unresolvable-arguments) you must [set them](#setting-arguments) before you make the object or else pass them to the `addCall()` method.

## Using Factories (Delegating Instantiation) ##
A factory consists of a regular expression and a callable. The regular expression is matched against the name of the class being made, if there is a match then the instantiation of the object is delegated to the factory.

**If the class name you're trying to match contains backslashes you do not need to escape them, this is done automatically by the injector.**

```php
// The regular expression ^PdoAdapter$ will match the classname PdoAdapter
$injector->setFactory('^PdoAdapter$', function()
{
  return new PdoAdapter('localhost', 'david', 'mypassword', 'my_database_name');
});

$pdoAdapter = $injector->make('PdoAdapter'); // Factory instantiates the object.
```

### Advanced Factory Usage ###
Let's say you have some services in your application which reside in the `Service` namespace. For some reason you don't want the injector to instantiate any of the services and want to delegate that job to your own `ServiceFactory`.

```php
namespace Service;

class Recognition
{
}

class Shopping
{
}
```

```php
class ServiceFactory
{
  public function create($className)
  {
    // Instantiate and return the object.
  }
}
```

```php
$injector->setFactory('^Service\\', function($className, ServiceFactory $serviceFactory)
{
  return $serviceFactory->create($className);
});

$recognition = $injector->make('Service\Recognition');
$shopping = $injector->make('Service\Shopping');
```

The instantiation of any class name that begins with `Service\` is delegated to the callable factory. If the callable factory has a parameter named `$className` then the injector will pass the full name of the class which matched the regular expression of the factory.

## Multiple Instances of the Same Class ##
So far we have only dealt with a single instance of the same class. Sometimes you may want multiple instances of the same class. Such a use case would be dealing with multiple databases. Your application may need to connect to a local database and a remote database.

When making an object, setting its parameters or adding calls to its methods you can specify which instance of the object you are referring to by putting a `#instance-name` after the class name. If you do not supply an instance name then you are dealing with the `#default` instance of the class.

```php
$injector->make('PdoAdapter');

// Is the same as
$injector->make('PdoAdapter#default');
```

Two different instances of the same class:

```php
$injector
  ->getConstructor('PdoAdapter')
  ->addArguments([
      'host' => 'localhost',
      'user' => 'local_user',
      'password' => 'local_password',
      'schema' => 'local_database_name'
    ]);

$injector
  ->getConstructor('PdoAdapter#remote')
  ->addArguments([
      'host' => '103.243.0.78',
      'user' => 'remote_user',
      'password' => 'remote_password',
      'schema' => 'remote_database_name'
    ]);

$localPdoAdapter = $injector->make('PdoAdapter');
$remotePdoAdapter = $injector->make('PdoAdapter#remote');

var_dump($localPdoAdapter === $remotePdoAdapter); // bool(false)
```

## Sharing Instances ##
By default, if an object is created by the injector it is stored and used every time an instance of that object is needed.
```php
$userOne = $injector->make('Entity\User');
$userTwo = $injector->make('Entity\User');

var_dump($userOne === $userTwo); // bool(true)
```

You can tell the injector to always create a new instance of an object:

```php
$injector->singleton('Entity\User', false);

$userOne = $injector->make('Entity\User');
$userTwo = $injector->make('Entity\User');

var_dump($userOne === $userTwo); // bool(false)
```

Sharing an object:
```php
$injector->share([$pdoAdapter]);
```

Sharing an object and specifying the instance name:
```php
$injector->share(['remote' => $remotePdoAdapter]);

// To retrieve the $remotePdoAdapter
$remotePdoAdapter = $injector->make('PdoAdapter#remote');
```

## Invoking a Method or Function ##
The `invoke()` method accepts any valid PHP [callable](https://secure.php.net/manual/en/language.types.callable.php) and an optional second parameter which contains arguments you want to pass into the method/function when it is invoked.