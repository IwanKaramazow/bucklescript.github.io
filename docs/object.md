---
title: Object
---

JavaScript objects are used for two major purposes:

- As a **hash map** (or "dictionary"), where keys can be dynamically added/removed and where values are of the same type.
- As a **record**, where fields are fixed (though still maybe sometimes optional) and where values can be of different types.

Correspondingly, BuckleScript works with JS objects in these 2 ways.

## Hash Map Mode

Until recently, where JS finally got proper Map support, objects have been (ab)used as a map. If you use your JS object like this:

- might or might not add/remove arbitrary keys
- values might or might not be accessed using a dynamic/computed key
- values are all of the same type

Then use our [`Js.Dict`](https://bucklescript.github.io/bucklescript/api/Js.Dict.html) (for "dictionary") API to bind to that JS object! In this mode, you can do all the metaprogramming you're used to with JS objects: get all keys through `Js.Dict.keys`, get values through `Js.Dict.values`, etc.

### Example

```ocaml
(* Create a JS object ourselves *)
let myMap = Js.Dict.empty ()
let () = Js.Dict.set myMap "Allison" 10

(* Use an existing JS object *)
external studentAges : int Js.Dict.t = "student" [@@bs.val]
let () =
  match Js.Dict.get studentAges "Joe" with
  | None -> Js.log "Joe can't be found"
  | Some age -> Js.log ("Joe is " ^ (string_of_int age))
```

```reason
/* Create a JS object ourselves */
let myMap = Js.Dict.empty();
Js.Dict.set(myMap, "Allison", 10);

/* Use an existing JS object */
[@bs.val] external studentAges : Js.Dict.t(int) = "student";
switch (Js.Dict.get(studentAges, "Joe")) {
| None => Js.log("Joe can't be found")
| Some(age) => Js.log("Joe is " ++ string_of_int(age))
};
```

Output:

```js
var Js_dict = require("./stdlib/js_dict.js");

var myMap = { };

myMap["Allison"] = 10;

var match = Js_dict.get(student, "Joe");

if (match !== undefined) {
  console.log("Joe is " + String(match));
} else {
  console.log("Joe can't be found");
}
```

### Design Decisions

You can see that under the hood, a `Js.Dict` is simply backed by a JS object. The entire API uses nothing but ordinary BuckleScript `external`s and wrappers, so the whole API mostly disappears after compilation. It is very convenient when converting files over from JS to BuckleScript.

## Record Mode

If your JS object:

- has a known, fixed set of fields
- might or might not contain values of different types

Then you're really using it like a "record" in most other languages. For example, think of the difference of use-case and intent between the object `{"John": 10, "Allison": 20, "Jimmy": 15}` and `{name: "John", age: 10, job: "CEO"}`. The former case would be the aforementioned "hash map mode". The latter would be "record mode", which in BuckleScript is modeled with the `bs.deriving abstract` feature:

```ocaml
type person = {
  name: string;
  age: int;
  job: string;
} [@@bs.deriving abstract]

external john : person = "john" [@@bs.val]
```

```reason
[@bs.deriving abstract]
type person = {
  name: string,
  age: int,
  job: string,
};

[@bs.val] external john : person = "john";
```

**Note**: the `person` type is **not** a record! It's a record-looking type that uses the record's syntax and type-checking. The `bs.deriving abstract` annotation turns it into an "abstract type" (aka you don't know what the actual value's shape).

### Creation

You don't have to bind to an existing `person` object from the JS side. You can also create such `person` JS object from BuckleScript's side.

Since `bs.deriving abstract` turns the above `person` record into an abstract type, you can't directly create a person record as you would usually. This doesn't work: `{name: "Joe", age: 20, job: "teacher"}`.

Instead, you'd use the **creation function** of the same name as the record type, implicitly generated by the `bs.deriving abstract` annotation:

```ocaml
let joe = person ~name:"Joe" ~age:20 ~job:"teacher"
```

```reason
let joe = person(~name="Joe", ~age=20, ~job="teacher")
```

Output:

```js
var joe = {
  name: "Joe",
  age: 20,
  job: "teacher"
};
```

Look ma, no runtime cost!

#### Rename Fields

Sometimes you might be binding to a JS object with field names that are invalid in BuckleScript/Reason. Two examples would be `{type: "foo"}` (reserved keyword in BS/Reason) and `{"aria-checked": true}`. Choose a valid field name then use `[@bs.as]` to circumvent this:

```ocaml
type data = {
  type_: string [@bs.as "type"];
  ariaLabel: string [@bs.as "aria-label"];
} [@@bs.deriving abstract]

let d = data ~type_:"message" ~ariaLabel:"hello"
```

```reason
[@bs.deriving abstract]
type data = {
  [@bs.as "type"] type_: string,
  [@bs.as "aria-label"] ariaLabel: string,
};

let d = data(~type_="message", ~ariaLabel="hello");
```

Output:

```js
var d = {
  type: "message",
  "aria-label": "hello"
};
```

#### Optional Labels

You can omit fields during the creation of the object:

```ocaml
type person = {
  name: string [@bs.optional];
  age: int;
  job: string;
} [@@bs.deriving abstract]

let joe = person ~age:20 ~job:"teacher" ()
```

```reason
[@bs.deriving abstract]
type person = {
  [@bs.optional] name: string,
  age: int,
  job: string,
};

let joe = person(~age=20, ~job="teacher", ());
```

**Note** that the `[@bs.optional]` tag turned the `name` field optional. Merely typing `name` as `option(string)` wouldn't work.

**Note**: now that your creation function contains optional fields, we mandate an unlabeled `()` at the end to indicate that [you've finished applying the function](https://reasonml.github.io/docs/en/function.html#optional-labeled-arguments).

### Accessors

Again, since `bs.deriving abstract` hides the actual record shape, you can't access a field using e.g. `joe.age`. We remediate this by generating getter and setters.

#### Read

One getter function is generated per `bs.deriving abstract` record type field. In the above example, you'd get 3 functions: `nameGet`, `ageGet`, `jobGet`. They take in a `person` value and return `string`, `int`, `string` respectively:

```ocaml
let twenty = ageGet joe
```

```reason
let twenty = ageGet(joe)
```

Alternatively, you can use the [Pipe First](pipe-first.md) feature in a later section for a nicer-looking access syntax:

```ocaml
let twenty = joe |. ageGet
```

```reason
let twenty = joe->ageGet
```

If you prefer shorter names for the getter functions, [we also support a 'light' setting](https://bucklescript.github.io/blog/2019/03/21/release-5-0):

```ocaml
type person = {
  name: string;
  age: int;
} [@@bs.deriving {abstract = light}]

let joe = person ~name:"Joe" ~age:20 ()
let joeName = name joe
```

```reason
[@bs.deriving {abstract: light}]
type person = {
  name: string,
  age: int,
};

let joe = person(~name="Joe", ~age=20, ());
let joeName = name(joe);
```

The getter functions will now have the same names as the object fields themselves.

#### Write

A `bs.deriving abstract` value is immutable by default. To mutate such value, you need to first mark one of the abstract record field as `mutable`, the same way you'd mark a normal record as mutable:

```ocaml
type person = {
  name: string;
  mutable age: int;
  job: string;
} [@@bs.deriving abstract]
```

```reason
[@bs.deriving abstract]
type person = {
  name: string,
  mutable age: int,
  job: string,
};
```

Then, a setter of the name `ageSet` will be generated. Use it like so:

```ocaml
let joe = person ~name:"Joe" ~age:20 ~job:"teacher"
let () = ageSet joe 21
```

```reason
let joe = person(~name="Joe", ~age=20, ~job="teacher");
ageSet(joe, 21);
```

Alternatively, with the Pipe First syntax:

```ocaml
joe |. ageSet 21
```

```reason
joe->ageSet(21)
```

### Methods

You can attach arbitrary methods onto a type (_any_ type, as a matter of fact. Not just `bs.deriving abstract` record types). See [Object Method](function.md#object-method) in the function section later.

### Tips & Tricks

You can leverage `bs.deriving abstract` for finer-grained access control.

#### Mutability

You can mark a field as mutable in the implementation (`ml`/`re`) file, while _hiding_ such mutability in the interface file:

```ocaml
(* test.ml *)
type cord = {
  mutable x: int [@bs.optional];
  y: int;
} [@@bs.deriving abstract]
```

```reason
/* test.re */
[@bs.deriving abstract]
type cord = {
  [@bs.optional] mutable x: int,
  y: int,
};
```

```ocaml
(* test.mli *)
type cord = {
  x: int [@bs.optional];
  y: int;
} [@@bs.deriving abstract]
```

```reason
/* test.rei */
[@bs.deriving abstract]
type cord = {
  [@bs.optional] x: int,
  y: int,
};
```

Tada! Now you can mutate inside your own file as much as you want, and prevent others from doing so!

#### Hide the Creation Function

Mark the record as `private` to disable the creation function:

```ocaml
type cord = private {
  x: int [@bs.optional];
  y: int
} [@@bs.deriving abstract]
```

```reason
[@bs.deriving abstract]
type cord = pri {
  [@bs.optional] x: int,
  y: int,
};
```

The accessors are still there, but you can no longer create such data structure. Great for binding to a JS object while preventing others from creating more such object!
