---
title: "Building a Tiny Pub/Sub System in TypeScript"
date: "2026-05-24"
tags: ["typescript", "pub-sub", "backend"]
---

So I was playing around with this little piece of code that I wrote, and it basically acts like a simple **pub/sub (publish–subscribe)** system. Nothing fancy, but it works and is actually kinda neat once you see how it’s wired up.

---

### The `Listener` type

```TypeScript
type Listener<T> = (payload: T) => void;
```

This just says: a listener is a function that takes some data (payload) and does something with it. That’s it.

---

### The `SimplePubSub` class

```TypeScript
class SimplePubSub {
  private topics = new Map<string, Set<Listener<any>>>();
```

Here I’ve got a `Map` called `topics`. Each topic is just a string, and the value is a `Set` of listener functions. Why a set? Because I don’t want duplicate listeners floating around.

---

### Publishing stuff

```TypeScript
publish<T>(topic: string, payload: T) {
  const listeners = this.topics.get(topic);
  if (!listeners) return;
  for (const l of listeners) l(payload);
}
```

When I call `publish`, it looks up the topic, finds all the listeners subscribed to it, and runs them with the payload. So if three people are listening on `"chat.123"`, all three get the new message.

---

### Subscribing to stuff

```TypeScript
subscribe<T>(topic: string) {
  const listeners = this.topics.get(topic) ?? new Set<Listener<T>>();
  this.topics.set(topic, listeners);
```

When you subscribe, it either finds the existing set of listeners for that topic, or makes a new one if none exist.

Then it sets up a little **queue** and some plumbing for async iteration:

```TypeScript
const queue: T[] = [];
let resolveNext: ((v: IteratorResult<T>) => void) | null = null;
```

Basically:
- If a message comes in before you ask for it → it gets stored in `queue`.
- If you’re already waiting for the next message → it resolves immediately.

That’s handled in this bit:

```TypeScript
const onMessage = (payload: T) => {
  if (resolveNext) {
    resolveNext({ value: payload, done: false });
    resolveNext = null;
  } else {
    queue.push(payload);
  }
};
```

Then I add this listener to the set:

```TypeScript
listeners.add(onMessage as Listener<T>);
```

---

### The async iterator part

This is where it gets GraphQL-friendly:

```TypeScript
const asyncIterator: AsyncIterator<T> = {
  next: () =>
    new Promise<IteratorResult<T>>((resolve) => {
      if (queue.length) {
        const value = queue.shift()!;
        resolve({ value, done: false });
      } else {
        resolveNext = resolve;
      }
    }),
```

- When GraphQL asks for the next value, if something’s already in `queue`, return it.
- Otherwise, hold onto `resolveNext` and wait until a new payload comes in.

Then there’s cleanup code:

```TypeScript
return: () => {
  listeners.delete(onMessage as Listener<T>);
  return Promise.resolve({ value: undefined, done: true });
},
throw: (err) => {
  listeners.delete(onMessage as Listener<T>);
  return Promise.reject(err);
},
```

So if you unsubscribe or error out, your listener gets removed.

And finally, this makes the object work nicely with `for await` loops:

```TypeScript
[Symbol.asyncIterator]() {
  return this;
}
```

---

### Topic helpers

```TypeScript
export const topicUserChatEvents = (userId: string) =>
  `user.${userId}.chat-events`;

export const topicChatMessageEvents = (chatId: string) =>
  `chat.${chatId}.message-events`;
```

These are just helper functions to keep topic naming consistent. So instead of me remembering to type `"chat.123.message-events"`, I just call `topicChatMessageEvents("123")`.

---

### Exporting the pubsub

```TypeScript
export const pubsub = new SimplePubSub();
```

At the end, I just make one instance of this pubsub thing so I can import it anywhere in my app.

---

### Little Notes

- It’s super lightweight - no external libs, just `Map`, `Set`, and async iterators.
- Listeners aren’t “users” directly. They’re just functions, but usually each one maps to a user’s active subscription.
- Perfect for simple GraphQL subscriptions or any little event system I want to hack together.
