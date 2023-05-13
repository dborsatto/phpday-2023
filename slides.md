---
theme: ./theme
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
css: unocss
title: 'From domain to code: a practical approach'
colorSchema: light
---

# From domain to code:<br>a practical approach

<br><br>

### phpday 2023

<!--
Stuff that works well with DDD
-->

---
layout: section
transition: fade-out
---

# 👋 Hello there!

<br>

- I'm Davide (he/him)
- PHP Developer for 15+ years
- Technical Leader at Reverse
- I love building stuff

<!--
- Who has ever heard about domain-driven design?
- This talk is not about domain-driven design itself, but an understanding of what it is will help you follow what we talk about
-->

---
layout: section
transition: slide-up
---

# About Domain-Driven Design

<br>

> Domain-Driven Design is an approach to software development that centers the development on programming a domain model that has a rich understanding of the processes and rules of a domain (Martin Fowler)

<!--
- It's less about code and more about learning how to talk to people
- It's about understanding what your users need, how your business actually works
-->

---
layout: center
transition: slide-up
---

<img src="/images/ddd-book.jpg" class="m-80 h-80 shadow" />

---
transition: fade-out
---

## Cheat sheet

- <strong>Ubiquitous language</strong>: a common language used to describe the domain, spoken by developers, stakeholders, and users, and so on.
- <strong>Bounded context</strong>: the scope within which the ubiquitous language is applied
- <strong>Aggregate</strong>: a cluster of associated objects that are treated as a unit (for instance an order and its line items). One of its components is known as the _aggregate root_, which is the "entry point" for the whole aggregate.
- <strong>Invariant</strong>: an assertion about a business rule that must be true at all times.
- <strong>Repository pattern</strong>: an abstraction over storage of aggregates.
- <strong>Event</strong>: something in your domain that has happened in the past.

---
layout: center
transition: fade-out
---

# What problem are we trying to solve?

Well, this is much more complex to get started than usual!

<!--
- We are going to take a look at some of the choices, ideas, and tools so the next time you may want to implement DDD in a project, you can think back to this presentation and say huh, I can do *that*
- It's not meant to be a complete guide, and I'm not claiming any of this to be the best way. This is something that works for us, so hopefully it may be helpful for someone else!

Most of this is based on a Symfony + Doctrine project

-->

---
layout: center
transition: slide-left
---

# Architecture

<!--
While DDD does not directly force any architectural pattern, there are two that are kind of de-facto standards
-->

---
transition: slide-left
---

# Command-Query Responsibility Segregation

- A _command_ is an instruction to perform a specific task
- A _query_ is a request for information

They both work with the concept of _handlers_, which are those elements (classes) that actually process a command/query and do something with them

<!--
- CQRS is an architectural pattern that treats retrieving data and changing data differently
- Commands and queries are simple value objects which contain data, nothing more than that
- The fantastic thing about it is that it forces you to name every interaction with your domain: it's great for communicating with the business people, it clearly expresses intents, it improves testability...
-->

---
transition: slide-up
---

# Layered architecture

Also <em>kind of known</em> as

- Hexagonal architecture
- Clean architecture
- Onion architecture
- Ports and adapters

<!--
- They are all based on grouping your code according to some logical boundaries, which we're about to see
- This will be a pragmatic implementation of a layered architecture, along with some "good enough" guidelines of what goes where
-->

---
transition: slide-up
---

## Layered architecture: Domain layer

- The base layer: it encapsulates the business rules that must be true at all times
- It contains the subjects of your business (aggregates) and their behavior
- It behaves the same across all platforms and systems: it must contain “placeholders” (as interfaces) where it can’t be completely independent and agnostic

<!--
- Imagine a book entity, and it must have the ISBN code
-->

---
layout: section
transition: slide-up
---

```php
// Entities
class Book { /** ... */ }

// Value objects
class BookId { /** ... */ }
class Isbn { /** ... */ }

// Interfaces for repositories
interface BookRepositoryInterface { /** ... */ }

// Exceptions use by elements in the domain layer
class BookWithGivenIdCouldNotBeFoundException extends Exception { /** ... */ }
class IsbnIsNotValidException extends Exception { /** ... */ }
```

---
transition: slide-up
---

## Layered architecture: Application layer

- This layer encapsulates application-specific business rules, it's what your application can actually do
- It has knowledge of the domain, and works with it to provide use cases for the users
- Just like the domain, it’s still quite abstract so it can’t interact with external actors

<!--
Commands and queries fit nicely here, as they provide entry points for interacting with your system
-->

---
layout: section
transition: slide-up
---

```php
class AddBookCommand {
    public function __construct(
        public readonly BookId $id,
        public readonly Isbn $isbn,
        public readonly $title,
    ) {}
}

class AddBookHandler {
    public function __construct(
        private readonly BookRepository $bookRepository,
    ) {}

    public function __invoke(AddBookCommand $command): void {
        $book = new Book($command->id, $command->isbn, $command->title);
        
        $this->bookRepository->save($book);
    }
}
```

---
transition: slide-up
---

## Layered architecture: Infrastructure layer

This is where you'll find the concrete implementations of the interfaces defined in the domain and application layer, and anything strictly related

---
layout: section
transition: slide-up
---

```php
class DoctrineBookRepository implements BookRepositoryInterface {
    public function save(Book $book): void {
        try {
            $this->entityManager->persist($book);
            $this->entityManager->flush();
        } catch (Exception $exception) {
            throw BookCouldNotBeSavedException::create($book, $exception);
        }
    }
}
```

---
transition: slide-up
---

## Layered architecture: User Interface

- This where you'll have the code that is specific to any platform where you application runs
- It contains controllers, CLI commands, forms, API response definitions, etc

<!--
- Sometimes called presentation layer
-->

---
layout: section
transition: fade-out
---

```php
class AddBookController {
    public function __invoke(
        CommandBusInterface $commandBus,
        QueryBusInterface $queryBus,
        Request $request
    ) {
        // Map your request to $bookId, $isbn and $title values
        
        $command = new AddBookCommand($bookId, $isbn, $title);
        $commandBus->dispatch($command);
        
        $query = new GetBookQuery($bookId);
        $result = $queryBus->dispatch($query);
        
        // Do something with $result->book
    }
}
```

---
layout: center
transition: slide-left
---

# Code

<!--
- Don't be afraid of creating your own abstractions.
- They allow you to avoid having your framework leak into your domain and application
- They also make upgrades easier
-->

---
transition: slide-up
---

# How to organize your code

Do not organize by type (Controller, Entity, etc): organize by something that has meaning in your domain

<!--
- We go back to the idea of focusing on our domain needs rather than on the technical aspects
-->

---
layout: section
transition: slide-left
---

```shell
$ tree src/
src
└── Library  # This is your main bounded context
    └── Book  # The aggregate root
        ├── Application
        │   └── Command  # The same structure applies to Query
        │       └── AddBook  #  Commands live alongside their handlers
        │           ├── AddBookCommand.php
        │           └── AddBookHandler.php
        ├── Domain
        │   ├── Book.php  # Keep aggregate roots and other entities at the base level
        │   ├── Exception
        │   │   ├── BookWithGivenIdCouldNotBeFound.php
        │   ├── Model
        │   │   └── Isbn.php
        │   └── Repository
        │       └── BookRepository.php
        └── Infrastructure
            └── Doctrine  # Try to group infrastructure elements by a shared trait, like with "Doctrine" here
                ├── Entity
                │   └── Book.orm.xml
                ├── Repository
                │   └── DoctrineBookRepository.php
                └── Type
                    └── IsbnDoctrineType.php
```

<!--
- A very pragmatic approach is to use bounded contexts and aggregate roots as a way of grouping concerns
-->

---
layout: section
transition: slide-up
---

# IDs

Aggregates must be valid at all times: this means IDs must be handled by your application and not by the database

<!--
- One of the first problems you'll face is IDs: every entity needs to be valid at all times (that's an invariant), so we cannot wait for the database to have an ID 

-->

---
transition: slide-up
---

## UUIDs

`df34b7ea-744e-448e-99fa-f47a8e3929a1`

- The easy way of generating in your application an identifier which is <em>virtually</em> unique at all times
- They contain some randomness, so their creation must be abstracted from the domain layer: use a library such as `ramsey/uuid` or `symfony/uid`

<!--
- How many of you know what a UUID is?
- The chances of collisions are so low they can be ignored
-->

---
layout: section
transition: slide-left
---

```php
// Having the ID defined in the constructor allows the entity to be immediately valid
// without having to wait for the database to have an identity value
class Book {
    public function __construct(
        private readonly BookId $bookId,
    ) {}
}
```

<!--
- We create a book ID value object which encapsulates the UUID, and we give it to the entity constructor
- Our entity is immediately valid, without having to wait for the database
- SPEAKING OF DATABASES
-->

---
layout: section
transition: slide-up
---

# Doctrine and DDD

<!--
- How do we make Doctrine play nice with all of this?
- It's quite easy, but there are a few tips I can give you
-->

---
transition: slide-up
---
## Attributes VS XML

XML metadata allows you to keep Doctrine outside your entities...

...but not completely, because of collections 😢

```php
class Order {
    private \Doctrine\Common\Collections\Collection $lineItems;
    public function __construct() {
        $this->lineItems = new \Doctrine\Common\Collections\ArrayCollection();
    }
}
```

Entity behavior can still be unit-tested regardless of how you define metadata!

<!--
- Because of how Doctrine handles collections, it's inevitable to have some Doctrine code in your entities
- In the end, attributes and XML are both good enough options
-->

---
transition: slide-up
---

## Aggregates with Doctrine

The overall idea is to make Doctrine work well with clusters of objects (aggregates), treating them *as a single unit* and handling their consistency boundaries correctly

---
transition: slide-up
---

## Cascades

```php
class Order {
    private ShippingAddress $shippingAddress;
}
```

```xml
<one-to-one field="shippingAddress" target-entity="App\ShippingAddress" inversed-by="order" orphan-removal="true">
    <!--
    In 1:1 relations, keep the identifier inside the entity closest to the aggregate root,
    this way the owner of the relation is the "parent" and not the "child"
    -->
    <join-column name="shipping_address_id" referenced-column-name="id" nullable="false" />
    <cascade>
        <cascade-all />
    </cascade>
</one-to-one>
```

<!--
- Configure cascade and orphan removal options in a way that persistence and removal operations on your aggregate root are automatically transferred to all the other members of the aggregate
-->

---
transition: slide-up
---

## Change tracking policy

The default option is called `DEFERRED_IMPLICIT`, which means Doctrine will flush all changes to every entity to the database

```php
// ...
$book = $entityManager->find(Book::class, $bookId);
$book->rename('...');

// This will save the new name to the database
$entityManager->flush();
```

---
transition: slide-up
---

Using `DEFERRED_EXPLICIT`, Doctrine will flush only changes to those entities that were explicitly set to be persisted or removed, along with their cascades

```xml
<entity name="App\Book" table="book" change-tracking-policy="DEFERRED_EXPLICIT">
<!-- ... -->
</entity>
```

```php
$book = $entityManager->find(Book::class, $bookId);
$book->rename('...');

// This * will not * trigger a save
$entityManager->flush();

// This * will *
$entityManager->persist($book);
$entityManager->flush();
```

<!--
- This goes back to the idea of treating an aggregate as a single unit, and having database operations be as limited as possible.
-->

---
transition: slide-left
---

## Custom types

- Custom Doctrine types allow you to map a column to a value object, and vice versa
- Great for persisting value objects inside your aggregates
- Almost no perceivable performance impact
- Better option than embeddables, which have issues with how null values are treated

---
transition: slide-up
---

# Repository pattern

As repositories in the domain layer are just interfaces, they can’t define actual behavior

<!--
- Imagine you need to verify that a ISBN code is unique in your system, you just cannot do that in an interface
-->

---
transition: slide-left
---

`Persister` services do this, and they should be the entry points for all persistence logic. Repositories should not be used directly anywhere else.

```php
class BookPersister {
    /**
     * @throws BookCouldNotBeSavedException
     * @throws BookWithGivenIsbnCouldNotBeSavedBecauseOneAlreadyExistsException
     */
    public function save(Book $book): void {
        if ($this->bookRepository->isIsbnAlreadyTaken($book)) {
            throw BookWithGivenIsbnCouldNotBeSavedBecauseOneAlreadyExistsException::create($book);
        }
        
        $this->bookRepository->save($book);
    }
}
```

---
transition: slide-up
---

# Events, commands and queries

They are value objects that must be forwarded to the handler is responsible for processing them.

A _bus_ is the service that does this.

---
transition: slide-up
---

## Bus

- A bus is something that matches an object expressing an intention to a service that does something with it
- Commands and queries must match exactly with one handler
- Events can have zero, one or many handlers.

<!--
You can build bus implementations on top of `symfony/messenger`. Even though `symfony/event-dispatcher` might seem an obvious candidate as an event bus, `symfony/messenger` allows for more flexibility. The rule of thumb is to use a custom event bus for your custom events, and the Symfony event dispatcher for framework events only.
-->

---
layout: section
transition: slide-up
---

```php
interface CommandBusInterface {
    /**
     * @throws Exception
     */
    public function dispatch(CommandInterface $command): void;
}

class SymfonyMessengerCommandBus implements CommandBusInterface {
    public function __construct(private MessageBusInterface $symfonyMessengerMessageBus) {
        $this->symfonyMessengerMessageBus = $symfonyMessengerMessageBus;
    }

    public function dispatch(CommandInterface $command): void {
        try {
            $this->symfonyMessengerMessageBus->dispatch($command);
        } catch (HandlerFailedException $exception) {
            throw $exception->getPrevious() ?? $exception;
        }
    }
}
```

---
layout: section
transition: slide-up
---

```php
class AddBookController {
    public function __invoke(
        CommandBusInterface $commandBus,
        QueryBusInterface $queryBus,
        Request $request
    ) {
        // Map your request to $bookId, $isbn and $title values
        
        $command = new AddBookCommand($bookId, $isbn, $title);
        $commandBus->dispatch($command);
        
        $query = new GetBookQuery($bookId);
        $result = $queryBus->dispatch($query);
        
        // Do something with $result->book
    }
}
```

---
transition: slide-up
---

## Recording and dispatching events

As they are part of the domain layer, the natural place to record them is the aggregate root, especially if you use it as entry point for all your actions

An easy way of doing this is to create a specific trait and including it in the aggregate root.

<!--
- Commands and queries are dispatched by the UI layer, but who is dispatching events, and how?
- If we go back to the definitions of events, they are something that happens in your domain
- This means that they must be handled in your domain
- The best place to do that is the aggregate root, because it's the entry point for all write interactions
-->

---
layout: section
transition: slide-up
---

```php
trait EventRecorderTrait {
    /** @var list<EventInterface> */
    private array $recordedEvents = [];

    /** @return list<EventInterface> */
    public function releaseEvents(): array {
        $events = $this->recordedEvents;
        $this->recordedEvents = [];

        return $events;
    }

    protected function recordThat(EventInterface $event): void {
        $this->recordedEvents[] = $event;
    }
}

class Book {
    use EventRecorderTrait;
    public function __construct() {
        // ...
        $this->recordThat(new BookWasAdded($this->id));
    }
}
```

<!--
- We saw who is responsible for creating events, but what about dispatching them?
- We don't have access to the event bus in the entity!
-->

---
transition: slide-up
---

## Dispatching events

```php
class BookPersister {
    public function __construct(
        private readonly EventBusInterface $eventBus,
    ) {}

    public function save(Book $book): void {
        // ...
        
        foreach ($book->releaseEvents() as $event) {
            $this->eventBus->dispatch($event);
        }

        // Not really necessary, but often useful
        $this->eventBus->dispatch(new BookWasSaved($book->id));
    }
}
```

<!--
- The most pragmatic solution is to dispatch events in a persister, right after save.
- This aligns nicely with using the aggregate root both as the only entity who is handled by the persister and the one who is in charge of recording events.
- Every event is stored in the aggregate root, so the persister can dispatch every event
-->

---
transition: slide-left
---

## What should go in an event?

Best practice would be to store *everything* about the event that has happened, but almost always storing the ID is good enough

<!--
- You can fetch the aggregate later in the event handler
-->

---
layout: section
transition: slide-up
---

# Error handling

My golden rules:

- If a method cannot do what it says it does, it should throw an exception
- Avoid implicit conventions like "`findX` may return null but `getX` can't"
- Always use custom exceptions and extend the base `Exception` class
- Exception names and messages should be as detailed as possible
- Always declare exceptions using `@throws` annotations

<!--
You need to have a clear way for the consumer and the implementation to talk to each other
-->

---
transition: slide-up
---

```php
interface BookRepositoryInterface {
    /**
     * @throws BookCouldNotBeSavedException
     */
    public function save(Book $book): Book;
}

class BookCouldNotBeSavedException extends Exception {
    public static function create(Book $book, Throwable $previous = null): self {
        return new self(
            sprintf('Book with ID "%s" could not be saved.', $book->id->toString()),
            0,
            $previous,
        );
    }
}

class DoctrineBookRepository implements BookRepository {
    public function save(Book $book): void {
        try {
            $this->entityManager->persist($book);
            $this->entityManager->flush();
        } catch (Exception $exception) {
            throw BookCouldNotBeSavedException::create($book, $exception);
        }
    }
}
```

---
transition: slide-left
---

## Bubbling up exceptions

```php
class ShowBookController {
    public function __invoke(QueryBusInterface $queryBus, Request $request) {
        // Map your request to $bookId
        $query = new GetBookQuery($bookId);
        try {
            $result = $queryBus->dispatch($query);
        } catch (BookWithGivenIdCouldNotBeFoundException $exception) {
            // Display a 404 page
        }
        
        // Do something with $result->book
    }
}
```

<!--
- Unless you have a very good reason to catch them, let exception bubble up outside command and query handlers: catch them in the controller and decide how to handle them for the user there
-->

---
transition: fade-out
---

# Shared components

- Generic value objects, like `Email` or `PhoneNumber`
- Interfaces for commands, queries and events
- General-purpose utilities like services for handling pagination
 
Golden rules:
- Other bounded contexts *can* access this, but this *cannot* access others
- Treat it as you could make an external library out of it and reuse it in any of your projects, so it can't contain anything specific to the current one

<!--
Sooner or later, you will have the need to reuse components in different subdomains: you can create a fake bounded context, call it something like `Shared` or `Common`, and use it to store
-->

---
layout: center
transition: slide-up
---

# Testing

---
transition: fade-out
---

## Behavior-Driven Development

> BDD is designed to test an application’s behavior from the end user’s standpoint, whereas TDD is focused on testing smaller pieces of functionality in isolation

Experiment with BDD-oriented testing tools, like PHPSpec (unit) or Behat (functional)

[Sylius](https://github.com/Sylius/Sylius) uses both, check out their code!

<!--
- PHPUnit is the standard and we all love it, but explore other tools and ideas
- Behavior-driven development (BDD) is an "extension" of test-driven development (TDD) with an eye towards DDD
-->

---
layout: center
transition: slide-up
---

# Static analysis

---
transition: slide-up
---

## More precise way of documenting your domain

```php
/** @var positive-int $age */
$age = 1;

/** @var non-empty-string $title */
$title = 'Some title';

/** @var list<Book> $books */
$books === array_values($books); // Lists are homogenous arrays with sequential integer keys starting at 0.
```

---
transition: fade-out
---

## More than just null checks

- Define custom types with `@psalm-type`
- Define conditional assertions with `@psalm-assert-if-true`
- Limit usage with `@psalm-internal My\Namespace` to prevent leaking into other layers
- Check for `@throws` annotations so you can be sure exceptions are always annotated correctly

---
layout: section
transition: slide-left
---

# What did we learn today?

- Maintainable software development is complex
- I wanted to give you some basic ideas about how you might get started

<!--
- This is not "all or nothing": if you wanted to start somewhere, I would recommend with creating commands and queries, because they get you in the habit of naming things
-->

---
layout: section
transition: fade-out
---

# What's left?

Well, a lot! We didn't talk about a ton of things
- How to deal with assets?
- Where should authentication and authorization live?
- There are plenty of "side discussions" to be had, like "who are your exceptions for?" or "what does null mean in your domain?"


---
layout: center
---

# Thank you!

If you have questions now or later, feel free to stop me and say hi 👋

<br>
<img src="images/reverse.svg" style="width: 30px" />

https://tech.reverse.hr
