# Design

## Important considerations

### Control

- Is it possible to manually switch to a specific continuation/stack?
  - Use-case 1 is implementing simulation of various threads interleaving
    to formally verify concurrent data structures
    i.e. that no possible thread interleaving lead to undefined behaviors.
  - Use-case 2 is enabling Lua like coroutines.

### Debugging

- What does debugging look like?
- How does it compare to closure iterator or async stack traces?

### Embedded

- Is there a subset or primitives that can be used in embedded?

### Overhead

- Allocations, custom allocator
