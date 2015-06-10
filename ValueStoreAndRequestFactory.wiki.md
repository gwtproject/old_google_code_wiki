# Introduction

I've just posted a wave with a sketch of how I think we should deal with two related issues: data binding (ValueStore), and manipulating JPA-style server side entities from a gwt app with a minimum of DRY (RequestFactory).

Since Wave access isn't yet universal, I've used ExportyBot to publish a (kind of ugly) static version of it. I'm hoping that people who can't edit in wave will add their two cents on the comments on this wiki.

[Slice of life with ValueStore and RequestFactory](https://wave.google.com/wave/#restored:wave:googlewave.com!w%252B3IMMyKcVB) ([static](http://exporty-bot.appspot.com/export?waveId=googlewave.com!w%252B3IMMyKcVB))

Apologies in advance: this isn't yet a proper design document, it's more like a core dump.

rjrjr

# Moving Parts

Here's an outline of some of the classes and services presumed by the sketch.

JPA-aware script runs across services interfaces and finder methods. It creates

  * Shared
    * Id and Property classes for every entity (everything with an @Id)
    * Builder style Request classes for every service interface (including static finder methods on entity classes)
      * In addition to rpc-friendly default constructors, request classes have constructors for client side use which accept valuebox
  * Client
    * A GIN style RequestFactory interface with getters for every shared Request class

RequestFactoryGenerator

  * Generates an implementation for the RequestFactory interface the script made
  * Defines and generates a ValueStore as a side effect, and provides access to the canonical instance.
  * RequestFactory depends upon ValueStore, but ValueStore can be used without RequestFactory

ValueStoreGenerator,

  * GWT.create(ValueBox.class) simply presumes control of every subtype of Id.
  * GWT.create(CustomValueBox.class) looks for @Types() annotation which itemizes classes that can be managed, can include both Ids and beans. @AllIds() to claim all implementors of that interface
  * Adds validation rules to generated ValueBox based on JSR 303 annotations

ValueStore

  * allows subscriptions to
    * whole classes of Id,
    * specific properties on specific values of Id (for HasValue subscribers)
    * subranges of list properties on specific ids (for HasValueList subscribers)
  * Allows validation contraints to be declared per property, per id/bean
  * spawns DeltaValueStore for uncommitted edits
    * sends subscribers errors related to these (for HasErrors subscribers)
    * Does not require use of RequestFactory

DataBinder generator

  * generated object performs ValueBox subscribe / unsubscribe operations
  * UIObjects implement HasValue and HasValueList, and these are the ValueBox subscriber interfaces

ReflectiveDataBinder and MockUiBinder

  * for use in JRE tests
  * provide GWT.create() bridge s.t. tests are not required to pass in DB and UiB instances

RequestFactoryService (and servlet)

  * GWT RPC
    * to start with for ease of implementation
    * No reason it couldn't be JSON / PB based for restricted implementations (no serializing custom value objects, anything else?)
  * Sends Request objects and DeltaValueStore across the wire
  * Besides the generated Request objects per service and per entity, there is a standard CRUD service.