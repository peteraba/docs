# Unit of Work

## Table of Contents
1. [Introduction](#introduction)
2. [Example](#example)
3. [Entity Registry](#entity-registry)
    1. [Aggregate Roots](#aggregate-roots)
4. [Comparing Entities](#comparing-entities)
5. [Entity Ids](#entity-ids)
    1. [Id Accessors](#id-accessors)
        1. [Reducing Boilerplate Code](#reducing-boilerplate-code)
    2. [Id Generators](#id-generators)

<h2 id="introduction">Introduction</h2>

**Units of work** act as transactions across multiple repositories.  They also schedule entity updates/insertions/deletions in the data mappers.  The benefits of using units of work include:

1. Transactions across multiple repositories can be rolled back, giving you "all or nothing" functionality
2. Changes made to entities retrieved by repositories are automatically checked for changes and, if any are found, scheduled for updating when the unit of work is committed
3. Database writes are queued and executed all at once when the unit of work is committed, giving you better performance than executing writes throughout the lifetime of the application
4. Querying for the same object will always give you the same, single instance of that object

<h2 id="example">Example</h2>

First, let's create a unit of work:
```php
use Opulence\Orm\ChangeTracking\ChangeTracker;
use Opulence\Orm\EntityRegistry;
use Opulence\Orm\Ids\Accessors\IdAccessorRegistry;
use Opulence\Orm\Ids\Generators\IdGeneratorRegistry;
use Opulence\Orm\Repositories\Repository;
use Opulence\Orm\UnitOfWork;
use Project\Infrastructure\Users\UserDataMapper;

$idAccessorRegistry = new IdAccessorRegistry();
$changeTracker = new ChangeTracker();
$entityRegistry = new EntityRegistry($idAccessorRegistry, $changeTracker);

// Assume $connection was set previously
$unitOfWork = new UnitOfWork(
    $entityRegistry,
    $idAccessorRegistry,
    new IdGeneratorRegistry(),
    $changeTracker,
    $connection
);
```

Next, let's take a look at how units of work can manage entities retrieved through repositories:

```php
// Create our repository
$dataMapper = new UserDataMapper();
$users = new Repository("Project\\Domain\\Users\\User", $dataMapper, $unitOfWork);

// Assume user with ID 123 has username "foo"
$someUser = $users->getById(123);
echo $someUser->getUsername(); // "foo"

// Let's change his username
$someUser->setUsername('bar');

// Committing the unit of work will automatically detect and save changes back to storage
$unitOfWork->commit();

// To prove that this really worked, let's print the user's username
echo $users->getById(123)->getUsername(); // "bar"
```

<h2 id="entity-registry">Entity Registry</h2>

Entities that are scheduled for insertion/deletion/update are managed by an `Opulence\Orm\EntityRegistry`.

<h4 id="aggregate-roots">Aggregate Roots</h4>

Let's say that when creating an invoice you also create a list of line items.  The invoice is what we call an **aggregate root** because, without it, the line items wouldn't exist.  If your line items need to know their invoice's Id before storing them, you can use `registerAggregateRootCallback()`:

```php
// Order here matters: aggregate roots should be added before their children
$unitOfWork->scheduleForInsertion($invoice);
$callback = function ($invoice, $lineItem) {
    $lineItem->setInvoiceId($invoice->getId());
};

// Set the invoice Id for each line item
foreach ($lineItems in $lineItem) {
    $unitOfWork->scheduleForInsertion($lineItem);
    $entityRegistry->registerAggregateRootCallback($invoice, $lineItem, $callback);
}

$unitOfWork->commit();
```

> **Note:** Aggregate root callbacks are executed for entities scheduled for insertion and update.

<h2 id="comparing-entities">Comparing Entities</h2>

`Opulence\Orm\ChangeTracking\ChangeTracker` is responsible for tracking any changes made to the entities it manages.  By default, it uses reflection, which for some classes might be slow.  To speed up the comparison between two objects to see if they're identical, you can use `registerComparator()`.

Let's say that all you care about when checking if two users are identical is whether or not their usernames are identical:

```php
// Assume $changeTracker is the same instance of ChangeTracker in the unit of work
$changeTracker->registerComparator($className, function ($userA, $userB) {
    return $userA->getUsername() == $userB->getUsername();
});
```

> **Note:** PHP's `clone` feature performs a shallow clone.  In other words, it only clones the object, but not any objects contained in that object.  If your object contains another object and you'd like to take advantage of automatic change tracking, you must write a `__clone()` method for that class to clone any objects it contains.  Otherwise, the automatic change tracking will not pick up on changes made to the objects contained in other objects.

<h2 id="entity-ids">Entity Ids</h2>

<h4 id="id-accessors">Id Accessors</h4>

Opulence lets you use plain-old PHP objects with the ORM, which means Opulence doesn't know which methods to call to get and set the unique identifiers in your classes.  So, you must let Opulence know using the `Opulence\Orm\Ids\Accessors\IdAccessorRegistry`:

```php
use Opulence\Orm\ChangeTracking\ChangeTracker;
use Opulence\Orm\EntityRegistry;
use Opulence\Orm\Ids\Accessors\IdAccessorRegistry;

class Foo
{
    private $id;

    public function getId()
    {
        return $this->id;
    }

    public function setId($id)
    {
        $this->id = $id;
    }
}

$idAccessorRegistry = new IdAccessorRegistry();
$entityRegistry = new EntityRegistry($idAccessorRegistry, new ChangeTracker());
// Accepts the entity and must return the identifier
$getter = function ($entity) {
    return $entity->getId();
};
// Accepts the entity and identifier and must set the new identifier
$setter = function ($entity, $id) {
    $entity->setId($id);
};

$idAccessorRegistry->registerIdAccessors(Foo::class, $getter, $setter);
```

`registerIdAccessors()` also accepts an array of class names.

To use the accessor registry in your unit of work, pass it into the unit of work constructor.

> **Note:**  You must always register Id getters, but Id setters are optional.

> **Note:**  If you're using the <a href="https://github.com/opulencephp/Project" target="_blank">skeleton project</a>, you can set your Id accessors in `Project\Application\Bootstrappers\Orm\OrmBootstrapper::registerIdAccessors()`.

If you don't have getter/setter methods for your Id, you can use reflection to get/set it using `registerReflectionIdAccessors()`:

```php
namespace Project\Domain\Users;

use Opulence\Orm\Ids\Accessors\IdAccessorRegistry;

class User
{
    private $id = -1;
}

$idAccessorRegistry = new IdAccessorRegistry();
// The second parameter is the name of the Id property in the User class
$idAccessorRegistry->registerReflectionIdAccessors(User::class, 'id');
```

<h5 id="reducing-boilerplate-code">Reducing Boilerplate Code</h5>

Opulence's flexibility comes at the price of a little bit of boilerplate code on your end to register Id accessors.  However, if you want to get rid of the boilerplate code, you can optionally implement `Opulence\Orm\IEntity`, which has two methods:  `getId()` and `setId($id)`.  Classes that implement `IEntity` automatically have their Id accessors registered.

<h4 id="id-generators">Id Generators</h4>

Opulence can automatically generate Ids for entities managed by the unit of work.  A common way of setting Ids is using sequences from your database.  In this case, you can use:

* `Opulence\Orm\Ids\Generators\BigIntSequenceGenerator`
    * Generates big integer sequence Ids
* `Opulence\Orm\Ids\Generators\IntSequenceGenerator`
    * Generates integer sequence Ids
* `Opulence\Orm\Ids\Generators\UuidV4Generator`
    * Generates UUIDs, which are suitable for distributed databases

All Id generators in Opulence implement `Opulence\Orm\Ids\Generators\IIdGenerator`, which has two methods:

* `generate()`
    * Generates a new Id for the input entity
* `getEmptyValue()`
    * Gets the value of an Id that is considered empty
* `isPostInsert()`
    * Whether or not this generator should be run before or after the unit of work executes its inserts

To let the unit of work know which Id generator to use with your classes, register them to `Opulence\Orm\Ids\Generators\IdGeneratorRegistry`:

```php
use Opulence\Orm\Ids\Generators\IdGeneratorRegistry;
use Opulence\Orm\Ids\Generators\IntSequenceIdGenerator;
use Project\Domain\Users\User;

$idGeneratorRegistry = new IdGeneratorRegistry();
$idGeneratorRegistry->registerIdGenerator(
    User::class,
    new IntSequenceIdGenerator('user_id_seq')
);
```

Then, pass `$idGeneratorRegistry` into the unit of work constructor, and you're set.

> **Note:**  If you're using the <a href="https://github.com/opulencephp/Project" target="_blank">skeleton project</a>, you can set your Id generators in `Project\Application\Bootstrappers\Orm\OrmBootstrapper::registerIdGenerators()`.
