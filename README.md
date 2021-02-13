# Object.mapEntries()

## Status

Champion(s): None (Currently looking for champions)

Author(s): Scotty Jamison

Stage: -1

## Motivation

ECMAScript currently does not provide much support for transforming objects. Because of this, it is common to see programmers use `Object.entries()` to turn an object into an array of entries, manipulate the array using the plethora of array-manipulation functions provided, then convert it back to an object with `Object.fromEntries()`. Other techniques exist, such as building the desired object by reducing over a source object's entries.

This proposal aims to reduce a lot of friction related to object-manipulation by providing a general-purpose object-mapping function. Having a standard `Object.mapEntries()` function would not only provide a convenient tool to map objects but will also provide documentation on how numerous edge-cases get handled when performing such an operation (e.g. how to handle inherited properties, non-enumerable properties, symbols, etc).

## Description

`Object.mapEntries(obj, mapFn) -> resultObj` takes two parameters:
* An `obj`. Each own entry of the object will be mapped over by the `mapFn` to produce a new object.
* A `mapFn` that receives one parameter: An entry of the shape `[key, value]`. The map function will then return a new entry of the same shape, which will be used to construct the new object.

Caveats to be aware of:
* Only an object's enumerable, own entries with string keys will be iterated over. All symbols will be ignored.
* The result object prototype will always be `Object.prototype`, irrespective of the source object's prototype.
* all of the properties of the result object will be writable, configurable data properties, irrespective of how the properties were defined on the source object.

## Examples

Example function from a REST API facade:
```javascript
// Without Object.mapEntries()
async function fetchUsers() {
  const rawUsers = await fetch(GET_USERS_ENDPOINT).then(response => response.json())

  return Object.entries(rawUsers)
    .reduce((users, [id, { name, age }]) => ({
      ...users,
      [id]: { fullName: name, isAdult: age >= 18 },
    }), {})
}

// With Object.mapEntries()
async function fetchUsers() {
  const rawUsers = await fetch(GET_USERS_ENDPOINT).then(response => response.json())

  return Object.mapEntries(rawUsers,
    ([id, { name, age }]) => [id, { fullName: name, isAdult: age >= 18 }]
  )
}
```

Example utility function - flipping a mapping:
```javascript
// Without Object.mapEntries()
const flip = obj => {
  const entries = Object.entries(obj)
  const flippedEntries = entries.map(([key, value]) => [value, key])
  return Object.fromEntries(flippedEntries)
}

// With Object.mapEntries()
const flip = obj => Object.mapEntries(obj, ([key, value]) => [value, key])
```

## Comparison

This is how other programming languages provide the same functionality:

* Python users can use [dictionary comprehension](https://docs.python.org/3/tutorial/datastructures.html#dictionaries) to achieve this same task.
* In Java, you first convert your map to pairs, turn it into a [stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html), call map, before finally collecting the pairs back into a map object. Mapping array values in java follow a similar process.
* Haskell provides the following functions ([docs](https://hackage.haskell.org/package/containers-0.4.0.0/docs/Data-Map.html)):
  * map(): Maps values to values
  * mapWithKeys(): Maps key-value pairs to new values
  * mapKeys(): Maps keys to new keys
  * A handful of other mapping functions to do other tasks as you map, such as remove entries.

This is how javascript libraries add this functionality:
* Lodash provides `_.mapKeys(obj, mapper)` which maps key-value pairs to new keys, and `_.mapValues(obj, mapper)` which maps key-value pairs to new values.
* Rambda provides `R.map(mapper, obj)` which maps values to values (and works on both arrays and objects), and `R.mapObjIndexed(mapper, obj)` which maps key-values pairs to values. They do not provide a way to change an object's keys.
* MooTools provides `Object.map(obj, mapper[, bind])` to map key-value pairs to values.

`Object.mapEntries()`'s behavior was chosen for its power - it allows one to modify both an object's keys and values at the same time. Such behavior does seem to be unique compared to other libraries, thus these decisions are very much open for discussion. We could, for example, provide functions such as the following instead of `Object.mapEntries()`, or in addition to it:
* `Object.mapValues(obj, mapper)`, where mapper is of the form `(value, key) => newValue`. This tailors to the most common object mapping use-case.
* `Object.mapKeys(obj, mapper)`, where mapper is of the form `key => newKey`

## Implementation

`Object.mapEntries()`'s definition is equivalent to the following code snippet:

```javascript
Object.defineProperty(Object, 'mapEntries', {
  writable: true,
  configurable: true,
  value(obj, mapFn) {
    const entries = Object.entries(obj)
    const mappedEntries = entries.map(entry => mapFn(entry))
    return Object.fromEntries(mappedEntries)
  }
})
```