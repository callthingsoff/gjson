<p align="center">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="/.github/images/logo-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="/.github/images/logo-light.png">
  <img src="/.github/images/logo-light.png" width="240" alt="GJSON" >
</picture>
<br>
<a href="https://godoc.org/github.com/tidwall/gjson"><img src="https://img.shields.io/badge/api-reference-blue.svg?style=flat-square" alt="GoDoc"></a>
<a href="https://tidwall.com/gjson-play"><img src="https://img.shields.io/badge/%F0%9F%8F%90-playground-9900cc.svg?style=flat-square" alt="GJSON Playground"></a>
<a href="SYNTAX.md"><img src="https://img.shields.io/badge/{}-syntax-33aa33.svg?style=flat-square" alt="GJSON Syntax"></a>
	
</p>

<p align="center">get json values quickly</a></p>

GJSON is a Go package that provides a [fast](#performance) and [simple](#get-a-value) way to get values from a json document.
It has features such as [one line retrieval](#get-a-value), [dot notation paths](#path-syntax), [iteration](#iterate-through-an-object-or-array), and [parsing json lines](#json-lines).

Also check out [SJSON](https://github.com/tidwall/sjson) for modifying json, and the [JJ](https://github.com/tidwall/jj) command line tool.

This README is a quick overview of how to use GJSON, for more information check out [GJSON Syntax](SYNTAX.md).

GJSON is also available for [Python](https://github.com/volans-/gjson-py) and [Rust](https://github.com/tidwall/gjson.rs)

Getting Started
===============

## Installing

To start using GJSON, install Go and run `go get`:

```sh
$ go get -u github.com/tidwall/gjson
```

This will retrieve the library.

## Get a value
Get searches json for the specified path. A path is in dot syntax, such as "name.last" or "age". When the value is found it's returned immediately. 

```go
package main

import "github.com/tidwall/gjson"

const json = `{"name":{"first":"Janet","last":"Prichard"},"age":47}`

func main() {
	value := gjson.Get(json, "name.last")
	println(value.String())
}
```

This will print:

```
Prichard
```
*There's also [GetBytes](#working-with-bytes) for working with JSON byte slices.*

## Path Syntax

Below is a quick overview of the path syntax, for more complete information please
check out [GJSON Syntax](SYNTAX.md).

A path is a series of keys separated by a dot.
A key may contain special wildcard characters '\*' and '?'.
To access an array value use the index as the key.
To get the number of elements in an array or to access a child path, use the '#' character.
The dot and wildcard characters can be escaped with '\\'.

```json
{
  "name": {"first": "Tom", "last": "Anderson"},
  "age":37,
  "children": ["Sara","Alex","Jack"],
  "fav.movie": "Deer Hunter",
  "friends": [
    {"first": "Dale", "last": "Murphy", "age": 44, "nets": ["ig", "fb", "tw"]},
    {"first": "Roger", "last": "Craig", "age": 68, "nets": ["fb", "tw"]},
    {"first": "Jane", "last": "Murphy", "age": 47, "nets": ["ig", "tw"]}
  ]
}
```
```
"name.last"          >> "Anderson"
"age"                >> 37
"children"           >> ["Sara","Alex","Jack"]
"children.#"         >> 3
"children.1"         >> "Alex"
"child*.2"           >> "Jack"
"c?ildren.0"         >> "Sara"
"fav\.movie"         >> "Deer Hunter"
"friends.#.first"    >> ["Dale","Roger","Jane"]
"friends.1.last"     >> "Craig"
```

You can also query an array for the first match by using `#(...)`, or find all 
matches with `#(...)#`. Queries support the `==`, `!=`, `<`, `<=`, `>`, `>=` 
comparison operators and the simple pattern matching `%` (like) and `!%` 
(not like) operators.

```
friends.#(last=="Murphy").first    >> "Dale"
friends.#(last=="Murphy")#.first   >> ["Dale","Jane"]
friends.#(age>45)#.last            >> ["Craig","Murphy"]
friends.#(first%"D*").last         >> "Murphy"
friends.#(first!%"D*").last        >> "Craig"
friends.#(nets.#(=="fb"))#.first   >> ["Dale","Roger"]
```

*Please note that prior to v1.3.0, queries used the `#[...]` brackets. This was
changed in v1.3.0 as to avoid confusion with the new
[multipath](SYNTAX.md#multipaths) syntax. For backwards compatibility, 
`#[...]` will continue to work until the next major release.*

## Result Type

GJSON supports the json types `string`, `number`, `bool`, and `null`. 
Arrays and Objects are returned as their raw json types. 

The `Result` type holds one of these:

```
bool, for JSON booleans
float64, for JSON numbers
string, for JSON string literals
nil, for JSON null
```

To directly access the value:

```go
result.Type           // can be String, Number, True, False, Null, or JSON
result.Str            // holds the string
result.Num            // holds the float64 number
result.Raw            // holds the raw json
result.Index          // index of raw value in original json, zero means index unknown
result.Indexes        // indexes of all the elements that match on a path containing the '#' query character.
```

There are a variety of handy functions that work on a result:

```go
result.Exists() bool
result.Value() any
result.Int() int64
result.Uint() uint64
result.Float() float64
result.String() string
result.Bool() bool
result.Time() time.Time
result.Array() []gjson.Result
result.Map() map[string]gjson.Result
result.Get(path string) Result
result.ForEach(iterator func(key, value Result) bool)
result.Less(token Result, caseSensitive bool) bool
```

The `result.Value()` function returns an `any` which requires type assertion and is one of the following Go types:

```go
boolean >> bool
number  >> float64
string  >> string
null    >> nil
array   >> []any
object  >> map[string]any
```

The `result.Array()` function returns back an array of values.
If the result represents a non-existent value, then an empty array will be returned.
If the result is not a JSON array, the return value will be an array containing one result.

### 64-bit integers

The `result.Int()` and `result.Uint()` calls are capable of reading all 64 bits, allowing for large JSON integers.

```go
result.Int() int64    // -9223372036854775808 to 9223372036854775807
result.Uint() uint64   // 0 to 18446744073709551615
```

## Modifiers and path chaining 

New in version 1.2 is support for modifier functions and path chaining.

A modifier is a path component that performs custom processing on the 
json.

Multiple paths can be "chained" together using the pipe character. 
This is useful for getting results from a modified query.

For example, using the built-in `@reverse` modifier on the above json document,
we'll get `children` array and reverse the order:

```
"children|@reverse"           >> ["Jack","Alex","Sara"]
"children|@reverse|0"         >> "Jack"
```

There are currently the following built-in modifiers:

- `@reverse`: Reverse an array or the members of an object.
- `@ugly`: Remove all whitespace from a json document.
- `@pretty`: Make the json document more human readable.
- `@this`: Returns the current element. It can be used to retrieve the root element.
- `@valid`: Ensure the json document is valid.
- `@flatten`: Flattens an array.
- `@join`: Joins multiple objects into a single object.
- `@keys`: Returns an array of keys for an object.
- `@values`: Returns an array of values for an object.
- `@tostr`: Converts json to a string. Wraps a json string.
- `@fromstr`: Converts a string from json. Unwraps a json string.
- `@group`: Groups arrays of objects. See [e4fc67c](https://github.com/tidwall/gjson/commit/e4fc67c92aeebf2089fabc7872f010e340d105db).
- `@dig`: Search for a value without providing its entire path. See [e8e87f2](https://github.com/tidwall/gjson/commit/e8e87f2a00dc41f3aba5631094e21f59a8cf8cbf).

### Modifier arguments

A modifier may accept an optional argument. The argument can be a valid JSON 
document or just characters.

For example, the `@pretty` modifier takes a json object as its argument. 

```
@pretty:{"sortKeys":true} 
```

Which makes the json pretty and orders all of its keys.

```json
{
  "age":37,
  "children": ["Sara","Alex","Jack"],
  "fav.movie": "Deer Hunter",
  "friends": [
    {"age": 44, "first": "Dale", "last": "Murphy"},
    {"age": 68, "first": "Roger", "last": "Craig"},
    {"age": 47, "first": "Jane", "last": "Murphy"}
  ],
  "name": {"first": "Tom", "last": "Anderson"}
}
```

*The full list of `@pretty` options are `sortKeys`, `indent`, `prefix`, and `width`. 
Please see [Pretty Options](https://github.com/tidwall/pretty#customized-output) for more information.*

### Custom modifiers

You can also add custom modifiers.

For example, here we create a modifier that makes the entire json document upper
or lower case.

```go
gjson.AddModifier("case", func(json, arg string) string {
  if arg == "upper" {
    return strings.ToUpper(json)
  }
  if arg == "lower" {
    return strings.ToLower(json)
  }
  return json
})
```

```
"children|@case:upper"           >> ["SARA","ALEX","JACK"]
"children|@case:lower|@reverse"  >> ["jack","alex","sara"]
```

## JSON Lines

There's support for [JSON Lines](http://jsonlines.org/) using the `..` prefix, which treats a multilined document as an array. 

For example:

```
{"name": "Gilbert", "age": 61}
{"name": "Alexa", "age": 34}
{"name": "May", "age": 57}
{"name": "Deloise", "age": 44}
```

```
..#                   >> 4
..1                   >> {"name": "Alexa", "age": 34}
..3                   >> {"name": "Deloise", "age": 44}
..#.name              >> ["Gilbert","Alexa","May","Deloise"]
..#(name="May").age   >> 57
```

The `ForEachLines` function will iterate through JSON lines.

```go
gjson.ForEachLine(json, func(line gjson.Result) bool{
    println(line.String())
    return true
})
```

## Get nested array values

Suppose you want all the last names from the following json:

```json
{
  "programmers": [
    {
      "firstName": "Janet", 
      "lastName": "McLaughlin", 
    }, {
      "firstName": "Elliotte", 
      "lastName": "Hunter", 
    }, {
      "firstName": "Jason", 
      "lastName": "Harold", 
    }
  ]
}
```

You would use the path "programmers.#.lastName" like such:

```go
result := gjson.Get(json, "programmers.#.lastName")
for _, name := range result.Array() {
	println(name.String())
}
```

You can also query an object inside an array:

```go
name := gjson.Get(json, `programmers.#(lastName="Hunter").firstName`)
println(name.String())  // prints "Elliotte"
```

## Iterate through an object or array

The `ForEach` function allows for quickly iterating through an object or array. 
The key and value are passed to the iterator function for objects.
Only the value is passed for arrays.
Returning `false` from an iterator will stop iteration.

```go
result := gjson.Get(json, "programmers")
result.ForEach(func(key, value gjson.Result) bool {
	println(value.String()) 
	return true // keep iterating
})
```

## Simple Parse and Get

There's a `Parse(json)` function that will do a simple parse, and `result.Get(path)` that will search a result.

For example, all of these will return the same result:

```go
gjson.Parse(json).Get("name").Get("last")
gjson.Get(json, "name").Get("last")
gjson.Get(json, "name.last")
```

## Check for the existence of a value

Sometimes you just want to know if a value exists. 

```go
value := gjson.Get(json, "name.last")
if !value.Exists() {
	println("no last name")
} else {
	println(value.String())
}

// Or as one step
if gjson.Get(json, "name.last").Exists() {
	println("has a last name")
}
```

## Validate JSON

The `Get*` and `Parse*` functions expects that the json is well-formed. Bad json will not panic, but it may return back unexpected results.

If you are consuming JSON from an unpredictable source then you may want to validate prior to using GJSON.

```go
if !gjson.Valid(json) {
	return errors.New("invalid json")
}
value := gjson.Get(json, "name.last")
```

## Unmarshal to a map

To unmarshal to a `map[string]any`:

```go
m, ok := gjson.Parse(json).Value().(map[string]any)
if !ok {
	// not a map
}
```

## Working with Bytes

If your JSON is contained in a `[]byte` slice, there's the [GetBytes](https://godoc.org/github.com/tidwall/gjson#GetBytes) function. This is preferred over `Get(string(data), path)`.

```go
var json []byte = ...
result := gjson.GetBytes(json, path)
```

If you are using the `gjson.GetBytes(json, path)` function and you want to avoid converting `result.Raw` to a `[]byte`, then you can use this pattern:

```go
var json []byte = ...
result := gjson.GetBytes(json, path)
var raw []byte
if result.Index > 0 {
    raw = json[result.Index:result.Index+len(result.Raw)]
} else {
    raw = []byte(result.Raw)
}
```

This is a best-effort no allocation sub slice of the original json. This method utilizes the `result.Index` field, which is the position of the raw data in the original json. It's possible that the value of `result.Index` equals zero, in which case the `result.Raw` is converted to a `[]byte`.

## Performance

Benchmarks of GJSON alongside [encoding/json](https://golang.org/pkg/encoding/json/), 
[ffjson](https://github.com/pquerna/ffjson), 
[EasyJSON](https://github.com/mailru/easyjson),
[jsonparser](https://github.com/buger/jsonparser),
and [json-iterator](https://github.com/json-iterator/go)

```
BenchmarkGJSONGet-10             17893731    202.1 ns/op      0 B/op     0 allocs/op
BenchmarkGJSONUnmarshalMap-10     1663548   2157 ns/op     1920 B/op    26 allocs/op
BenchmarkJSONUnmarshalMap-10       832236   4279 ns/op     2920 B/op    68 allocs/op
BenchmarkJSONUnmarshalStruct-10   1076475   3219 ns/op      920 B/op    12 allocs/op
BenchmarkJSONDecoder-10            585729   6126 ns/op     3845 B/op   160 allocs/op
BenchmarkFFJSONLexer-10           2508573   1391 ns/op      880 B/op     8 allocs/op
BenchmarkEasyJSONLexer-10         3000000    537.9 ns/op    501 B/op     5 allocs/op
BenchmarkJSONParserGet-10        13707510    263.9 ns/op     21 B/op     0 allocs/op
BenchmarkJSONIterator-10          3000000    561.2 ns/op    693 B/op    14 allocs/op
```

JSON document used:

```json
{
  "widget": {
    "debug": "on",
    "window": {
      "title": "Sample Konfabulator Widget",
      "name": "main_window",
      "width": 500,
      "height": 500
    },
    "image": { 
      "src": "Images/Sun.png",
      "hOffset": 250,
      "vOffset": 250,
      "alignment": "center"
    },
    "text": {
      "data": "Click Here",
      "size": 36,
      "style": "bold",
      "vOffset": 100,
      "alignment": "center",
      "onMouseUp": "sun1.opacity = (sun1.opacity / 100) * 90;"
    }
  }
}    
```

Each operation was rotated through one of the following search paths:

```
widget.window.name
widget.image.hOffset
widget.text.onMouseUp
```

**

*These benchmarks were run on a MacBook Pro M1 Max using Go 1.22 and can be found [here](https://github.com/tidwall/gjson-benchmarks).*
