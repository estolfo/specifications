.. role:: javascript(code)
  :language: javascript

==================
Command Monitoring
==================

:Spec: 200
:Title: Command Monitoring
:Authors: Durran Jordan
:Status: Approved
:Type: Standards
:Minimum Server Version: 2.4
:Last Modified: June 25, 2015

.. contents::

--------

Specification
=============

The performance monitoring specification defines a set of behaviour in the drivers for providing runtime information about commands to any 3rd party APM library as well internal driver use, such as logging.

-----------
Definitions
-----------

META
----

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.


Terms
-----

Document
  The term ``Document`` refers to the implementation in the driver's language of a BSON document.

GLE
  The term "GLE" refers to a ``getLastError`` command.


--------
Guidance
--------

Documentation
-------------

The documentation provided in code below is merely for driver authors and SHOULD NOT be taken as required documentation for the driver.


Operations
----------

All drivers MUST implement the API. Implementation details are noted in the comments when a specific implementation is required. Within each API, all methods are REQUIRED unless noted otherwise in the comments.


Operation Parameters
--------------------

All drivers MUST include the specified parameters in each operation, with the exception of the options parameter which is OPTIONAL.


Naming
------

All drivers MUST name operations, parameters and topic names as defined in the following sections. Exceptions to this rule are noted in the appropriate section. Class and interface names may vary according to the driver and language best practices.


Publishing & Subscribing
------------------------

The driver SHOULD publish events in a manner that is standard to the driver's language publish/subscribe patterns and is not strictly mandated in this specification.


Guarantees
----------

The driver MUST guarantee that every ``CommandStartedEvent`` has either a correlating ``CommandSucceededEvent`` or ``CommandFailedEvent``.


Unacknowledged/Acknowledged Writes
----------------------------------

For server versions that do not support write commands, the driver MUST treat an acknowledged write as a single command event, where the GLE command is ignored as a started event and the response to the GLE is treated as the reply in the ``CommandSucceededEvent``. Unacknowledged writes must provide a ``CommandSucceededEvent`` with a ``null`` reply.


Error Handling
--------------

If an exception occurs while sending the operation to the server, the driver MUST generate a ``CommandFailedEvent`` with the exception or message and re-raise the exception.

If the response from the server completes but contains an error, the driver MUST generate a ``CommandFailedEvent`` with the error or message from the server instead of a ``CommandSucceededEvent``. This includes ``{ ok: 0 }`` command responses and replies with query_failure flags.

Upconversion
------------

All legacy operations MUST be converted to their equivalent commands in the 3.2 server in the event's
``command`` and ``reply`` fields. This includes OP_INSERT, OP_DELETE, OP_UPDATE, OP_QUERY, OP_GETMORE and
OP_KILLCURSORS. Upconversion expectations are provided in the tests.

---
API
---

.. code:: typescript

  interface CommandStartedEvent {

    /**
     * Returns the command.
     */
    command: Document;

    /**
     * Returns the database name.
     */
    databaseName: String;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the driver generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;
  }

  interface CommandSucceededEvent {

    /**
     * Returns the execution time of the event in the highest possible resolution for the platform.
     * The calculated value MUST be the time to send the message and receive the reply from the server,
     * including BSON serialization and deserialization. The name can imply the units in which the
     * value is returned, i.e. durationMS, durationNanos.
     */
    duration: Int64;

    /**
     * Returns the command reply.
     */
    reply: Document;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the driver generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;
  }

  interface CommandFailedEvent {

    /**
     * Returns the execution time of the event in the highest possible resolution for the platform.
     * The calculated value MUST be the time to send the message and receive the reply from the server,
     * including BSON serialization and deserialization. The name can imply the units in which the
     * value is returned, i.e. durationMS, durationNanos.
     */
    duration: Int64;

    /**
     * Returns the command name.
     */
    commandName: String;

    /**
     * Returns the failure. Based on the language, this SHOULD be a message string, exception
     * object, or error document.
     */
    failure: String,Exception,Document;

    /**
     * Returns the client generated request id.
     */
    requestId: Int64;

    /**
     * Returns the driver generated operation id. This is used to link events together such
     * as bulk write operations. OPTIONAL.
     */
    operationId: Int64;

    /**
     * Returns the connection id for the command. For languages that do not have this,
     * this MUST return the driver equivalent which MUST include the server address and port.
     * The name of this field is flexible to match the object that is returned from the driver.
     */
    connectionId: ConnectionId;
  }


--------
Examples
--------

A Ruby subscriber to a query series and how it could handle it with respect to logging.

Ruby:

.. code:: ruby

  class LoggingSubscriber

    def started(event)
      Logger.info("COMMAND.#{event.command_name} #{event.connection} STARTED: #{event.command_args.inspect}")
    end

    def succeeded(event)
      Logger.info("COMMAND.#{event.command_name} #{event.connection} COMPLETED: #{event.command_reply.inspect} (#{event.duration}s)")
    end

    def failed(event)
      Logger.info("COMMAND.#{event.command_name} #{event.connection} FAILED: #{event.message.inspect} (#{event.duration}s)")
    end
  end

  subscriber = LoggingSubscriber.new
  Mongo::Monitoring::Global.subscribe(Mongo::Monitoring::COMMAND, subscriber)

  # When the subscriber handles the events the log could show:
  # COMMAND.query 127.0.0.1:27017 STARTED: { $query: { name: 'testing' }}
  # COMMAND.query 127.0.0.1:27017 COMPLETED: { number_returned: 50 } (0.050s)

-------
Testing
-------

Tests are provided in YML and JSON format to assert proper upconversion of commands.

Requirements
------------

Some assertions cannot be represented in YML or JSON format but MUST be tested. They are
as follows:

For ``find`` command tests, the tests MUST also assert when a ``cursor_id`` is not provided in the
expectation that the ``cursor_id`` in the reply of the ``CommandSucceededEvent`` is equal to:

- The value of the ``getMore`` field in the getmore's ``CommandStartedEvent`` command.
- The value of the ``id`` field in the ``cursor`` subdocument of the getmore's ``CommandSucceededEvent`` reply.
- The value exists in the ``cursors`` array in the killcursor's ``CommandStartedEvent``.
- The value is always present and greater than 0.

Example:

.. code:: yml

  -
    description: "A successful find event with a getmore and killcursors"
    operation:
      name: "find"
      arguments:
        filter: { _id: { $gte: 1 }}
        sort: { _id: 1 }
        batchSize: 3
        limit: 4
    expectations:
      -
        command_started_event:
          command:
            find: *collection_name
            filter: { _id: { $gte : 1 }}
            sort: { _id: 1 }
            limit: 3
          command_name: "find"
          database_name: *database_name
      -
        command_succeeded_event:
          reply:
            ok: 1
            cursor:
              id: 999999
              firstBatch:
                - { _id: 1, x: 11 }
                - { _id: 2, x: 22 }
                - { _id: 3, x: 33 }
          command_name: "find"
          database_name: *database_name
      -
        command_started_event:
          command:
            getMore: 999999
            collection: *collection_name
            batchSize: 1
          command_name: "getMore"
          database_name: *database_name
      -
        command_succeeded_event:
          reply:
            ok: 1
            cursor:
              id: 999999
              nextBatch:
                - { _id: 4, x: 44 }
          command_name: "getMore"
          database_name: *database_name
      -
        command_started_event:
          command:
            killCursors: *collection_name
            cursors:
              - 999999
          command_name: "killCursors"
          database_name: *database_name
      -
        command_succeeded_event:
          reply:
            ok: 1
          command_name: "killCursors"
          database_name: *database_name
