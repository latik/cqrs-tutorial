## CQRS tutorial

Code to discover the processes surrounding CQRS and Event Sourced systems.

Complete documentation is available at [https://gistlog.co/scazz/e9e0f83d09fd9c3396b9](https://gistlog.co/scazz/e9e0f83d09fd9c3396b9)

## Testing

Tests can be run with phpunit

### Contents ###
* [Introduction](#introduction)
* [Project Setup and first test](#project-setup-and-first-test)
* [Follow the test](#follow-the-test)
* [Booking a lesson using events - an overview](#booking-a-lesson-using-events---an-overview)
* [Extracting CQRS components and refactoring the write model](#extracting-cqrs-components-and-refactoring-the-write-model)
* [Persisting events to an event store](#persisting-events-to-an-event-store)
* [Listing to events and updating read models](#listing-to-events-and-updating-read-models)
* [Improving the read model with relationships](#improving-the-read-model-with-relationships)
* [Loading lessons](#loading-lessons)
* [Protecting against invariants](#protecting-against-invariants)
* [Generating UUIDs](#generating-uuids)
* ["Enforcing" Read Model immutability](#enforcing-read-model-immutability)

### Introduction ###

This post runs through the basics of creating an event sourced/CQRS system with PHP and Laravel. It assumes familiarity
with the command bus design pattern and events (publishing events to an array of listeners). Laracasts have you covered
if you need to brush up! It also assumes you have some familiarity with the concept of CQRS. If not, I highly recommend
two talks: [Practical Event Sourcing by Mathias Verraes](http://verraes.net/2014/03/practical-event-sourcing) and [CQRS
and Event sourcing by Greg Young](https://www.youtube.com/watch?v=JHGkaShoyNs).

Please don't use this code for your projects! It's a learning platform to uncover the ideas behind CQRS. It's not
robust, it's poorly tested and I'm rarely coding to interfaces which will make it harder to swap parts out. A much
better example of a CQRS package you can use is [Qandidate Lab's Broadway](https://github.com/qandidate-labs/broadway).
It's decoupled, clean code but some of the abstractions make it hard to follow if you've never seen an event sourced
system.

Finally, this code is coupled to Laravel's Events and Command Bus. I wanted to see what the code would look like with
Laravel (it's my framework of choice for smaller agency projects), but looking back I should have created my own
implementations. I hope people will be able to follow along, even if they don't use a framework.

The code is on github: [https://github.com/scazz/cqrs-tutorial.git](https://github.com/scazz/cqrs-tutorial.git), and the
tutorial takes you through the code, commit by commit. Hopefully this will make it easy to follow along!

We will be building the start of a booking system for a surf school. It will allow clients to book onto lessons. Our
domain rules for this process are:
- A lesson must have at least one client
- No more than 3 clients per lesson

One of the exciting things about CQRS and Event Sourced systems is creating read models specific to each metric you
require from the system. You can find examples projecting read models into ElasticSearch and Greg Young has a DSL for
Complex Event Processing built into his event store. But, in an effort to reduce the learning curve, our read projection
will be a standard SQL database you would use with Eloquent. We will end up with one table for lessons, and one for
clients.

Event Sourcing also gives you the ability to process the events off-line. This post will keep as close to "traditional"
development models as possible (again to reduce complexity) and our read projections will be updated in real time, as
soon as events are persisted into the event store.

### Project Set up and first test ###

Start a new laravel 5 project

```
$> laravel new cqrs-tutorial
```
And the first thing we need is a test. We're just going to stick with an integration test to assert that booking a
lesson for a client results in that lesson being created in our Eloquent model.

```php
<?php

use Illuminate\Foundation\Bus\DispatchesCommands;

class CQRSTest extends TestCase {

    use DispatchesCommands;

    /**
     * Assert the BookLesson Command creates a lesson in our read projection
     * @return void
     */
     public function testFiringEventUpdatesReadModel()
     {
         $testLessonId = '123e4567-e89b-12d3-a456-426655440000';
         $clientName = "George";
         $lessonId = new LessonId($testLessonId);

         $command = new BookLesson($lessonId, $clientName);
         $this->dispatch($command);

        $this->assertNotNull(Lesson::find($testLessonId));
        $this->assertEquals( Lesson::find($testLessonId)->clientName, $clientName );
     }
}
```

We provide the ID for our new lesson up front, create a command to book a new lesson and tell Laravel to dispatch the
command. We want a new entry to be created in the lessons table, which we can read with an Eloquent model. We will be
needing a database, so fill in your .env file as required.

Each event logged to our event store is attached to an Aggregate Root (which we will just call an Entity - the
abstraction for learning purposes just adds to the confusion). This ID is a universally unique ID. The event store
doesn't care if the event is to be applied to a Lesson or a Client, it just knows it's attached to an ID.

### Follow the test ###

Being driven by our test errors, we can create our missing classes. First we create a LessonId class, followed by the
BookLesson command (don't worry about the handle method yet, just keep following the test). The Lesson class is a read
model outside of the Lesson namespace - it will only ever be a read model - no domain logic will reside in here.
Finally, we have to create a migration for the lessons table.

I'm using an assertion library to keep the code clean. You can grab it with:

```
composer require beberlei/assert
```

```php
<?php
namespace App\School\Lesson;

use Assert\Assertion as Assert;

class LessonId {

    private $lessonId;

    public function __construct($lessonId) {
        Assert::string( $lessonId );
        Assert::uuid( $lessonId );

        $this->lessonId = $lessonId;
    }

    /**
    * @return string
    */
    public function __toString()
    {
        return $this->lessonId;
    }
}
```

```php
<?php
namespace App\School\Lesson\Commands;

use App\Commands\Command;
use App\School\Lesson\LessonId;
use Illuminate\Contracts\Bus\SelfHandling;

class BookLesson extends Command implements SelfHandling {

    /** @var LessonId  */
    private $lessonId;

    public function __construct(LessonId $lessonId)
    {
        $this->lessonId = $lessonId;
    }

    /**
    * @return LessonId
    */
    public function getLessonId()
    {
        return $this->lessonId;
    }

    public function handle()
    {
        //TODO: handle dispatched command
    }

 }
```

```php
<?php
namespace App\School\ReadModels;

use Illuminate\Database\Eloquent\Model;

class Lesson extends Model {

}
```

```php
<?php

use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateLessonTable extends Migration {

    /**
    * Run the migrations.
    *
    * @return void
    */
    public function up()
    {
        Schema::create('lessons', function(Blueprint $table) {
            $table->string('id'); // uuid
        });
    }

    /**
    * Reverse the migrations.
    *
    * @return void
    */
    public function down()
    {
        Schema::drop('lesson');
    }

}
```

### Booking a lesson using events - an overview ###

Let's take a look at the process this command should initiate:

1. Validation: imperative commands can fail, events are past tense and must not.
2. Generate a new LessonWasBooked event
3. Update the Lesson's state. (The write model needs to be aware of the model's state so it can perform validation)
4. Add this event to a stream of uncommitted events stored on the lesson write model
5. Persist the uncommitted events stream to the event store
6. Fire the LessonWasBooked event globally to inform all read projectors they should update the lessons table


So first, we need to create a write model for the lesson. We use a static factory method,
Lesson::bookClientOntoNewLesson(). This generates a new LessonWasOpened event, applies the event to itself (just sets
it's ID), adds the new event to the uncommitted events list in the form of a DomainEventMessage (the event, plus some
meta data which we will use when persisting to the event store).

It repeats the process to add a client to the event. When applying the ClientWasBookedOntoLesson event, the write model
doesn't keep a track of the client names, just how many clients are booked in. The write model doesn't need to care
about client names to protect invariants.

The applyLessonWasOpened and applyClientWasBookedOntoLesson methods might seem a little weird at the moment. They will
be used later when we need to replay old events to build up the write model's state,

It's a tricky concept to explain so here is some code to help outline the process. Later, we will extract out the code
handling uncommittedEvents and domain event message creation.

```php
<?php

namespace App\School\Lesson;

class Lesson {
    private $lessonId;
    private $numberOfClients;
    private $uncommittedEvents = [];

    public static function bookClientOntoNewLesson(LessonId $lessonId, $clientName) {
        $lesson = new Lesson();
        $lesson->openLesson( $lessonId );
        $lesson->bookClient( $clientName );
        return $lesson;
    }

    private function openLesson( LessonId $lessonId ) {
        /* here we would check any invariants - but we don't have any to protect, so we can just generate the events */
        $event = new LessonWasOpened( $lessonId);
        $this->applyLessonWasOpened($event);
        $this->uncommittedEvents[] = DomainEventMessage::recordNow( $this->lessonId, $event );
    }

    private function applyLessonWasOpened( $event ) {
        $this->lessonId = $event->lessonId;
        $this->numberOfClients = 0;
    }

    public function bookClient( $clientName ) {
        if ($this->numberOfClients >= 3) {
            throw new Exception("Too many clients");
        }

        $event = new ClientBookedOntoLesson( $this->lessonId, $clientName);
        $this->applyClientBookedOntoLesson( $event );
        $this->uncommittedEvents[] = DomainEventMessage::recordNow( $this->lessonId, $event );
    }

    /*
    * Here, we only keep track of the number of clients -
    * this is the only thing the write model cares about
    *
    * If a domain rules was "no clients can have the same name",
    * we would need to keep a track of client names.
    */
    private function applyClientBookedOntoLesson( $event ) {
        $this->numberOfClients++;
    }
}
```

### Extracting CQRS components and refactoring the write model ###

We can extract the CQRS components out from our write model - the parts of the class dealing with uncommitted events. We
can also clean up the API for an event sourced entity by creating a protected function called apply() which takes an
event, calls the appropriate applyEventName() method and appends a new DomainEventMessage to the uncommitted events
list.

The extracted class is an implementation detail of CQRS, and doesn't contain any domain logic so we can create a new
namespace: App\CQRS:

```php
<?php
namespace App\CQRS;

abstract class EventSourcedEntity implements EventSourcedEntityInterface {

    private $uncommittedEvents = [];

    public function apply( $event ) {
        $this->handle( $event );
        $this->uncommittedEvents[] = DomainEventMessage::recordNow( $this->getEntityId(), $event );
    }

    public function getUncommittedDomainEvents() {
        return $this->uncommittedEvents;
    }

    private function handle( $event ) {
        $method_name = $this->getApplyMethodName($event);

        if (! method_exists($this, $method_name)) {
            return;
        }

        $this->$method_name($event);
    }

    private function getApplyMethodName($event) {
        $className = get_class($event);

        $classParts = explode('\\', $className);
        $methodName = end($classParts);

        return 'apply'. $methodName;
    }
}
```

To get the code to run, we need to add a DomainEventMessage class, which is just a simple DTO:

```php
<?php
namespace App\CQRS;

use DateTime;

class DomainEventMessage {

    private $id;
    private $event;
    /**
     * @var DateTime
     */
    private $dateTime;

    public function __construct($id, $event, DateTime $dateTime) {
        $this->id = $id;
        $this->event = $event;
        $this->dateTime = $dateTime;
    }

    public static function recordNow($id, $event ) {
        return new DomainEventMessage($id, $event, new DateTime());
    }

    /**
    * @return DateTime
    */
    public function getDateTime()
    {
        return $this->dateTime;
    }

    /**
    * @return mixed
    */
    public function getEvent()
    {
        return $this->event;
    }

    /**
    * @return mixed
    */
    public function getId()
    {
        return $this->id;
    }
}
```

### Persisting events to an event store ###

So, we have a system which generates events for every write and uses the events to recorded changes necessary to prevent
invariants. The next step is to persist these events to the EventStore.

First, we need an event store. Keeping things simple, we'll use an Eloquent model, just a simple SQL table with fields
for:
* UUID (so we know which entity to apply the event to)
* event_payload (a serialized message containing everything we need to rebuild the event)
* recordedAt - a timestamp so we know when the event occurred

If you check the code, you'll see I've created two commands to create and destory our event store table:
* php artisan eloquenteventstore:create (App\CQRS\EloquentEventStore\CreateEloquentEventStore)
* php artisan eloquenteventstore:drop (App\CQRS\EloquentEventStore\DropEloquentEventStore)

(not forgetting to add these to App\Console\Kernel.php so they're loaded)

There are two very good reasons not to use an SQL table as your event store: it's not an append-only model (events
should be immutable) and SQL is not a great temporal query language. We're coding to an interface, so it will be
straight forward to swap the event store in a later blog post.

We will use a repository to handle the saving of events. Whenever save() is called for a write model, we persist the
list of uncommittedEvents to the event store.

To store events, we need to a way to serialize/deserialize them. We can create a Serializer to handle this. We will need
meta data, such as the event class (eg App\School\Lesson\Events\LessonWasOpened) and the event payload (data needed to
reconstruct the event). All will then be JSON encoded and then written to our database, along with the entity UUID and
timestamp.

As soon each event is persisted, we will want our read models to update, so the repository will fire each event after
saving.

The Serializer will be responsible for recording the event class, whilst the event is responsible for the serializing of
its payload. A fully serialized event will look something like:

```
{
    class: "App\\School\\Lesson\\Events\\MyEvent",
    event: $event->serialize()
}
```
Since all events will need a serialize and deserialize method, we can create a SerializableEvent interface, and start
type hinting :)

Update our LessonWasOpened event:

```php
class LessonWasOpened implements SerializableEvent {

    public function serialize() {
        return ['lessonId'=> (string) $this->getLessonId()];
    }
}
```

Create the LessonRepository. We can refactor and extract generic CQRS stuff later.

```php
<?php
namespace App\School\Lesson;

use App\CQRS\DomainEventMessage;
use App\CQRS\EloquentEventStore\EloquentEventStoreRepository;
use App\CQRS\Serializer\EventSerializer;
use Event;

class LessonRepository {

    /* @TODO: Dependency Injection! */
    public function __construct() {
        $this->eventStoreRepository = new EloquentEventStoreRepository( new EventSerializer() );
    }

    public function save(Lesson $lesson) {
        /** @var DomainEventMessage $domainEventMessage */
        foreach( $lesson->getUncommittedDomainEvents() as $domainEventMessage ) {
            $this->eventStoreRepository->append(
            $domainEventMessage->getId(),
            $domainEventMessage->getEvent(),
            $domainEventMessage->getRecordedAt()
            );
            Event::fire($domainEventMessage->getEvent());
        }
    }
}
```

If you run the integration test again, then check the domain_events SQL table, you should see two events in the
database.

### Listing to events and updating read models ###

Our final step to get our test to pass: listening for the broadcasted events and updating the Lesson read model
projection.

The broadcasted Lesson events will be caught by a LessonProjector, which applies the neccessary changes to a
LessonProjection (just an Eloquent model of the lessons table):

```php
<?php
namespace App\School\Lesson\Projections;

use App\School\Lesson\Events\ClientBookedOntoLesson;
use App\School\Lesson\Events\LessonWasOpened;
use Illuminate\Events\Dispatcher;

class LessonProjector {

    public function applyLessonWasOpened( LessonWasOpened $event )
    {
        $lessonProjection = new LessonProjection();
        $lessonProjection->id = $event->getLessonId();
        $lessonProjection->save();
    }

    public function applyClientBookedOntoLesson( ClientBookedOntoLesson $event )
    {
        $lessonProjection = LessonProjection::find($event->getLessonId());
        $lessonProjection->clientName = $event->getClientName();
        $lessonProjection->save();
    }

    public function subscribe(Dispatcher $events)
    {
        $fullClassName = self::class;

        $events->listen(
            LessonWasOpened::class,
            $fullClassName.'@applyLessonWasOpened'
        );
        $events->listen(
            ClientBookedOntoLesson::class,
            $fullClassName.'@applyClientBookedOntoLesson'
        );
    }
}
```

```php
<?php
namespace App\School\Lesson\Projections;

use Illuminate\Database\Eloquent\Model;

class LessonProjection extends Model {
    public $timestamps = false;
    protected $table = "lessons";
}
```

And obviously, don't forget to register the event subscriber with laravel:
:

```php
public function boot(DispatcherContract $events)
{
    parent::boot($events);
    Event::subscribe( new LessonProjector() );
}
```

If you run the test, you'll see that there is an SQL error:

<blockquote><code class="hljs">Unknown column 'clientName' in 'field list'
</code></blockquote>

Once we create a migration to add clientName to the lessons table, our test should pass :). We have implemented basic
CQRS functionality: commands create events which are used to generate read models.


### Improving the read model with relationships ###

We've reach a mile stone, but we're not there yet! The read model only has support for one client (we specified 3 in
our domain rules).

The changes we make to the read model are fairly straight forward; we just create a Client projection model and
ClientProjector which catches the ClientBookedOntoLesson event.

First, let's update our test to reflect the changes we want to see in our read model:

```php
public function testFiringEventUpdatesReadModel()
{
    $testLessonId = '123e4567-e89b-12d3-a456-426655440000';
    $clientName = "George";
    $lessonId = new LessonId($testLessonId);

    $command = new BookLesson($lessonId, $clientName);
    $this->dispatch($command);

    $lesson = Lesson::find($testLessonId);
    $this->assertEquals( $lesson->id, $testLessonId );

    $client = $lesson->clients()->first();
    $this->assertEquals($client->name, $clientName);
}
```

The updated projector, read model and migrations are on github - it's basic laravel stuff. It highlights how easy it
is to change your read models. Everything up to the event store remains the same.

Smoke testing is something we get for free with an event sourced system - if we change our read model projector, we
have a listen of every event that has ever happened in our system. We replay these events using the new projector,
check for exceptions and compare the output with the old projections. If the system has been live for some time, we
will have a comprehensive list of events to test our projectors with.

### Loading lessons ###

Our write model currently has no way of loading the state of existing. If we want to add a second client to a lesson,
we could just create a second ClientWasAddedToLesson event but we wouldn't be able to protect against invariants.

To explain what I mean, let's write a second test simulating booking two clients into a lesson.

```php
public function testLoadingWriteModel()
{
    $testLessonId = '123e4567-e89b-12d3-a456-426655440001';
    $lessonId = new LessonId($testLessonId);
    $clientName_1 = "George";
    $clientName_2 = "Fred";

    $command = new BookLesson($lessonId, $clientName_1);
    $this->dispatch($command);

    $command = new BookClientOntoLesson($lessonId, $clientName_2);
    $this->dispatch($command);

    $lesson = Lesson::find($testLessonId);
    $this->assertClientCollectionContains($lesson->clients, $clientName_1);
    $this->assertClientCollectionContains($lesson->clients, $clientName_2);
}

private function assertClientCollectionContains(Collection $clients, $nameToFind)
{
    $attributeArray = [];
    foreach( $clients as $object ) {
        $attributeArray[] = $object->name;
    }

    $this->assertTrue( in_array($nameToFind, $attributeArray), "Could not find client named: ${nameToFind}" );
}
```

We need a way for our write model to "load" an entity that already has events applying to it in the event store. We
can achieve this by replaying every event which refers to the entity's UUID.

The basic process is:

1. Get all relevant event messages from the event store
2. For each message, recreate the appropriate event
3. Create a new entity write model and replay each event

At the moment, our tests throws exceptions so we start by creating the required BookClientOntoLesson command, using
the BookLesson command as a template. The handle method will look like:

```php
public function handle(LessonRepository $repository) {
    /** @var Lesson $lesson */
    $lesson = $repository->load($this->lessonId);
    $lesson->addClient($this->clientName);
    $repository->save($lesson);
}
```

Add the load event in the lesson repository:

```php
public function load(LessonId $id) {
    $events = $this->eventStoreRepository->load($id);
    $lesson = new Lesson();
    $lesson->initializeState($events);
    return $lesson;
}
```

The repository's load function returns an array recreated events. It does this by first finding the event messages in
the event store, then delegating to the Serializer to turn each event message into an event.

The Serializer created the messages from events, so we need to add a deserialize() method to reverse the process.
You'll remember the Serializer delegated to each event to manage the serialzing of event data (eg client name). We'll
do the same when reversing the process so our SerializableEvent interface should be updated with a deserialize()
method.

Let's take a look at the code to explain things better.

First the EventStoreRepository's load function:

```php
public function load($uuid) {
    $eventMessages = EloquentEventStoreModel::where('uuid', $uuid)->get();
    $events = [];

    foreach($eventMessages as $eventMessage) {
        /* We serialized our event into an event_payload, so we need to deserialize before returning */
        $events[] = $this->eventSerializer->deserialize(
            json_decode($eventMessage->event_payload)
        );
    }

    return $events;
}
```

With the corresponding deserialize function in the eventSerializer:

```php
public function deserialize( $serializedEvent ) {
    $eventClass = $serializedEvent->class;
    $eventPayload = $serializedEvent->payload;

    return $eventClass::deserialize($eventPayload);
}
```

Finally, the static factory deserialize() method in LessonWasOpened (we need to add this method to every event)

```php
public static function deserialize($data) {
    $lessonId = new LessonId($data->lessonId);
    return new self($lessonId);
}
```

Now we have an array of all previous events, we just replay them against our Entity write model to initialize state:

```php
public function initializeState($events) {
    foreach( $events as $event ) {
    $this->handle($event);
}
}
```

And rerun our test - we're back to Green!

### Protecting against invariants ###

We don't actually have a test here to ensure we're enforcing our domain rules, so let's write one:

```php
public function testMoreThan3ClientsCannotbeAddedToALesson() {
    $testLessonId = '123e4567-e89b-12d3-a456-426655440002';
    $lessonId = new LessonId($testLessonId);
    $this->dispatch( new BookLesson($lessonId, "bob") );
    $this->dispatch( new BookClientOntoLesson($lessonId, "george") );
    $this->dispatch( new BookClientOntoLesson($lessonId, "fred") );

    $this->setExpectedException( TooManyClientsAddedToLesson::class );
    $this->dispatch( new BookClientOntoLesson($lessonId, "emma") );
}
```

You'll notice that we only need a lessonId - this test re-initializing the lesson's state with each command.

### Generating UUIDs ###

At the moment, we're just passing in hand crafted UUIDs when really we want to generate these automatically. I'm going
to use the Ramsy\UUID package, so let's install that with composer:

```
composer require ramsey/uuid
```

And update our tests to use the new package:

```php
public function testEntityCreationWithUUIDGenerator() {
    $lessonId = new LessonId( (string) \Rhumsaa\Uuid\Uuid::uuid1() );
    $this->dispatch( new BookLesson($lessonId, "bob") );
    $this->assertInstanceOf( Lesson::class, Lesson::find( (string) $lessonId) );
}
```

### "Enforcing" Read Model immutability ###

Currently, a new developer to a project could look at the code, see App\School\ReadModels which contains a set of
Eloquent models and use these models to write changes to the lessons table. We can stop this by creating an
ImmutableModel class which extends the Eloquent Model class and overrides the save method:

```php
<?php
namespace App\CQRS\ReadModel;

use Illuminate\Database\Eloquent\Model;

class ImmutableModel extends  Model {

    public function save(array $options = [])
    {
        throw new SavingImmutableModel(
            "Generate events in order to change this model!"
        );
    }
}
```

Obviously a determined developer can still update the read model if they want to, but this will help prevent confusion.
