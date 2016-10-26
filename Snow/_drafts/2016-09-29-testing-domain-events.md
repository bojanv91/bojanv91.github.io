http://www.marccostello.com/using-domain-events/
https://lostechies.com/jimmybogard/2010/04/08/strengthening-your-domain-domain-events/
http://udidahan.com/2009/06/14/domain-events-salvation/
---
layout: post
title: Testing domain events
---

Domain event is important concept in Domain-Driven-Design systems. It helps us decouple our domain models from application/infrastructure services.

As [defined by Martin Fowler](http://martinfowler.com/eaaDev/DomainEvent.html), a domain event "*captures the memory of something interesting which affects the domain*".

There are many different ways how one can structure their domain events. I like [Jimmy Bogard's approach](https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/) by separating the two concerns of **raising** and **dispatching**. 

## General flow

The general flow is:

- **Domain model raises events** - e.g. ``UserAccount`` raises ``PasswordResetedEvent`` when someone calls ``UserAccount.ResetPassword()`` command.
- **Infrastructure code subscribes to the raised events and dispatches them to appropriate handlers (in same thread/transaction or pass them to bus/message queue)** - e.g. For each raised event in transaction, call ``mediator.Publish(@event)`` for in-transaction handlers or/and ``messageBus.Send(@event)`` for sending message via queues (RabbitMQ?) for background processing.
- **Event handlers handle the dispatched events** - e.g. Event handlers accept the dispatched events and execute appropriate logic in reactive fashion.

## Given use-case

As an example use-case for the purpose of this writing I will use the user account password recovery feature. 

Users can ask the system to reset their password.
When they do that, the system should do three things:

- generate verification code
- record audit log of the password reset request
- send the verification code via SMS to user's mobile phone

## The Model

    public interface IDomainEventsRaisable
    {
    //
    }

Domain events are always named in past tense, as they represent something that has already happened.

## Raising an event

## Subscribe to an event (in same thread and transaction)

## Subscribe to an event (elsewhere in message handler)

## Testing if domain events has been raised

And the current test code:


Here is a nice extension that I made which can be used for simpler code in the testing part

## Summary

Many things in DDD are related directly to good OO programming. Once you understand the basic concepts, you can easily combine them together to craft  encapsulated and modular domain models. Domain event is just another *tool* which can help you see your system from another perspective and solve the common set of related problems - the need to use application/infrastructure services in domain models.

You can find the complete example on following GitHub repository.


Thanks for stopping by and **Always Be Coding**!