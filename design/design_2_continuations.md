## Constraints

- try/except over a resume point?
- move only
- On the stack and reuse only if they
  involve trivially destructible types `supportsCopyMem`
  - Can we extend for non-GC type?
    - non-ref types with non-trivial destructors
      should be OK since they are moved.


## Passing parameters when resuming
