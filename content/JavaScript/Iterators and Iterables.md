---
created: 2024-04-10T00:24
updated: 2024-04-10T18:25
---


# Iterable Protocol

The iterable protocol allows JavaScript objects to customize their iteration behavior meaning what values are produced when we loop over such an object using the `for...of ` loops.

## Builtin Iterables

Examples of builtin are `Array`, `Map` or `Set`.

## How to create an Iterable
In order for an object to be an iterable an object must implement the `@@iterator` method available via the `Symbol.iterator` constant.

Whenever an object needs to be iterated, its @@iterator method is called with no arguments, and the returned iterator is used to obtain the values to be iterated.

```ts
function createIterable(start: number, stop: number) {
  return {
    current: start,
    [Symbol.iterator]() {
      return this
    },
    next() {
      while (this.current < stop) {
        return { value: this.current++, done: false };
      }
	
    },
  };
}

```

# Iterator Protocol

