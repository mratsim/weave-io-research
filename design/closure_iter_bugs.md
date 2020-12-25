when true:
  import sugar

  proc countTo(n: int): iterator(): int =
    return iterator(): int =
      var i = 0
      while i <= n:
        yield i
        inc i

  proc takeWhile*[T](iter: iterator(): T, cond: proc(x: T):bool): iterator(): T =
    result = iterator(): T =
      var r {.noinit.} = iter()
      while cond(r):
        yield r
        r = iter()

  let countTo4 = countTo(20).takeWhile(x => x <= 4)

  while not countTo4.finished():
    echo countTo4()

else: # unintuitive solution

  import sugar

  proc countTo(n: int): iterator(): int =
    return iterator(): int =
      var i = 0
      while i <= n:
        yield i
        inc i

  proc takeWhile*[T](iter: iterator(): T, cond: proc(x: T):bool): iterator(): T =
    result = iterator(): T =
      var r = iter()
      while not finished(iter) and cond(r):
        yield r
        r = iter()

  let countTo4 = countTo(20).takeWhile(x => x <= 4)

  while (let next = countTo4(); not countTo4.finished()):
    echo next
