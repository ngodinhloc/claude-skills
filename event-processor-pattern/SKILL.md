---
name: event-processor-pattern
description: Use when building or extending a message/event-consuming module in a NestJS (or similar DI-based) backend — e.g. adding a new event type to a queue consumer, wiring up a new RabbitMQ/Kafka/SQS message handler, or reviewing/refactoring event-handling code for consistency. Based on the event-processing module in model-arena/backend/src/event. Trigger on requests like "add a handler for event X", "wire up a new queue consumer", "how should this event module be structured".
---

# Event Processor Pattern

A layered structure for consuming messages off a broker (RabbitMQ/Kafka/etc.) and routing them to per-event handlers, keeping transport, dispatch, and business logic in separate, independently testable layers. Reference implementation: `backend/src/event/` in model-arena.

## Layer breakdown

```
event/
  contracts/
    event.interfaces.ts     # event name constants, exchange/queue constants, EventHandler contract
  configs/
    event.config.ts         # EVENT_REGISTRY: maps event name -> handler instance
  handlers/
    <event-name>.handler.ts # one class per event type, implements EventHandler
  services/
    message.processor.ts    # transport-agnostic dispatcher: raw payload -> registry lookup -> handler.handle()
    <broker>.consumer.ts    # transport-specific: subscribes to broker, feeds MessageProcessor
  event.module.ts           # wires providers, registers EVENT_REGISTRY as a DI-injected factory
```

Data flows one direction: **broker → consumer → processor → registry lookup → handler → domain logic**. Each layer only knows about the layer directly below it; nothing but the consumer knows about the broker, and nothing but the handler knows about domain repositories/entities.

## The pieces

**1. Contracts (`contracts/event.interfaces.ts`)** — the shared vocabulary. String constants for exchange names, queue names, and event names (avoid magic strings elsewhere), plus the `EventHandler` interface every handler implements:

```ts
export const EXCHANGE_SCORES = 'model_arena.scores';
export const EVENT_SCORES_RESPONDED = 'model_arena.scores.responded';
export const BACKEND_QUEUE = 'backend.queue';

export interface EventHandler {
  handle(event: ExperimentEvent): Promise<void>;
}
```

**2. Registry (`configs/event.config.ts`)** — a factory function, built via a DI `useFactory` provider, that maps event-name string to a concrete handler instance. This is the single place that knows "which handler owns which event":

```ts
export const EVENT_REGISTRY = 'EVENT_REGISTRY';

export function createEventRegistry(
  scoreRespondedHandler: ScoreRespondedHandler,
): Record<string, EventHandler> {
  return {
    [EVENT_SCORES_RESPONDED]: scoreRespondedHandler,
  };
}
```

**3. Handlers (`handlers/*.handler.ts`)** — one `@Injectable()` class per event type, implementing `EventHandler.handle()`. This is where domain/business logic lives: repository lookups, cache reads, status transitions, persistence. Handlers guard on missing/malformed data by logging a warning and returning early rather than throwing — a bad message shouldn't crash the consumer.

**4. Processor (`services/message.processor.ts`)** — transport-agnostic dispatcher. Takes a raw untyped payload, extracts `eventName`, looks it up in the injected `EVENT_REGISTRY`, normalizes/builds the typed event object, and calls `handler.handle()`. Logs and returns (doesn't throw) when `eventName` is missing or unrecognized — unknown events are dropped, not fatal.

**5. Consumer (`services/<broker>.consumer.ts`)** — the only layer that touches the broker client. Implements `OnModuleInit`, subscribes to an exchange/queue/routing-key on startup, and forwards each raw payload to `messageProcessor.process(payload)`. Contains no business logic and no registry knowledge.

**6. Module (`event.module.ts`)** — registers handlers, the processor, and the consumer as providers; registers `EVENT_REGISTRY` as a `useFactory` provider with handlers listed in `inject`; exports what other modules need (typically the registry token, handlers, processor, consumer).

## Adding a new event type

1. Add the event-name constant (and exchange/queue constants if new) to `contracts/event.interfaces.ts`.
2. Create `handlers/<event-name>.handler.ts`: an `@Injectable()` class implementing `EventHandler.handle(event)`, injecting whatever repositories/services it needs.
3. Register the handler in `event.config.ts`: add it to `createEventRegistry`'s parameters and add the `[EVENT_NAME]: handler` entry to the returned map.
4. Add the handler to the module's `providers` array and to `EVENT_REGISTRY`'s `inject` array in `event.module.ts`.
5. If it's a new queue/routing-key (not just a new event on an existing subscription), add a `subscribe(...)` call in the consumer, or bind it in the existing subscription if it shares a queue.

No existing handler, the processor, or the consumer needs to change for a same-queue new event — only the registry and the module wiring. That's the Open/Closed payoff of this structure (see [[solid-principles]]).

## Conventions to preserve

- **Fail soft, log loud.** Missing `eventName`, unknown event, missing required fields on the event body: log a `warn` with context and return — never throw out of the processor or a handler for a malformed message. Reserve throwing for the consumer's ack/nack boundary (a thrown error there should mean "retry/dead-letter this message," which the broker client handles via nack).
- **Handlers stay narrow.** One handler per event name. If a handler starts branching on sub-types within one event, consider whether that's actually two events.
- **No business logic in the consumer or processor.** They stay generic across every event type; only handlers know about entities, repositories, or domain state transitions.
- **DI token for the registry, not a plain import.** The registry is provided via a string/symbol token (`EVENT_REGISTRY`) and `useFactory`, so it can be swapped/mocked in tests and so handler wiring is explicit and visible in one file.
- **Constants over string literals.** Event names, exchange names, and queue names are exported constants from `contracts/`, referenced everywhere else — never re-typed as string literals in handlers or the consumer.
