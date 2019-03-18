# `nil` in Go

## When is `nil` when is not?

```go
var s fmt.Stringer      // Stringer(nil, nil)
fmt.Println(s == nil)   // true
// (nil, nil) == nil
```

```go
var p *Person             // nil of type *Person
var s fmt.Stringer = p    // Stringer(*Person, nil)
fmt.Println(s == nil)     // false
// (*Person, nil) != nil
```

```go
func do() error {
  var err *doError          // error (*doError, nil)
  return err                // nil of type *doError
}

func main() {
  err := do()               // error (*doError, nil)
  fmt.Println(err == nil)   // false
}
```

## Do not declare ***concrete error*** vars

```go
func do() *doError {  // nil of type *doError
  return nil
}

func main() {
  err := do()               // nil of type *doError
  fmt.Println(err == nil)   // true
}
```

Which will lead to

```go
func do () *doError {    // nil of type *doError
  return nil
}

func wrapDo() error {   // error (*doError, nil)
  return do()           // nil of type *doError
}

func main() {
  err := wrapDo()           // error (*doError, nil)
  fmt.Println(err == nil)   // false
}
```

> *!!! DO NOT return **concrete error** types*

## Kinds of `nil`

| type       | pointer                                       |
| ---------- | --------------------------------------------- |
| pointers   | point to nothing                              |
| slices     | have no backing array                         |
| mpas       | are not initialized                           |
| channels   | are not initialized                           |
| functions  | are not initialized                           |
| interfaces | have no value assigned, no even a nil pointer |

### How are these useful

#### `nil` pointer

![image](https://user-images.githubusercontent.com/15927967/54538811-ec1ea500-49cf-11e9-9f07-d7cb6deeeadf.png)

```go
type tree struct {
  v int
  l *tree
  r *tree
}

func (t *tree) Sum() int {
  sum := t.v
  
  if t.l != nil {
    sum += t.l.Sum()
  }

  if t.r != nil {
    sum += t.r.Sum()
  }

  return sum

}
/* Issues
 * - code repetition:
 *     if v != nil { v.m() }
 * - panic when t is nil
 *     var t *tree
 *     sum := t.Sum()
 */
```

> Pointer receivers
> ```go
> type person struct {}
> func sayHi(p *person)     { fmt.Println("hi") }
> func (p *person) sayHi()  { fmt.Println("hi") }
> var p *person
> p.sayHi()   // hi
>```

nil receivers are useful: 
- Sum
  ```go
  func (t *tree) Sum() int {
    if t == nil {
      return 0
    }

    return t.v + t.l.Sum() + t.r.Sum()
  }
  ```

- String
  ```go
  func (t *tree) String() string {
    if t == nil {
      return ""
    }

    return fmt.Sprint(t.l.String(), t.v, t.r.String())
  }
  ```

- Find
  ```go
  func (t *tree) Find(v int) bool {
    if t == nil {
      return false
    }

  return t.v == v || t.l.Find(v) || t.r.Find(v)
  }
  ```
Try to keep init values to be `nil` to leverage `nil`
> if posssible, if not NewX()

#### `nil` slices

```go
var s []slice

len(s)        // 0
cap(s)        // 0
for rage 0    // iterates zero times
s[i]          // panic: index out of range
```

```go
// guess what will be produced here
var s []int
for i := 0; i < 10; i++ {
  fmt.Printf("len: %2d cap: %2d\n", len(s), cap(s))
  s = append(s, i)
}
// len:  0 cap:  0  []  -> s is nil
// len:  1 cap:  2  [0]  -> allocation
// len:  2 cap:  2  [0 1]  -> reallocation
// len:  3 cap:  4  [0 1 2]  -> reallocation
// len:  4 cap:  4  [0 1 2 3]
// len:  5 cap:  8  [0 1 2 3 4]  -> reallocation
// len:  6 cap:  8  [0 1 2 3 4 5]
// len:  7 cap:  8  [0 1 2 3 4 5 6]
// len:  8 cap:  8  [0 1 2 3 4 5 6 7]
// len:  9 cap: 16  [0 1 2 3 4 5 6 7 8]  -> reallocation
```

#### `nil` maps
```go
var m map[t]u

len(m)          // 0
for range m     // iterates zero times
v, ok := m[i]   // zero(u), false
m[i]            // panic: assignment to entry in nil map
```
using maps
```go
func NewGet(url string, headers map[string]string) (*http.Request, error) {

  req, err := http.NewRequest(http.MethodGet, url, nil)
  if err != nil {
    return nil, err
  }

  for k, v := range headers {
    req.Header.Set(k, v)
  }

  return req, nil
}

NewGet(
  "http://google.com",
  map[string]string{
    "USER_AGENT": "golang/gopher",
  },
)
// GET / HTTP/1.1
// Host: google.com
// User_agent: golang/gopher

NewGet("http://google.com", map[string]string{})
// GET / HTTP/1.1
// Host: google.com

NewGet("http://google.com", nil)
// GET / HTTP/1.1
// Host: google.com
```

*use `nil` maps as **read-only empty maps***

#### `nil` channels

```go
var c chan t
<- c        // blocks forever
c <- x      // blocks forever
close(c)    // panic: close of nil channel
```

imagine this `1` from `a` and `2` from `b` \
![image](https://user-images.githubusercontent.com/15927967/54541981-e926b300-49d5-11e9-8b1f-bba185ce6b35.png)

```go
// a native solution
func merge(out chan<- int, a, b <-chan int) {
  for {
    select {
      case v := <-a:
        out <- v
      case v := <-b:
        out <- v
    }
  }
}
// 2 2 1 2 2 1 2 1 2 1 2 2 2 1 2 1 2 2 1 1 2 0 0 1 0 0 0 0 0 0 0 0 0
// 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
// 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
// 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
// 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
// 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 ...
```

```go
//closed channeds
var c chan t
v, ok <- c    // zert(t), false
c <- x        // panic: send on closed channel
close(c)      // panic: close of nil channel
```

```go
// to fix
func merge(out chan<- int, a, b <-chan int) {
  var aClosed, bClosed bool
  for !aClosed || !bClosed {
    select {
      case v, ok := <-a:
        if !ok {
          a = nil // (fix 2)
          aClosed = true; fmt.Println("a is closed"); continue 
        }
        out <- v
      case v, ok := <-b:
        if !ok { bClosed = true; fmt.Println("b is closed"); continue }
        out <- v
    }
  }
  close(out) // (fix)
}
// 2 2 1 2 2 1 2 1 2 1 2 2 2 1 2 1 2 2 1 1 2
// fatal error: all goroutines are asleep - deadlock!

// to fix, add (fix) - then it will case more problem
// 2 2 1 2 2 1 2 1 2 1 2 2 2 1 2 1 2 2 1 1 2
// b is closed
// ...
// b is closed
// 1
// b is closed
// ...
// b is closed
// 1
// a is closed
// whic will burn the CPU
// to fix, add (fix 2)
```


#### `nil` functions