<div align="center">
  <img src="logo.png?raw=true" />
  <br>
  <br>

  <p>
    A binary schema-based serialization algorithm. <br>
    Although it was born to be used on <a href="https://github.com/colyseus/colyseus">Colyseus</a>, this library can be used as standalone.
  </p>

  <a href="https://travis-ci.org/colyseus/schema">
    <img src="https://img.shields.io/travis/colyseus/schema.svg?style=for-the-badge" alt="Build status" />
  </a>
  <a href="https://patreon.com/endel" title="Donate to this project using Patreon">
    <img src="https://img.shields.io/badge/endpoint.svg?url=https%3A%2F%2Fshieldsio-patreon.herokuapp.com%2Fendel&style=for-the-badge" alt="Patreon donate button"/>
  </a>
</div>

Decoders currently available for JavaScript, LUA, C#, C++ and Haxe.

## Defining Schema

As Colyseus is written in TypeScript, the schema is defined as type annotations inside the state class. Additional server logic may be added to that class, but client-side generated (not implemented) files will consider only the schema itself.

```typescript
import { Schema, type, ArraySchema, MapSchema } from '@colyseus/schema';

export class Player extends Schema {
  @type("string")
  name: string;

  @type("number")
  x: number;

  @type("number")
  y: number;
}

export class State extends Schema {
  @type('string')
  fieldString: string;

  @type('number') // varint
  fieldNumber: number;

  @type(Player)
  player: Player;

  @type([ Player ])
  arrayOfPlayers: ArraySchema<Player>;

  @type({ map: Player })
  mapOfPlayers: MapSchema<Player>;
}
```

See [example](test/Schema.ts).

## Supported types

## Primitive Types

| Type | Description | Limitation |
|------|-------------|------------|
| string | utf8 strings | maximum byte size of `4294967295` |
| number | auto-detects `int` or `float` type. (extra byte on output) | `0` to `18446744073709551615` |
| boolean | `true` or `false` | `0` or `1` |
| int8 | signed 8-bit integer | `-128` to `127` |
| uint8 | unsigned 8-bit integer | `0` to `255` |
| int16 | signed 16-bit integer | `-32768` to `32767` |
| uint16 | unsigned 16-bit integer | `0` to `65535` |
| int32 | signed 32-bit integer | `-2147483648` to `2147483647` |
| uint32 | unsigned 32-bit integer | `0` to `4294967295` |
| int64 | signed 64-bit integer | `-9223372036854775808` to `9223372036854775807` |
| uint64 | unsigned 64-bit integer | `0` to `18446744073709551615` |
| float32 | single-precision floating-point number | `-3.40282347e+38` to `3.40282347e+38`|
| float64 | double-precision floating-point number | `-1.7976931348623157e+308` to `1.7976931348623157e+308` |

### Declaration:

#### Primitive types (`string`, `number`, `boolean`, etc)

```typescript
@type("string")
name: string;

@type("int32")
name: number;
```

#### Custom `Schema` type

```typescript
@type(Player)
player: Player;
```

#### Array of custom `Schema` type

```typescript
@type([ Player ])
arrayOfPlayers: ArraySchema<Player>;
```

#### Array of a primitive type

You can't mix types inside arrays.

```typescript
@type([ "number" ])
arrayOfNumbers: ArraySchema<number>;

@type([ "string" ])
arrayOfStrings: ArraySchema<string>;
```

#### Map of custom `Schema` type

```typescript
@type({ map: Player })
mapOfPlayers: MapSchema<Player>;
```

#### Map of a primitive type

You can't mix types inside maps.

```typescript
@type({ map: "number" })
mapOfNumbers: MapSchema<number>;

@type({ map: "string" })
mapOfStrings: MapSchema<string>;
```

### Reflection

The Schema definitions can encode itself through `Reflection`. You can have the
definition implementation in the server-side, and just send the encoded
reflection to the client-side, for example:

```typescript
import { Schema, type, Reflection } from "@colyseus/schema";

class MyState extends Schema {
  @type("string")
  currentTurn: string;

  // more definitions relating to more Schema types.
}

// send `encodedStateSchema` across the network
const encodedStateSchema = Reflection.encode(new MyState());

// instantiate `MyState` in the client-side, without having its definition:
const myState = Reflection.decode(encodedStateSchema);
```

### Planned feature: Data filters (not supported atm)

On the example below, we are filtering entities which are close to the player entity.

```typescript
import { Schema, type, filter } from "@colyseus/schema";

export class State extends Schema {
  @filter(function(this: State, client: any, value: Entity) {
    const currentPlayer = this.entities[client.sessionId]

    var a = value.x - currentPlayer.x;
    var b = value.y - currentPlayer.y;

    return (Math.sqrt(a * a + b * b)) <= 10;

  })
  @type({ map: Entity })
  entities = new MapSchema<Entity>();
}
```

## Limitations and best practices

- `NaN` or `null` numbers are encoded as `0`
- `null` strings are encoded as `""`
- `Infinity` numbers are encoded as `Number.MAX_SAFE_INTEGER`
- Multi-dimensional arrays are not supported.
- Items inside Arrays and Maps must be all instance of the same type.
- `@colyseus/schema` encodes only field values in the specified order.
  - Both encoder (server) and decoder (client) must have same schema definition.
  - The order of the fields must be the same.
- Avoid manipulating indexes of an array. This result in at least `2` extra bytes for each index change. **Example:** If you have an array of 20 items, and remove the first item (through `shift()`) this means `38` extra bytes to be serialized.
- Avoid moving keys of maps. As of arrays, it adds `2` extra bytes per key move.

## Decoding / Listening for changes

> TODO: describe how changes will arrive on array and map types

```typescript
import { DataChange } from "@colyseus/schema";
import { State } from "./YourStateDefinition";

const decodedState = new State();
decodedState.onChange = function(changes: DataChange[]) {
  assert.equal(changes.length, 1);
  assert.equal(changes[0].field, "fieldNumber");
  assert.equal(changes[0].value, 50);
  assert.equal(changes[0].previousValue, undefined);
}
decodedState.decode(incomingData);
```

## Generating client-side schema files (for strictly typed languages)

> If you're using JavaScript or LUA, there's no need to bother about this.
> Interpreted programming languages are able to re-build the Schema locally through the use of `Reflection`.

You can generate the client-side schema files based on the TypeScript schema definitions automatically.

```
# C#/Unity
schema-codegen ./schemas/State.ts --output ./unity-project/ --csharp

# C/C++
schema-codegen ./schemas/State.ts --output ./cpp-project/ --cpp

# Haxe
schema-codegen ./schemas/State.ts --output ./haxe-project/ --haxe
```

## Benchmarks:

| Scenario | `@colyseus/schema` | `msgpack` + `fossil-delta` |
|---|---|---|
| Initial state size (100 entities) | 2671 | 3283 |
| Updating x/y of 1 entity after initial state | 9 | 26 |
| Updating x/y of 50 entities after initial state | 342 | 684 |
| Updating x/y of 100 entities after initial state | 668 | 1529 |


## Decoder implementations

Decoders for each target language are located at [`/decoders/`](decoders). They have no third party dependencies.

## Why

Initial thoghts/assumptions, for Colyseus:
- little to no bottleneck for detecting state changes.
- have a schema definition on both server and client
- better experience on staticaly-typed languages (C#, C++)
- mutations should be cheap.

Practical Colyseus issues this should solve:
- Avoid decoding large objects that haven't been patched
- Allow to send different patches for each client
- Better developer experience on statically-typed languages

## Inspiration:

- [schemapack](https://github.com/phretaddin/schemapack/)
- [avro](https://avro.apache.org/docs/current/spec.html)
- [flatbuffers](https://google.github.io/flatbuffers/flatbuffers_white_paper.html)


## License

MIT
