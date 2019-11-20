---
title: Polymorphic Variants considered harmful
revealOptions:
    transition: 'none'
    slideNumber: true
---
<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.0.13/css/all.css" integrity="sha384-DNOHZ68U8hZfKXOrtjWvjxusGo9WQnrNx2sqG0tfsghAvtVlRW3tvkXWZh58N9jp" crossorigin="anonymous">
<h1>Polymorphic Variants considered harmful</h1>

---

<small>
Side note : the first «X considered harmful» was presented in 1968 by Dijkstra with <b>Go To Statements considered harmful</b>
</small>

---

### What are Polymorphic Variants?

---

They are generalized, more dynamic, variants :

```
type a = [ `A | `B ]
```

- They can have a payload
- They don't have to be declared beforhand

---

```
let f = function
 | `A -> "A"
 | `B -> "B"
 | `C -> "C"
```

---

*Type signatures*
- `[ 'A ]` = «exactly 'A»
- `[ 'A | 'B ]` = «'A or 'B»
- `[< 'A | 'B ]` = «'A or 'B or less»
- `[> 'A | 'B ]` = «'A or 'B or more»

---

*Polymorphic Variants can be nested*

```
type a = [ `A | `B ]
type b = [ a | `C ]
type c = [ `D ]
type d = [ b | c ]
```

---

### Example : Caqti_error.t

---

```
type call = [
| `Encode_rejected of coding_error
| `Encode_failed of coding_error
| `Request_rejected of query_error
| `Request_failed of query_error
| `Response_rejected of query_error
]

type retrieve = [
| `Decode_rejected of coding_error
| `Response_failed of query_error
| `Response_rejected of query_error
]

type call_or_retrieve = [
| call
| retrieve
]

type transact = [
| call
| retrieve
]

type load = [
| `Load_rejected of load_error
| `Load_failed of load_error
]

type connect = [
| `Connect_rejected of connection_error
| `Connect_failed of connection_error
| `Post_connect of call_or_retrieve
]

type load_or_connect = [
| load
| connect
]

type t = [
| load
| connect
| call
| retrieve
]
```

---

### A few more examples

---

```
let f = function
 | `A -> "A"
 | `B -> "B"
 | _ -> "?"

val f : [> `A | `B ] -> string
```

---

```
val f : [< `A | `B ] list -> string

val l : [ `A ] list
```

---

### And now for the tricky parts

---

```
type a = [ `A of int ]

let b ( l : [< `A ] ) : string =
  match l with
  | `A -> "A"
```

This is fine. And you will have two different types : one *a* and one *'A*.

---

```
let a = function
  | `A s -> s
  | `B -> "B"

let b = function
  | `A -> "A"
  | `B s -> s
```

---

Pattern matching does not coerce the type. You have to be explicit about it.

```
type a = [ `A ]
type b = [ a | `B ]

let a_to_string = function
  | `A -> "A"

let b_to_string (v : [< `A | `B ]) =
  match v with
  | `A -> a_to_string v
  | `B -> "B"
```

This does not work.

---

```
type a = [ `A ]
type b = [ a | `B ]

let a_to_string = function
  | `A -> "A"

let b_to_string (v : [< `A | `B ]) =
  match v with
  | #a as x -> a_to_string x
  | ( `A ) as x -> a_to_string x (** Two ways to write this **)
  | `B -> "B"
```

---

Never use catch-all pattern matching with polymorphic variants. Bad things may happen :

```
type a = [ `Add1 | `Add2 ]

let f (v : [> a ]) : string = function
  | `Add1 -> "A"
  | `Add2 -> "B"
  | _ -> "Error"
  
f `Addl (** here there is an L instead of a 1, but it will still compile and run **)
```

---

Let's define lists with stuff in them :
```
let l = [ `A; `B ]
>>> [> `A | `B ] list = [`A; `B]
```

The compiler assumes a non-strict type.
This means this works :
```
let l2 = `C::l;;
>>> [> `A | `B | `C ] list = [`C; `A; `B]
```

---

But we can force a stricter type :
```
let l : [< `A | `B ] = [`A; `B];;
>>> [ `A | `B ] list = [`A; `B]
```

This means this does no longer work :
```
let l2 = `C::l;;
Error: This expression has type [ `A | `B ] list
       but an expression was expected of type [> `C ] list
       The first variant type does not allow tag(s) `C
```

---

Sometimes, the compiler will ignore your type constraints.
The following compiles just fine when in an mli :
```
let a (v : [ `I ]) : string =
  match a with
  | `I -> "I"
  | _ -> "?"

let b = a `J
```

---

## Conclusion

---

Mostly don't.

<small>And if you have to : fall back to tinkering in a utop session</small>
