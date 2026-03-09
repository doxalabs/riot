# Riot — Design Document

## Building a Valibot/Zod-style Validation Library in Zig

This document is a comprehensive guide for building **Riot**, a schema-based data validation library for Zig inspired by [Valibot](https://valibot.dev) and [Zod](https://zod.dev). It covers architecture, API design, implementation strategies, comptime techniques, limitations, trade-offs, and everything needed to build such a library from scratch.

---

## Table of Contents

1. [Philosophy & Goals](#1-philosophy--goals)
2. [Key Differences: JS/TS vs Zig](#2-key-differences-jsts-vs-zig)
3. [Architecture Overview](#3-architecture-overview)
4. [Schema Design](#4-schema-design)
   - 4.1 [The Schema Interface](#41-the-schema-interface)
   - 4.2 [Primitive Schemas](#42-primitive-schemas)
   - 4.3 [Compound Schemas](#43-compound-schemas)
   - 4.4 [Special Schemas](#44-special-schemas)
5. [Pipelines & Actions](#5-pipelines--actions)
   - 5.1 [Validation Actions](#51-validation-actions)
   - 5.2 [Transformation Actions](#52-transformation-actions)
   - 5.3 [Custom Actions](#53-custom-actions)
6. [Parsing & Validation Methods](#6-parsing--validation-methods)
7. [Error Handling & Issues](#7-error-handling--issues)
8. [Struct Methods (pick, omit, partial, etc.)](#8-struct-methods-pick-omit-partial-etc)
9. [Comptime Type Inference](#9-comptime-type-inference)
10. [Allocator Strategy](#10-allocator-strategy)
11. [Advanced Patterns](#11-advanced-patterns)
    - 11.1 [Recursive Schemas](#111-recursive-schemas)
    - 11.2 [Discriminated Unions](#112-discriminated-unions)
    - 11.3 [Dependent Validation (forward)](#113-dependent-validation-forward)
    - 11.4 [Fallback Values](#114-fallback-values)
    - 11.5 [Coercion / Codecs](#115-coercion--codecs)
    - 11.6 [JSON Integration](#116-json-integration)
12. [Input Sources](#12-input-sources)
13. [Testing Strategy](#13-testing-strategy)
14. [Limitations](#14-limitations)
15. [Pros & Cons](#15-pros--cons)
16. [Comparison with Valibot & Zod](#16-comparison-with-valibot--zod)
17. [Full API Reference Sketch](#17-full-api-reference-sketch)
18. [Example: Complete Usage](#18-example-complete-usage)

---

## 1. Philosophy & Goals

### Inspiration

Valibot and Zod both follow the same core idea: **define a schema that describes your data, then parse unknown input against it to get validated, typed output**. Valibot favors a modular, functional design (many small functions composed via `pipe()`), while Zod uses a fluent, method-chaining API. Riot draws from both:

- **From Valibot**: Modular architecture. Independent, composable functions. Pipeline-based validation and transformation. Small, tree-shakeable units.
- **From Zod**: Fluent method syntax where it makes sense in Zig (chained builder-style calls). Discriminated unions. Refinements. Codecs. `.parse()` / `.safeParse()` duality.

### Goals for Riot

| Goal | Description |
|------|-------------|
| **Zero-cost schemas** | All schema construction must happen at `comptime`. No runtime allocations to build a validator. |
| **Comptime type inference** | The validated output type is fully determined at compile time. Users get a concrete Zig struct/type out of `.parse()`. |
| **Modular composition** | Schemas and actions are small, independent units composed via pipes. Unused schemas are dead-code-eliminated. |
| **No dependencies** | Pure Zig. No libc. No external packages. |
| **Minimal allocations** | Most validation requires zero allocator. Allocator is only needed for operations that produce dynamically-sized output (collecting all error messages, transformations that change string length, etc.). |
| **Comprehensive** | Cover primitives, structs, arrays/slices, tagged unions, optionals, enums, and custom types. |
| **Detailed errors** | Collect all validation issues with field paths, codes, and human-readable messages. |

### Non-Goals

- **Async validation** — Zig has no built-in async runtime like JS. Async validation (e.g., "check if email exists in DB") is out of scope. Users can validate synchronously, then perform async checks separately.
- **Serialization** — Riot validates and transforms data; it is not a serialization framework. However, it should integrate cleanly with `std.json`.
- **100% API parity with Valibot/Zod** — Many Valibot/Zod features are JS-specific (e.g., `Blob`, `File`, `Promise`, `symbol`). Riot maps concepts to Zig equivalents where it makes sense.

---

## 2. Key Differences: JS/TS vs Zig

Understanding these differences is critical to the entire design. Every design decision flows from how Zig fundamentally differs from TypeScript.

| Concept | TypeScript (Valibot/Zod) | Zig (Riot) |
|---------|--------------------------|------------|
| **Type system** | Erased at runtime. Schemas exist to recover type info. | Types exist at comptime AND runtime. Schemas enforce invariants, not type recovery. |
| **Generics** | Runtime generics via class/function type params | `comptime` generics — resolved entirely at compile time, monomorphized |
| **Union types** | `string \| number` — ad-hoc, structural | Tagged unions (`union(enum)`) — nominal, must be declared |
| **Optional** | `T \| undefined` | `?T` — first-class language feature |
| **Null** | `null` is a value | `null` is the absence of an optional's value |
| **Error handling** | Exceptions / Result objects | Error unions (`!T`), error sets |
| **Objects** | Dynamic shape, structural typing | Structs — fixed shape, nominal typing |
| **Arrays** | Dynamic `Array<T>` | Slices (`[]const T`), arrays (`[N]T`) — length may or may not be comptime-known |
| **Strings** | UTF-16 `string` primitive | `[]const u8` — byte slices, UTF-8 by convention |
| **Memory** | Garbage collected | Manual / arena / allocator-based |
| **Closures** | First-class, capture anything | Function pointers + explicit context pointer, or comptime closures |
| **Reflection** | Limited (`typeof`, `instanceof`) | Powerful comptime reflection via `@typeInfo`, `@Type`, `@field` |
| **Bundle size** | Tree-shaking is a design concern | Dead code elimination is automatic; not a design concern |
| **Modularity** | Import/export, bundlers | `@import`, comptime resolution — unused decls are never analyzed |

### Key Implication

In JS, schemas exist primarily to **bridge the gap between runtime values and static types**. In Zig, the type system is already present at both comptime and runtime. Therefore, Riot's primary value is:

1. **Validating untrusted input** (network data, file contents, user input, IPC)
2. **Providing a declarative, composable API** for complex validation rules
3. **Generating the output type automatically** so users don't hand-write validation structs
4. **Detailed error reporting** with paths and messages

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                        User Code                        │
│                                                         │
│   const schema = riot.object(.{                         │
│       .name = riot.pipe(.{ riot.string(), riot.trim(),  │
│                            riot.minLength(1) }),         │
│       .age  = riot.pipe(.{ riot.int(u32),               │
│                            riot.minValue(0) }),          │
│   });                                                   │
│                                                         │
│   const result = schema.parse(raw_input, allocator);    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                   Schema Layer (comptime)                │
│                                                         │
│   Schemas are comptime values. Each schema is a struct   │
│   with a `parse` / `safeParse` method and a comptime    │
│   `Output` type declaration.                            │
│                                                         │
│   SchemaType = union(enum) {                            │
│       string, int, float, bool, optional, object,       │
│       array, union_type, enum_type, pipe, ...           │
│   }                                                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Pipeline Layer (comptime)               │
│                                                         │
│   Pipes compose a base schema with N actions.           │
│   Actions are either validators or transformers.        │
│                                                         │
│   pipe(.{ string(), trim(), email(), maxLength(255) })  │
│         ▲base      ▲transform ▲validate ▲validate       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Validation Engine (runtime)             │
│                                                         │
│   Walks the schema tree, validates input, collects      │
│   issues, applies transformations, returns Result.      │
│                                                         │
│   Result(T) = union(enum) {                             │
│       ok: T,                                            │
│       err: Issues,                                      │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
```

### Core Types

```zig
/// Every schema must expose these two things at comptime:
/// 1. An `Output` type — the Zig type that a successful parse produces
/// 2. A `parse` function — validates input and returns Result(Output)

/// The universal result type returned by safeParse
pub fn Result(comptime T: type) type {
    return union(enum) {
        ok: T,
        err: Issues,
    };
}

/// A single validation issue
pub const Issue = struct {
    path: []const PathSegment,   // e.g., .{ .field("email"), .index(0) }
    code: IssueCode,             // e.g., .invalid_type, .too_small, .custom
    message: []const u8,         // human-readable
    expected: ?[]const u8,       // optional: what was expected
    received: ?[]const u8,       // optional: what was received
};

pub const PathSegment = union(enum) {
    field: []const u8,
    index: usize,
};

pub const IssueCode = enum {
    invalid_type,
    too_small,
    too_big,
    invalid_string,
    invalid_enum_value,
    invalid_union,
    custom,
    // ... more as needed
};

pub const Issues = struct {
    items: []const Issue,
    allocator: std.mem.Allocator,

    pub fn deinit(self: Issues) void { ... }
    pub fn format(self: Issues, writer: anytype) !void { ... }
};
```

---

## 4. Schema Design

### 4.1 The Schema Interface

In Zig, there is no `interface` keyword. Instead, Riot uses **comptime duck typing**: any type that exposes the right declarations is a valid schema.

A schema must provide:

```zig
/// A valid Riot schema must have:
///
///   pub const Output: type;          — the Zig type produced on success
///   pub const Input: type;           — the Zig type accepted as input
///   pub fn safeParse(self, input: Input, ally: Allocator) Result(Output);
///
/// The schema itself is a comptime-constructed value (a struct instance).
/// It carries configuration (min length, regex pattern, field schemas, etc.)
/// as comptime fields.
```

**Why comptime struct instances, not types?**

Schemas like `minLength(5)` need to carry the `5`. In Zig, the cleanest way is to make the schema a comptime value (struct instance) whose fields store configuration. The struct type itself is generated via a generic function.

```zig
pub fn minLength(comptime n: usize) MinLengthAction(n) {
    return .{};
}

// The type is parameterized on `n` — so the value 5 is baked into the type at comptime.
fn MinLengthAction(comptime n: usize) type {
    return struct {
        pub const threshold = n;

        pub fn validate(value: []const u8) ?Issue {
            if (value.len < n) {
                return Issue{
                    .code = .too_small,
                    .message = "String must be at least " ++ comptimeIntToString(n) ++ " characters",
                    // ...
                };
            }
            return null; // valid
        }
    };
}
```

### 4.2 Primitive Schemas

These validate that the input is of the correct Zig type and optionally apply constraints.

| Riot Schema | Input Type | Output Type | Notes |
|-------------|-----------|-------------|-------|
| `riot.string()` | `[]const u8` | `[]const u8` | Validates byte slice is valid UTF-8 (optional) |
| `riot.int(T)` | `T` (e.g., `i32`, `u64`) | `T` | Any Zig integer type |
| `riot.float(T)` | `T` (e.g., `f32`, `f64`) | `T` | Any Zig float type |
| `riot.boolean()` | `bool` | `bool` | |
| `riot.enumeration(E)` | `E` | `E` | Any Zig `enum` type |

#### Implementation: `string()`

```zig
pub fn string() StringSchema {
    return .{};
}

const StringSchema = struct {
    pub const Output = []const u8;
    pub const Input = []const u8;

    pub fn safeParse(_: StringSchema, input: anytype, _: ?Allocator) Result(Output) {
        // At comptime, check that the input type is compatible
        const T = @TypeOf(input);
        if (T != []const u8 and T != []u8 and T != [*:0]const u8) {
            // This would be a comptime error in practice — type mismatch.
            // But when parsing from std.json.Value, we'd check the union tag.
            @compileError("string() expects []const u8 input");
        }
        return .{ .ok = input };
    }
};
```

#### Implementation: `int(T)`

```zig
pub fn int(comptime T: type) IntSchema(T) {
    comptime {
        const info = @typeInfo(T);
        if (info != .int) @compileError("int() requires an integer type");
    }
    return .{};
}

fn IntSchema(comptime T: type) type {
    return struct {
        pub const Output = T;
        pub const Input = T;

        pub fn safeParse(_: @This(), input: T, _: ?Allocator) Result(T) {
            return .{ .ok = input };
        }
    };
}
```

#### Implementation: `float(T)`

```zig
pub fn float(comptime T: type) FloatSchema(T) {
    comptime {
        const info = @typeInfo(T);
        if (info != .float) @compileError("float() requires a float type");
    }
    return .{};
}

fn FloatSchema(comptime T: type) type {
    return struct {
        pub const Output = T;
        pub const Input = T;

        pub fn safeParse(_: @This(), input: T, _: ?Allocator) Result(T) {
            if (std.math.isNan(input)) {
                return .{ .err = ... }; // optionally reject NaN
            }
            return .{ .ok = input };
        }
    };
}
```

#### Implementation: `boolean()`

```zig
pub fn boolean() BoolSchema {
    return .{};
}

const BoolSchema = struct {
    pub const Output = bool;
    pub const Input = bool;

    pub fn safeParse(_: BoolSchema, input: bool, _: ?Allocator) Result(bool) {
        return .{ .ok = input };
    }
};
```

#### Implementation: `enumeration(E)`

```zig
pub fn enumeration(comptime E: type) EnumSchema(E) {
    comptime {
        if (@typeInfo(E) != .@"enum") @compileError("enumeration() requires an enum type");
    }
    return .{};
}

fn EnumSchema(comptime E: type) type {
    return struct {
        pub const Output = E;
        pub const Input = E;

        pub fn safeParse(_: @This(), input: E, _: ?Allocator) Result(E) {
            return .{ .ok = input };
        }

        /// Parse from a string tag name — useful for JSON/text input.
        pub fn fromString(_: @This(), input: []const u8) Result(E) {
            inline for (std.meta.fields(E)) |f| {
                if (std.mem.eql(u8, input, f.name)) {
                    return .{ .ok = @enumFromInt(f.value) };
                }
            }
            return .{ .err = ... }; // invalid enum value
        }
    };
}
```

### 4.3 Compound Schemas

#### `object()` — Struct Validation

This is the most complex and most important schema. It validates a Zig struct or a `std.json.Value` object against a set of field schemas.

**Design:**

```zig
/// `fields` is a comptime struct literal where each field's value is a schema.
///
/// Example:
///   riot.object(.{
///       .name  = riot.string(),
///       .age   = riot.int(u32),
///       .email = riot.pipe(.{ riot.string(), riot.email() }),
///   })
///
/// The Output type is a struct with the same field names, where each field's
/// type is the corresponding schema's Output type.
pub fn object(comptime fields: anytype) ObjectSchema(@TypeOf(fields)) {
    return .{ .fields = fields };
}
```

**Comptime Output type generation:**

```zig
fn ObjectSchema(comptime FieldsDef: type) type {
    return struct {
        fields: FieldsDef,

        // Generate the output type at comptime
        pub const Output = GenerateStructType(FieldsDef);

        fn GenerateStructType(comptime Def: type) type {
            const def_fields = @typeInfo(Def).@"struct".fields;
            var struct_fields: [def_fields.len]std.builtin.Type.StructField = undefined;

            for (def_fields, 0..) |f, i| {
                const FieldSchema = f.type;
                struct_fields[i] = .{
                    .name = f.name,
                    .type = FieldSchema.Output,   // <-- pull Output from each field's schema
                    .default_value_ptr = null,
                    .is_comptime = false,
                    .alignment = 0,
                };
            }

            return @Type(.{ .@"struct" = .{
                .layout = .auto,
                .fields = &struct_fields,
                .decls = &.{},
                .is_tuple = false,
            }});
        }

        pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
            var result: Output = undefined;
            var issues = std.ArrayList(Issue).init(ally);

            inline for (@typeInfo(FieldsDef).@"struct".fields) |f| {
                const field_schema = @field(self.fields, f.name);
                const field_input = @field(input, f.name);

                switch (field_schema.safeParse(field_input, ally)) {
                    .ok => |val| @field(result, f.name) = val,
                    .err => |errs| {
                        for (errs.items) |issue| {
                            var prefixed = issue;
                            // prepend field name to path
                            prefixed.path = prependPath(.{ .field = f.name }, issue.path, ally);
                            issues.append(prefixed);
                        }
                    },
                }
            }

            if (issues.items.len > 0) {
                return .{ .err = .{ .items = issues.toOwnedSlice(), .allocator = ally } };
            }
            return .{ .ok = result };
        }
    };
}
```

#### `array()` — Slice Validation

```zig
/// Validates a slice where every element matches the given schema.
///
/// Example:
///   riot.array(riot.string())  // validates []const []const u8
///
/// Output type: []const ElementSchema.Output
pub fn array(comptime element_schema: anytype) ArraySchema(@TypeOf(element_schema)) {
    return .{ .element = element_schema };
}

fn ArraySchema(comptime ElementSchemaType: type) type {
    return struct {
        element: ElementSchemaType,
        pub const Output = []const ElementSchemaType.Output;

        pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
            var results = std.ArrayList(ElementSchemaType.Output).init(ally);
            var issues = std.ArrayList(Issue).init(ally);

            for (input, 0..) |item, i| {
                switch (self.element.safeParse(item, ally)) {
                    .ok => |val| results.append(val),
                    .err => |errs| {
                        for (errs.items) |issue| {
                            var prefixed = issue;
                            prefixed.path = prependPath(.{ .index = i }, issue.path, ally);
                            issues.append(prefixed);
                        }
                    },
                }
            }

            if (issues.items.len > 0) {
                return .{ .err = .{ .items = issues.toOwnedSlice(), .allocator = ally } };
            }
            return .{ .ok = results.toOwnedSlice() };
        }
    };
}
```

#### `tuple()` — Fixed-Size Heterogeneous Tuple

```zig
/// Validates a tuple (Zig anonymous struct used as a tuple) where each
/// positional element has its own schema.
///
/// Example:
///   riot.tuple(.{ riot.string(), riot.int(u32), riot.boolean() })
///   // Output: struct { []const u8, u32, bool }
pub fn tuple(comptime schemas: anytype) TupleSchema(@TypeOf(schemas)) {
    return .{ .schemas = schemas };
}
```

#### `map()` — Key-Value Validation

Validates a `std.StringHashMap` or similar. Zig doesn't have literal object types like JS, so this validates map-like containers.

```zig
/// Example:
///   riot.map(riot.string(), riot.int(u32))
///   // Validates StringHashMap(u32) or similar
pub fn map(comptime key_schema: anytype, comptime value_schema: anytype) MapSchema(...) {
    return .{ .key = key_schema, .value = value_schema };
}
```

### 4.4 Special Schemas

#### `optional()`

```zig
/// Wraps a schema to accept null/optional values.
/// Output type becomes `?InnerSchema.Output`.
///
/// Example:
///   riot.optional(riot.string())  // accepts ?[]const u8, outputs ?[]const u8
pub fn optional(comptime inner: anytype) OptionalSchema(@TypeOf(inner)) {
    return .{ .inner = inner };
}

fn OptionalSchema(comptime InnerType: type) type {
    return struct {
        inner: InnerType,
        pub const Output = ?InnerType.Output;

        pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
            if (input == null) {
                return .{ .ok = null };
            }
            switch (self.inner.safeParse(input.?, ally)) {
                .ok => |val| return .{ .ok = val },
                .err => |errs| return .{ .err = errs },
            }
        }
    };
}
```

#### `literal()`

```zig
/// Validates that the input is exactly equal to a comptime-known value.
///
/// Example:
///   riot.literal("admin")       // only accepts "admin"
///   riot.literal(@as(u32, 42))  // only accepts 42
pub fn literal(comptime value: anytype) LiteralSchema(@TypeOf(value), value) {
    return .{};
}
```

#### `unionOf()`

```zig
/// Validates against multiple schemas, returning the first match.
/// Analogous to Zod's `z.union()` or Valibot's `v.union()`.
///
/// Example:
///   riot.unionOf(.{ riot.string(), riot.int(u32) })
///
/// Output type: a Zig tagged union generated at comptime
pub fn unionOf(comptime schemas: anytype) UnionSchema(@TypeOf(schemas)) {
    return .{ .schemas = schemas };
}
```

**Output type generation for unions:**

Since Zig requires tagged unions to be named types, Riot generates an anonymous tagged union at comptime:

```zig
// For unionOf(.{ string(), int(u32) }), the Output is:
// union(enum) { string: []const u8, int_u32: u32 }
//
// The tag names are auto-generated from the schema types.
```

#### `picklist()`

```zig
/// Validates that a string is one of a known set of values.
/// Analogous to Valibot's `v.picklist()` or Zod's `z.enum()`.
///
/// Example:
///   riot.picklist(.{ "admin", "user", "moderator" })
///
/// Output type: an enum generated at comptime:
///   enum { admin, user, moderator }
pub fn picklist(comptime values: anytype) PicklistSchema(values) {
    return .{};
}
```

#### `custom()`

```zig
/// Validates using an arbitrary user-provided function.
///
/// Example:
///   riot.custom([]const u8, struct {
///       pub fn validate(input: []const u8) ?[]const u8 {
///           if (isPixelValue(input)) return input;
///           return null;
///       }
///   }.validate)
pub fn custom(
    comptime T: type,
    comptime validate_fn: fn (T) ?T,
) CustomSchema(T, validate_fn) {
    return .{};
}
```

#### `lazy()` — Deferred Schema Resolution

For recursive types. See [Advanced Patterns](#111-recursive-schemas).

---

## 5. Pipelines & Actions

Pipelines are the core composition mechanism. A pipeline starts with a base schema and chains zero or more actions that validate or transform the data sequentially.

### Design

```zig
/// Compose a schema with a series of actions.
///
/// The first element MUST be a schema (has `Output` and `safeParse`).
/// Remaining elements are actions (validators or transformers).
///
/// Example:
///   riot.pipe(.{
///       riot.string(),       // base schema
///       riot.trim(),         // transformation: trim whitespace
///       riot.email(),        // validation: must be email format
///       riot.maxLength(255), // validation: max 255 chars
///   })
pub fn pipe(comptime stages: anytype) PipeSchema(@TypeOf(stages)) {
    return .{ .stages = stages };
}
```

**How it works at runtime:**

```
Input
  │
  ▼
┌──────────────┐     ┌──────────┐     ┌──────────┐     ┌───────────────┐
│ string()     │────▶│ trim()   │────▶│ email()  │────▶│ maxLength(255)│
│ base schema  │     │ transform│     │ validate │     │ validate      │
└──────────────┘     └──────────┘     └──────────┘     └───────────────┘
                                                              │
                                                              ▼
                                                         Result(T)
```

**Implementation:**

```zig
fn PipeSchema(comptime StagesTuple: type) type {
    const stages_info = @typeInfo(StagesTuple).@"struct";

    return struct {
        stages: StagesTuple,

        // The Output type is determined by walking the stages:
        // - Start with the base schema's Output
        // - Each transformer may change the type
        // - Validators preserve the type
        pub const Output = ComputePipeOutput(StagesTuple);

        fn ComputePipeOutput(comptime Stages: type) type {
            const fields = @typeInfo(Stages).@"struct".fields;
            // Start with the first stage's Output
            var T: type = fields[0].type.Output;
            // Walk remaining stages — transformers can change the type
            for (fields[1..]) |f| {
                if (@hasDecl(f.type, "TransformOutput")) {
                    T = f.type.TransformOutput(T);
                }
                // Validators don't change the type
            }
            return T;
        }

        pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
            // 1. Parse base schema
            const base = self.stages[0];
            var current = switch (base.safeParse(input, ally)) {
                .ok => |v| v,
                .err => |e| return .{ .err = e },
            };

            // 2. Run each action sequentially
            inline for (self.stages[1..]) |action| {
                if (@hasDecl(@TypeOf(action), "transform")) {
                    // Transformation — modify the value
                    current = action.transform(current);
                } else if (@hasDecl(@TypeOf(action), "validate")) {
                    // Validation — check the value, return issue if invalid
                    if (action.validate(current)) |issue| {
                        // Depending on config, either collect or abort
                        return .{ .err = ... };
                    }
                }
            }

            return .{ .ok = current };
        }
    };
}
```

### 5.1 Validation Actions

These check the value and produce an issue if invalid. They **do not** change the value or its type.

#### String Validations

| Action | Description | Example |
|--------|-------------|---------|
| `minLength(n)` | String must be at least `n` bytes | `riot.minLength(1)` |
| `maxLength(n)` | String must be at most `n` bytes | `riot.maxLength(255)` |
| `length(n)` | String must be exactly `n` bytes | `riot.length(36)` |
| `email()` | Must match email regex | `riot.email()` |
| `url()` | Must be a valid URL | `riot.url()` |
| `uuid()` | Must be a UUID v4 | `riot.uuid()` |
| `regex(pattern)` | Must match a regex pattern | `riot.regex("[a-z]+")` |
| `startsWith(prefix)` | Must start with prefix | `riot.startsWith("https://")` |
| `endsWith(suffix)` | Must end with suffix | `riot.endsWith(".com")` |
| `includes(substr)` | Must contain substring | `riot.includes("@")` |
| `nonEmpty()` | Must have length > 0 | `riot.nonEmpty()` |
| `ip()` | Must be a valid IPv4 or IPv6 | `riot.ip()` |
| `ipv4()` | Must be a valid IPv4 | `riot.ipv4()` |
| `ipv6()` | Must be a valid IPv6 | `riot.ipv6()` |
| `isoDate()` | Must be ISO 8601 date | `riot.isoDate()` |
| `isoDateTime()` | Must be ISO 8601 datetime | `riot.isoDateTime()` |
| `hexColor()` | Must be a hex color `#rrggbb` | `riot.hexColor()` |
| `slug()` | Must be URL-safe slug | `riot.slug()` |
| `base64()` | Must be valid base64 | `riot.base64()` |

**Implementation pattern:**

```zig
pub fn email() EmailAction {
    return .{};
}

const EmailAction = struct {
    // No TransformOutput — this is a validator, not a transformer

    pub fn validate(_: EmailAction, value: []const u8) ?Issue {
        // Simple email validation: must contain exactly one '@',
        // with non-empty local and domain parts
        if (std.mem.indexOf(u8, value, "@")) |at_pos| {
            if (at_pos > 0 and at_pos < value.len - 1) {
                const domain = value[at_pos + 1 ..];
                if (std.mem.indexOf(u8, domain, ".")) |dot_pos| {
                    if (dot_pos > 0 and dot_pos < domain.len - 1) {
                        return null; // valid
                    }
                }
            }
        }
        return Issue{
            .code = .invalid_string,
            .message = "Invalid email address",
            .path = &.{},
            .expected = "email",
            .received = value,
        };
    }
};
```

#### Numeric Validations

| Action | Description | Example |
|--------|-------------|---------|
| `minValue(n)` | Must be ≥ n | `riot.minValue(0)` |
| `maxValue(n)` | Must be ≤ n | `riot.maxValue(150)` |
| `multipleOf(n)` | Must be divisible by n | `riot.multipleOf(5)` |
| `positive()` | Must be > 0 | `riot.positive()` |
| `negative()` | Must be < 0 | `riot.negative()` |
| `nonNegative()` | Must be ≥ 0 | `riot.nonNegative()` |
| `finite()` | Must not be inf (floats) | `riot.finite()` |
| `integer()` | Float must be whole number | `riot.integer()` |

```zig
pub fn minValue(comptime n: anytype) MinValueAction(@TypeOf(n), n) {
    return .{};
}

fn MinValueAction(comptime T: type, comptime n: T) type {
    return struct {
        pub fn validate(_: @This(), value: T) ?Issue {
            if (value < n) {
                return Issue{
                    .code = .too_small,
                    .message = "Value must be at least " ++ comptimeFmt(n),
                    // ...
                };
            }
            return null;
        }
    };
}
```

#### Array/Slice Validations

| Action | Description | Example |
|--------|-------------|---------|
| `minItems(n)` | Slice must have at least `n` elements | `riot.minItems(1)` |
| `maxItems(n)` | Slice must have at most `n` elements | `riot.maxItems(100)` |
| `itemCount(n)` | Slice must have exactly `n` elements | `riot.itemCount(3)` |

### 5.2 Transformation Actions

These **modify** the value. They may or may not change the type.

| Action | Input → Output | Description |
|--------|----------------|-------------|
| `trim()` | `[]const u8 → []const u8` | Trim whitespace (returns a sub-slice) |
| `trimStart()` | `[]const u8 → []const u8` | Trim leading whitespace |
| `trimEnd()` | `[]const u8 → []const u8` | Trim trailing whitespace |
| `toLower(ally)` | `[]const u8 → []const u8` | Convert to lowercase (allocates) |
| `toUpper(ally)` | `[]const u8 → []const u8` | Convert to uppercase (allocates) |
| `toMinValue(n)` | `T → T` | Clamp to minimum |
| `toMaxValue(n)` | `T → T` | Clamp to maximum |
| `default(val)` | `?T → T` | Replace null with default |
| `transform(fn)` | `A → B` | Arbitrary transformation |

**Implementation pattern for `trim()`:**

```zig
pub fn trim() TrimAction {
    return .{};
}

const TrimAction = struct {
    // No type change, so no TransformOutput needed

    pub fn transform(_: TrimAction, value: []const u8) []const u8 {
        return std.mem.trim(u8, value, " \t\n\r");
    }
};
```

**Implementation pattern for type-changing `transform()`:**

```zig
pub fn transform(comptime func: anytype) TransformAction(@TypeOf(func)) {
    return .{ .func = func };
}

fn TransformAction(comptime FnType: type) type {
    const fn_info = @typeInfo(FnType).@"fn";
    const InputT = fn_info.params[0].type.?;
    const OutputT = fn_info.return_type.?;

    return struct {
        func: FnType,

        pub fn TransformOutput(comptime _: type) type {
            return OutputT;
        }

        pub fn transform(self: @This(), value: InputT) OutputT {
            return self.func(value);
        }
    };
}
```

### 5.3 Custom Actions

Users can write their own validation or transformation actions using `check()` and `transform()`.

#### `check()` — Custom Validation

Analogous to Valibot's `v.check()` and Zod's `.refine()`.

```zig
/// Custom validation via user-provided function.
///
/// Example:
///   riot.pipe(.{
///       riot.string(),
///       riot.check(isValidUsername, "Invalid username"),
///   })
pub fn check(
    comptime validate_fn: anytype, // fn(T) bool
    comptime message: []const u8,
) CheckAction(@TypeOf(validate_fn), message) {
    return .{};
}

fn CheckAction(comptime FnType: type, comptime message: []const u8) type {
    const T = @typeInfo(FnType).@"fn".params[0].type.?;
    return struct {
        pub fn validate(_: @This(), value: T) ?Issue {
            if (!FnType(value)) {
                return Issue{ .code = .custom, .message = message, ... };
            }
            return null;
        }
    };
}
```

#### `transform()` — Custom Transformation

```zig
/// Custom transformation via user-provided function.
///
/// Example:
///   riot.pipe(.{
///       riot.int(i32),
///       riot.transform(struct {
///           pub fn call(v: i32) u32 {
///               return @intCast(@abs(v));
///           }
///       }.call),
///   })
///
/// Or more concisely with a simple function pointer:
///   riot.pipe(.{
///       riot.string(),
///       riot.transform(computeHash),
///   })
```

---

## 6. Parsing & Validation Methods

Riot provides four primary methods for validating data.

### `parse` — Parse or Panic

Returns the validated output directly. If validation fails, it panics (or returns an error via Zig's error union, depending on design choice).

**Option A: Error union (idiomatic Zig):**

```zig
pub fn parse(schema: anytype, input: anytype, ally: Allocator) schema.Output!error{ValidationFailed} {
    switch (schema.safeParse(input, ally)) {
        .ok => |val| return val,
        .err => |issues| {
            // In debug builds, log the issues
            std.log.err("Validation failed: {d} issues", .{issues.items.len});
            return error.ValidationFailed;
        },
    }
}
```

**Option B: Using `@panic` (mirrors Zod's throw):**

```zig
pub fn parse(schema: anytype, input: anytype, ally: Allocator) schema.Output {
    switch (schema.safeParse(input, ally)) {
        .ok => |val| return val,
        .err => |issues| {
            // format issues into a message and @panic
            @panic("Validation failed");
        },
    }
}
```

**Recommendation**: Use Option A (error union). It is idiomatic Zig and allows callers to handle errors without catching panics.

### `safeParse` — Parse and Return Result

Returns a tagged union result — never panics or errors.

```zig
pub fn safeParse(schema: anytype, input: anytype, ally: Allocator) Result(schema.Output) {
    return schema.safeParse(input, ally);
}
```

### `is` — Type Guard / Check

Returns a boolean — useful for conditionals.

```zig
pub fn is(schema: anytype, input: anytype) bool {
    // Use a null allocator or a failing allocator since we don't need error details.
    switch (schema.safeParse(input, std.heap.page_allocator)) {
        .ok => return true,
        .err => return false,
    }
}
```

### `assert` — Debug Assertion

Asserts in debug builds, no-op in release.

```zig
pub fn assert(schema: anytype, input: anytype) void {
    if (builtin.mode == .Debug) {
        if (!is(schema, input)) {
            @panic("riot.assert: validation failed");
        }
    }
}
```

### `coerce` — Parse from `std.json.Value`

This is unique to Riot. Since Zig doesn't have a universal `unknown` type like JS, parsing from JSON requires special handling.

```zig
/// Parse and validate from a std.json.Value, coercing types as needed.
///
/// Example:
///   const json = try std.json.parseFromSlice(std.json.Value, ally, raw_bytes, .{});
///   const user = try riot.coerce(UserSchema, json.value, ally);
pub fn coerce(schema: anytype, json_value: std.json.Value, ally: Allocator) Result(schema.Output) {
    // Each schema type knows how to extract itself from a json.Value
    return schema.coerceFromJson(json_value, ally);
}
```

---

## 7. Error Handling & Issues

### Issue Structure

Every validation failure produces one or more `Issue` values. An issue contains:

```zig
pub const Issue = struct {
    /// Path to the value that failed validation.
    /// Empty for root-level values.
    /// e.g., [.field("address"), .field("zip")] for nested.address.zip
    path: []const PathSegment,

    /// Machine-readable error code
    code: IssueCode,

    /// Human-readable error message
    message: []const u8,

    /// What was expected (optional, for display)
    expected: ?[]const u8 = null,

    /// What was received (optional, for display)
    received: ?[]const u8 = null,
};

pub const PathSegment = union(enum) {
    field: []const u8,
    index: usize,

    pub fn format(self: PathSegment, writer: anytype) !void {
        switch (self) {
            .field => |name| try writer.print(".{s}", .{name}),
            .index => |idx| try writer.print("[{d}]", .{idx}),
        }
    }
};

pub const IssueCode = enum {
    // Type errors
    invalid_type,

    // String-specific
    invalid_string,   // failed email, url, uuid, etc.
    too_small,        // minLength
    too_big,          // maxLength

    // Number-specific
    not_multiple_of,
    not_finite,

    // Enum
    invalid_enum_value,

    // Union
    invalid_union,

    // Array
    too_few_items,
    too_many_items,

    // Custom
    custom,
};
```

### Issues Collection

Riot collects **all** issues by default (does not abort on first failure). This mirrors Valibot's default behavior and Zod's approach.

```zig
pub const Issues = struct {
    items: []const Issue,
    allocator: Allocator,

    pub fn deinit(self: Issues) void {
        self.allocator.free(self.items);
    }

    /// Pretty-print all issues
    pub fn format(self: Issues, writer: anytype) !void {
        for (self.items) |issue| {
            try writer.print("  ", .{});
            for (issue.path) |seg| {
                try seg.format(writer);
            }
            try writer.print(": [{s}] {s}\n", .{
                @tagName(issue.code),
                issue.message,
            });
        }
    }

    /// Get the count
    pub fn count(self: Issues) usize {
        return self.items.len;
    }
};
```

### Flattening Errors

Like Valibot's `v.flatten()`, provide a utility that groups errors by field path:

```zig
/// Flatten issues into a map from dotted path strings to error messages.
///
/// Example output:
///   "email"    => ["Invalid email address"]
///   "password" => ["Too short", "Must contain a number"]
///   "address.zip" => ["Required"]
pub fn flatten(issues: Issues, ally: Allocator) std.StringHashMap([]const []const u8) {
    var result = std.StringHashMap(std.ArrayList([]const u8)).init(ally);
    for (issues.items) |issue| {
        const path_str = formatPath(issue.path, ally);
        const entry = result.getOrPut(path_str);
        if (!entry.found_existing) entry.value_ptr.* = std.ArrayList([]const u8).init(ally);
        entry.value_ptr.append(issue.message);
    }
    // ... convert ArrayLists to slices ...
    return result;
}
```

### Custom Error Messages

Every schema and action accepts an optional custom message at comptime:

```zig
// Default message
riot.minLength(8)

// Custom message
riot.minLength(8).withMessage("Password must be at least 8 characters")

// Or pass it directly
riot.minLengthMsg(8, "Password must be at least 8 characters")
```

**Implementation:**

```zig
fn MinLengthAction(comptime n: usize, comptime msg: ?[]const u8) type {
    return struct {
        pub fn validate(_: @This(), value: []const u8) ?Issue {
            if (value.len < n) {
                return Issue{
                    .code = .too_small,
                    .message = msg orelse ("String must be at least " ++ comptimeIntToString(n) ++ " characters"),
                    // ...
                };
            }
            return null;
        }
    };
}
```

### Abort-Early Mode

Like Valibot's `abortPipeEarly`, Riot supports an option to stop validation on first failure:

```zig
pub const ParseOptions = struct {
    abort_early: bool = false,       // stop on first issue
    abort_pipe_early: bool = false,  // stop pipeline on first issue (per field)
};

// Used as:
schema.safeParseWithOptions(input, ally, .{ .abort_early = true });
```

---

## 8. Struct Methods (pick, omit, partial, etc.)

These utilities create new schemas by transforming existing object schemas. They mirror TypeScript's `Pick`, `Omit`, `Partial`, and `Required` utility types (used by both Zod and Valibot).

### `pick`

Creates a new object schema with only the specified fields.

```zig
/// Example:
///   const FullUser = riot.object(.{ .name = riot.string(), .age = riot.int(u32), .email = riot.string() });
///   const NameOnly = riot.pick(FullUser, .{"name"});
///   // Output type: struct { name: []const u8 }
pub fn pick(comptime schema: anytype, comptime fields: anytype) PickedSchema(...) {
    // At comptime, iterate schema.fields and include only those in `fields`
    ...
}
```

**Implementation strategy:**

```zig
fn pick(comptime obj_schema: anytype, comptime field_names: anytype) ... {
    comptime {
        const original_fields = @typeInfo(@TypeOf(obj_schema.fields)).@"struct".fields;
        var new_fields: [field_names.len]... = undefined;
        var i: usize = 0;

        for (field_names) |name| {
            for (original_fields) |f| {
                if (std.mem.eql(u8, f.name, name)) {
                    new_fields[i] = f;
                    i += 1;
                    break;
                }
            }
        }

        return object(buildAnonymousStruct(new_fields[0..i]));
    }
}
```

### `omit`

Creates a new object schema excluding the specified fields.

```zig
/// Example:
///   const WithoutEmail = riot.omit(FullUser, .{"email"});
///   // Output type: struct { name: []const u8, age: u32 }
pub fn omit(comptime schema: anytype, comptime fields: anytype) ...
```

### `partial`

Makes all fields optional.

```zig
/// Example:
///   const PartialUser = riot.partial(FullUser);
///   // Output type: struct { name: ?[]const u8, age: ?u32, email: ?[]const u8 }
pub fn partial(comptime schema: anytype) ...

/// Or partially partial — only make specific fields optional:
///   const PartialUser = riot.partial(FullUser, .{"age", "email"});
///   // Output type: struct { name: []const u8, age: ?u32, email: ?[]const u8 }
pub fn partial(comptime schema: anytype, comptime fields: anytype) ...
```

**Implementation strategy:**

```zig
fn partial(comptime obj_schema: anytype) ... {
    comptime {
        const fields = @typeInfo(@TypeOf(obj_schema.fields)).@"struct".fields;
        // For each field schema, wrap it in optional()
        var new_fields = fields;
        for (&new_fields) |*f| {
            f.type = OptionalSchema(f.type);
        }
        return object(buildStruct(new_fields));
    }
}
```

### `required`

Inverse of `partial` — makes all optional fields required.

```zig
/// Example:
///   const RequiredUser = riot.required(PartialUser);
///   // Output type: struct { name: []const u8, age: u32, email: []const u8 }
pub fn required(comptime schema: anytype) ...
```

### `merge`

Combine two object schemas into one.

```zig
/// Example:
///   const PersonSchema = riot.object(.{ .name = riot.string() });
///   const EmployeeSchema = riot.object(.{ .role = riot.string() });
///   const EmployedPerson = riot.merge(PersonSchema, EmployeeSchema);
///   // Output: struct { name: []const u8, role: []const u8 }
pub fn merge(comptime a: anytype, comptime b: anytype) ...
```

### `extend`

Add new fields to an existing object schema.

```zig
/// Example:
///   const UserWithAvatar = riot.extend(UserSchema, .{
///       .avatar_url = riot.pipe(.{ riot.string(), riot.url() }),
///   });
pub fn extend(comptime base: anytype, comptime extra_fields: anytype) ...
```

---

## 9. Comptime Type Inference

This is the most powerful feature of Riot and the area where Zig truly shines compared to TypeScript.

### How Output Types Are Generated

In Valibot/Zod, `InferOutput<typeof schema>` uses TypeScript's type-level programming to extract the output type from a schema. In Zig, we do the same thing using `@typeInfo` and `@Type` — but at comptime, generating actual concrete types.

#### The key `@Type` technique

```zig
/// Given a tuple of (name, type) pairs, generate a struct type at comptime.
fn GenerateStruct(comptime fields: anytype) type {
    const info = @typeInfo(@TypeOf(fields)).@"struct";
    var struct_fields: [info.fields.len]std.builtin.Type.StructField = undefined;

    for (info.fields, 0..) |f, i| {
        const schema = @field(fields, f.name);
        struct_fields[i] = .{
            .name = f.name,
            .type = @TypeOf(schema).Output, // Extract Output type from each field's schema
            .default_value_ptr = null,
            .is_comptime = false,
            .alignment = @alignOf(@TypeOf(schema).Output),
        };
    }

    return @Type(.{
        .@"struct" = .{
            .layout = .auto,
            .fields = &struct_fields,
            .decls = &.{},
            .is_tuple = false,
        },
    });
}
```

### Extracting the Output Type

Users can get the output type of any schema:

```zig
const UserSchema = riot.object(.{
    .name  = riot.string(),
    .age   = riot.int(u32),
    .email = riot.optional(riot.string()),
});

// Get the output type — this is a real, concrete Zig type
const User = @TypeOf(UserSchema).Output;
// User == struct { name: []const u8, age: u32, email: ?[]const u8 }

// You can use it as a regular type anywhere:
fn processUser(user: User) void { ... }
var users: std.ArrayList(User) = .init(ally);
```

### Pipeline Type Threading

When a pipeline includes transformations that change types, the Output type is threaded through:

```zig
const Schema = riot.pipe(.{
    riot.string(),                                    // Output: []const u8
    riot.trim(),                                      // Output: []const u8 (unchanged)
    riot.transform(struct {
        fn call(s: []const u8) usize { return s.len; }
    }.call),                                          // Output: usize
    riot.minValue(@as(usize, 1)),                     // Output: usize (unchanged)
});

// @TypeOf(Schema).Output == usize
```

### Comptime Assertion Helpers

```zig
/// Assert that two schemas have the same output type (useful in tests)
pub fn assertSameOutput(comptime a: anytype, comptime b: anytype) void {
    comptime {
        if (@TypeOf(a).Output != @TypeOf(b).Output) {
            @compileError("Schema output types do not match: " ++
                @typeName(@TypeOf(a).Output) ++ " vs " ++ @typeName(@TypeOf(b).Output));
        }
    }
}
```

---

## 10. Allocator Strategy

Memory management is the biggest difference between a JS validation library and a Zig one. Here's the strategy:

### When Is an Allocator Needed?

| Operation | Allocator Needed? | Why |
|-----------|-------------------|-----|
| Simple validation (type check, range check) | **No** | No dynamic data produced |
| `trim()` | **No** | Returns a sub-slice of the input |
| Collecting error issues | **Yes** | Issues are dynamically-sized |
| `toLower()` / `toUpper()` | **Yes** | Produces a new string |
| Object parsing | **Yes** | May need to allocate issue array |
| Array parsing | **Yes** | May need to allocate result slice |
| `transform()` that allocates | **Yes** | User's transform function may allocate |
| `flatten()` | **Yes** | Builds a hash map |

### API Design for Allocators

The allocator is passed to `safeParse()` / `parse()`, not stored in the schema:

```zig
// Schemas are comptime — they carry no runtime state, including no allocator.
const schema = riot.object(.{ ... }); // comptime, no allocator

// Allocator is provided at parse time
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const ally = arena.allocator();

const result = schema.safeParse(input, ally);
```

### Recommended Allocator Pattern

Use an arena allocator for validation. This way, all issues and transformed values can be freed in one call:

```zig
fn handleRequest(raw_body: []const u8) !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const ally = arena.allocator();

    const json = try std.json.parseFromSlice(std.json.Value, ally, raw_body, .{});
    const result = riot.coerce(RequestSchema, json.value, ally);

    switch (result) {
        .ok => |data| try processRequest(data),
        .err => |issues| try sendErrorResponse(issues),
    }
    // arena.deinit() frees everything — issues, transformed strings, etc.
}
```

### No-Allocator Validation

For simple schemas that don't need allocation, provide an allocator-free path:

```zig
// When you know your schema doesn't need allocation (no object, no array,
// no error collection), you can pass a failing allocator:
const valid = riot.is(riot.pipe(.{ riot.string(), riot.minLength(1) }), some_string);
```

---

## 11. Advanced Patterns

### 11.1 Recursive Schemas

Recursive types (e.g., a tree node that contains child tree nodes) require special handling because Zig's comptime evaluation doesn't allow self-referential types in the way JS schemas can reference themselves.

**The Problem:**

```zig
// This won't work — Category references itself during comptime evaluation
const Category = riot.object(.{
    .name = riot.string(),
    .children = riot.array(Category), // ERROR: Category is not yet defined
});
```

**Solution: `lazy()`**

```zig
/// lazy() defers schema resolution to parse time.
/// It takes a function that returns a schema, allowing self-reference.
///
/// Note: The Output type must be provided explicitly since it can't be inferred.
pub fn lazy(comptime T: type, comptime schema_fn: fn () anytype) LazySchema(T) {
    return .{ .resolver = schema_fn };
}

// Usage:
const Category = riot.object(.{
    .name = riot.string(),
    .children = riot.array(riot.lazy(Category, struct {
        fn resolve() @TypeOf(CategorySchema) {
            return CategorySchema;
        }
    }.resolve)),
});
```

**Limitation**: Unlike Zod's getter-based approach, Zig requires the user to explicitly provide the Output type for recursive schemas, because comptime type inference can't handle cycles.

### 11.2 Discriminated Unions

Analogous to Zod's `z.discriminatedUnion()`. This is a performance optimization for union parsing: instead of trying each variant sequentially, read a discriminator field to determine which variant to parse.

```zig
/// Example:
///   const Shape = riot.discriminatedUnion("kind", .{
///       riot.object(.{ .kind = riot.literal("circle"),    .radius = riot.float(f64) }),
///       riot.object(.{ .kind = riot.literal("rectangle"), .width = riot.float(f64), .height = riot.float(f64) }),
///       riot.object(.{ .kind = riot.literal("triangle"),  .base = riot.float(f64), .height = riot.float(f64) }),
///   });
///
/// The output type is a tagged union:
///   union(enum) {
///       circle:    struct { kind: void, radius: f64 },
///       rectangle: struct { kind: void, width: f64, height: f64 },
///       triangle:  struct { kind: void, base: f64, height: f64 },
///   }
pub fn discriminatedUnion(comptime discriminator: []const u8, comptime variants: anytype) ... {
    // At comptime:
    // 1. Extract the literal value of each variant's discriminator field
    // 2. Build a comptime string map from discriminator value → variant index
    // 3. At runtime, read the discriminator field first, then parse only the matching variant
}
```

**Runtime parsing:**

```zig
pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
    // 1. Read discriminator field
    const disc_value = @field(input, self.discriminator);

    // 2. Look up which variant to parse (comptime switch)
    inline for (self.variants, 0..) |variant, i| {
        const expected = variant.fields[self.discriminator].literal_value;
        if (std.mem.eql(u8, disc_value, expected)) {
            return variant.safeParse(input, ally);
        }
    }

    return .{ .err = ... }; // no variant matched
}
```

### 11.3 Dependent Validation (forward)

Analogous to Valibot's `v.forward()` — validate a relationship between multiple fields and attach the error to a specific field.

```zig
/// Example: Password confirmation
///
///   const RegisterSchema = riot.pipe(.{
///       riot.object(.{
///           .password  = riot.pipe(.{ riot.string(), riot.minLength(8) }),
///           .password2 = riot.string(),
///       }),
///       riot.forward(
///           riot.check(struct {
///               fn validate(input: anytype) bool {
///                   return std.mem.eql(u8, input.password, input.password2);
///               }
///           }.validate, "Passwords do not match"),
///           .{"password2"}, // attach issue to this path
///       ),
///   });
pub fn forward(comptime action: anytype, comptime path: anytype) ForwardAction(...) {
    return .{ .action = action, .path = path };
}
```

### 11.4 Fallback Values

Like Valibot's `v.fallback()` — if validation fails, return a default value instead of an error.

```zig
/// Example:
///   const Schema = riot.fallback(riot.int(u32), 0);
///   // Invalid input returns 0 instead of an error
///
///   riot.parse(Schema, "not a number", ally) // => 0
pub fn fallback(comptime schema: anytype, comptime default: schema.Output) FallbackSchema(...) {
    return .{ .inner = schema, .default = default };
}

fn FallbackSchema(comptime InnerType: type, comptime default: InnerType.Output) type {
    return struct {
        inner: InnerType,
        pub const Output = InnerType.Output;

        pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
            switch (self.inner.safeParse(input, ally)) {
                .ok => |val| return .{ .ok = val },
                .err => return .{ .ok = default },
            }
        }
    };
}
```

### 11.5 Coercion / Codecs

Analogous to Zod's `z.codec()` — bidirectional transformations. In Zig, this is primarily useful for converting between JSON types and Zig types.

```zig
/// Example: String ↔ Integer codec
///
///   const StringToInt = riot.codec(
///       riot.string(),    // input schema (decode direction)
///       riot.int(i64),    // output schema (encode direction)
///       .{
///           .decode = struct {
///               fn decode(s: []const u8) ?i64 {
///                   return std.fmt.parseInt(i64, s, 10) catch null;
///               }
///           }.decode,
///           .encode = struct {
///               fn encode(n: i64, ally: Allocator) []const u8 {
///                   return std.fmt.allocPrint(ally, "{d}", .{n});
///               }
///           }.encode,
///       },
///   );
///
///   riot.parse(StringToInt, "42", ally) // => @as(i64, 42)
///   riot.encode(StringToInt, 42, ally)  // => "42"
pub fn codec(comptime input_schema: anytype, comptime output_schema: anytype, comptime fns: anytype) ...
```

### 11.6 JSON Integration

Since Zig's primary use of a validation library is parsing untrusted JSON (from network, files, configs), Riot should integrate tightly with `std.json`.

**Approach 1: Parse from `std.json.Value`**

```zig
const raw = "{\"name\": \"Alice\", \"age\": 30}";
const json = try std.json.parseFromSlice(std.json.Value, ally, raw, .{});
const user = try riot.coerce(UserSchema, json.value, ally);
```

This requires each schema to implement a `coerceFromJson` method that knows how to extract its type from a `std.json.Value` union:

```zig
// StringSchema.coerceFromJson:
pub fn coerceFromJson(_: @This(), val: std.json.Value, _: Allocator) Result([]const u8) {
    switch (val) {
        .string => |s| return .{ .ok = s },
        else => return .{ .err = ... }, // invalid type
    }
}

// IntSchema(i64).coerceFromJson:
pub fn coerceFromJson(_: @This(), val: std.json.Value, _: Allocator) Result(i64) {
    switch (val) {
        .integer => |n| return .{ .ok = n },
        else => return .{ .err = ... },
    }
}

// ObjectSchema.coerceFromJson:
pub fn coerceFromJson(self: @This(), val: std.json.Value, ally: Allocator) Result(Output) {
    switch (val) {
        .object => |obj| {
            // iterate fields, look up each key in obj, coerce recursively
            ...
        },
        else => return .{ .err = ... },
    }
}
```

**Approach 2: Parse from `std.json` directly into schema-typed struct**

Leverage `std.json.parseFromSlice` with the generated Output type:

```zig
/// Use std.json to parse directly into the schema's Output type,
/// then run validation actions on the result.
pub fn parseJson(comptime schema: anytype, raw: []const u8, ally: Allocator) Result(schema.Output) {
    const parsed = std.json.parseFromSlice(schema.Output, ally, raw, .{}) catch {
        return .{ .err = ... }; // JSON parse failure
    };

    // Now validate using the schema's pipeline (non-type-checking actions only)
    return schema.validateParsed(parsed.value, ally);
}
```

**Recommendation**: Implement both approaches. Approach 1 is more flexible (handles coercion, type mismatches), while Approach 2 is faster for cases where the JSON structure is already known to match.

---

## 12. Input Sources

Unlike JS where `unknown` captures everything, Zig needs to know input types. Riot should handle these input scenarios:

| Input Source | Input Type | Approach |
|---|---|---|
| Already-typed Zig structs | `struct { name: []const u8, ... }` | Direct field access via `@field` |
| `std.json.Value` | `std.json.Value` | `coerceFromJson` method |
| Raw JSON bytes | `[]const u8` | `parseJson()` wrapper |
| `std.json.Parsed(T)` | `T` | Direct access to `.value` |
| Hash maps | `std.StringHashMap(V)` | Map schema reads by key |
| Command-line args | `[]const []const u8` | Custom schemas for arg parsing |
| Environment variables | Key-value strings | Custom coercion schemas |

The schema's `safeParse` uses `anytype` for the input parameter, and comptime reflection determines how to access values:

```zig
pub fn safeParse(self: @This(), input: anytype, ally: Allocator) Result(Output) {
    const InputType = @TypeOf(input);

    if (InputType == std.json.Value) {
        return self.coerceFromJson(input, ally);
    } else if (@typeInfo(InputType) == .@"struct") {
        return self.parseStruct(input, ally);
    } else {
        @compileError("Unsupported input type: " ++ @typeName(InputType));
    }
}
```

---

## 13. Testing Strategy

### Unit Tests per Schema

Each schema type should have comprehensive tests:

```zig
test "string schema" {
    const s = riot.string();

    // Valid
    try std.testing.expectEqual(Result([]const u8){ .ok = "hello" }, s.safeParse("hello", ally));

    // Invalid type handled at comptime — this shouldn't compile:
    // s.safeParse(42, ally); // comptime error
}

test "pipe with trim and minLength" {
    const s = riot.pipe(.{ riot.string(), riot.trim(), riot.minLength(1) });

    // Valid after trim
    const result = s.safeParse("  hello  ", ally);
    try std.testing.expectEqualStrings("hello", result.ok);

    // Invalid — becomes empty after trim
    const result2 = s.safeParse("   ", ally);
    try std.testing.expect(result2 == .err);
}
```

### Integration Tests

Test full object schemas with nested fields:

```zig
test "nested object validation" {
    const AddressSchema = riot.object(.{
        .street = riot.string(),
        .zip = riot.pipe(.{ riot.string(), riot.regex("[0-9]{5}") }),
    });

    const UserSchema = riot.object(.{
        .name = riot.pipe(.{ riot.string(), riot.minLength(1) }),
        .address = AddressSchema,
    });

    // Test that errors have correct paths
    const result = UserSchema.safeParse(.{
        .name = "",
        .address = .{ .street = "Main St", .zip = "bad" },
    }, ally);

    try std.testing.expect(result == .err);
    // Expect issues at paths: .name (too short), .address.zip (regex fail)
}
```

### Comptime Tests

Verify that output types are correct:

```zig
test "output type inference" {
    const Schema = riot.object(.{
        .name = riot.string(),
        .age = riot.optional(riot.int(u32)),
    });

    const Output = @TypeOf(Schema).Output;
    comptime {
        try std.testing.expectEqual([]const u8, @TypeOf(@as(Output, undefined).name));
        try std.testing.expectEqual(?u32, @TypeOf(@as(Output, undefined).age));
    }
}
```

---

## 14. Limitations

### Fundamental Limitations (Zig language constraints)

| Limitation | Description | Workaround |
|---|---|---|
| **No runtime schema construction** | Schemas must be comptime-known. You cannot build a schema from a runtime string (e.g., JSON Schema → Riot schema). | Use `std.json.parseFromValue` for dynamic schemas, or code-generate schemas. |
| **No recursive comptime types** | Zig's comptime cannot resolve self-referential type definitions (compiling `const A = object(.{ .child = A })` fails). | Use `lazy()` with explicit Output type annotation. |
| **No closures** | Zig functions cannot capture local variables. Custom `check()` and `transform()` functions must use function pointers (possibly with a context parameter) or comptime-known struct functions. | Use `struct { fn call(...) ... }.call` pattern, or pass context explicitly. |
| **No `unknown` type** | Zig has no equivalent of JS's `unknown`. Input types must be known at comptime. Parsing "truly unknown data" requires going through `std.json.Value` or `anytype`. | Use `anytype` with comptime dispatch, or provide `coerceFromJson()` methods. |
| **Comptime string limitations** | Zig's comptime can concatenate string literals but cannot format integers into strings easily. Error messages with dynamic values (e.g., "must be at least 8") require workarounds. | Use `std.fmt.comptimePrint` or precompute messages. |
| **No async validation** | Zig has no built-in async/await runtime. Async validation (e.g., checking DB uniqueness) is impractical. | Validate synchronously; perform async checks separately. |
| **No variadic generics** | Zig doesn't have variadic type parameters. Tuples of schemas are passed as anonymous struct literals and iterated with `inline for`. | Anonymous struct tuples + `inline for` is the standard pattern. |
| **Error message limitations** | Comptime-generated error messages are limited to compile-time-known strings. Runtime-specific context (e.g., "received: 42") requires allocating formatted strings. | Pre-format at comptime where possible; format at runtime for dynamic values. |

### Design Limitations (compared to Valibot/Zod)

| Feature | Valibot/Zod | Riot | Notes |
|---|---|---|---|
| Bundle size optimization | Major design concern | Not relevant | Zig dead-code eliminates automatically |
| Async validation | ✅ `parseAsync` | ❌ Not supported | Zig has no async runtime |
| Runtime schema composition | ✅ Schemas are runtime values | ❌ Schemas are comptime-only | Fundamental Zig constraint |
| TypeScript type inference | ✅ Via TS generics | ✅ Via `@typeInfo`/`@Type` | Different mechanism, same result |
| Arbitrary JS objects | ✅ `Record<string, unknown>` | ⚠️ Limited | Zig has no structural typing for ad-hoc objects |
| `instanceof` checks | ✅ `z.instanceof(Date)` | ❌ Not applicable | Zig has no class inheritance |
| Branded/flavored types | ✅ `v.brand()` | ⚠️ Possible via comptime | Can create distinct types via wrapping |
| Internationalization of error messages | ✅ Custom message functions | ⚠️ Comptime strings only | Runtime i18n would need a different approach |
| Ecosystem integration (forms, ORMs) | ✅ Massive ecosystem | ❌ None | Zig ecosystem is young |

### Performance Limitations

| Aspect | Notes |
|---|---|
| **Compile time** | Heavy use of comptime generics and `@Type` will increase compilation time, especially for large schemas. |
| **Binary size** | Each unique schema instantiation generates monomorphized code. Many distinct schemas = larger binary. (Mitigated by Zig's dead-code elimination.) |
| **Error collection** | Collecting all issues requires allocation. In hot paths, `abort_early` mode is recommended. |

---

## 15. Pros & Cons

### Pros

| Pro | Description |
|-----|-------------|
| **Zero runtime cost for schema construction** | Schemas are pure comptime values. No runtime allocations, no vtables, no indirection to build a validator. |
| **True type safety** | Output types are real Zig types, checked by the compiler. No `as` casts, no runtime type assertions. |
| **Compiler catches schema errors** | Misusing a schema (e.g., `minLength` on an integer, or referencing a non-existent field in `pick`) is a compile error, not a runtime crash. |
| **Optimal code generation** | The compiler sees through all the comptime abstraction and generates tight, inlined validation code. `inline for` over fields = unrolled loops. |
| **No dependencies** | Pure Zig. No libc. Minimal allocator usage. Works on any target Zig supports (including WASM, embedded, etc.). |
| **Memory safety** | Zig's ownership model means validated data has clear lifetimes. No GC pauses during validation. |
| **Dead code elimination** | Unlike JS where tree-shaking is a design concern, Zig automatically eliminates unused schemas. No need for modular imports. |
| **Composable** | Schemas compose via pipes, nesting, and struct methods exactly like Valibot/Zod. |
| **Integration with `std.json`** | Natural fit for validating parsed JSON in Zig web servers, CLI tools, and config file parsers. |
| **Detailed errors with paths** | Full path information (`.address.zip[0]`) in error messages, just like Valibot/Zod. |

### Cons

| Con | Description |
|-----|-------------|
| **Steep learning curve** | Comptime generics, `@typeInfo`, `@Type`, `inline for`, and anonymous struct literals are advanced Zig features. Users need to understand these to customize or debug schemas. |
| **Compile-time errors are cryptic** | When comptime goes wrong, Zig's error messages can be hard to interpret. A typo in a field name might produce a long chain of comptime errors. |
| **No runtime dynamism** | You cannot construct a schema from runtime data (e.g., from a config file or API response). This is a fundamental limitation of the comptime approach. |
| **Verbose custom validators** | Zig's lack of closures means custom `check()` functions require the `struct { fn call(...) ... }.call` pattern, which is more verbose than JS arrow functions. |
| **Limited string processing** | Zig's string handling is low-level (`[]const u8` with manual UTF-8). Validations like `email()` or `url()` require implementing regex-like matching from scratch or using `std.mem` utilities. |
| **No ecosystem** | No form integrations, no ORM bindings, no tRPC equivalent. The library exists in isolation. |
| **Increased compile times** | Heavy comptime usage (especially `@Type` for struct generation) can slow compilation for large schema definitions. |
| **Binary size for many schemas** | Monomorphization means each unique schema creates its own machine code. Projects with hundreds of distinct schemas may see larger binaries. |
| **Error messages are static** | Comptime error messages can't include runtime values. "Expected minimum length 8, received 3" requires runtime formatting and allocation. |
| **Testing complexity** | Testing comptime code is harder. Comptime errors can't be "caught" in tests — they fail compilation entirely. Testing error paths requires separate test files or conditional compilation. |

---

## 16. Comparison with Valibot & Zod

### API Style

```
┌──────────────┬─────────────────────────────┬──────────────────────────────────┬──────────────────────────────────────┐
│ Concept      │ Valibot                     │ Zod                              │ Riot (Zig)                           │
├──────────────┼─────────────────────────────┼──────────────────────────────────┼──────────────────────────────────────┤
│ String       │ v.string()                  │ z.string()                       │ riot.string()                        │
│ Number       │ v.number()                  │ z.number()                       │ riot.int(i32) / riot.float(f64)      │
│ Boolean      │ v.boolean()                 │ z.boolean()                      │ riot.boolean()                       │
│ Object       │ v.object({...})             │ z.object({...})                  │ riot.object(.{...})                  │
│ Array        │ v.array(v.string())         │ z.array(z.string())              │ riot.array(riot.string())            │
│ Optional     │ v.optional(v.string())      │ z.string().optional()            │ riot.optional(riot.string())         │
│ Enum         │ v.picklist(["a","b"])        │ z.enum(["a","b"])                │ riot.picklist(.{"a","b"})            │
│ Literal      │ v.literal("foo")            │ z.literal("foo")                 │ riot.literal("foo")                  │
│ Union        │ v.union([...])              │ z.union([...])                   │ riot.unionOf(.{...})                 │
│ Disc. Union  │ v.variant("type", [...])    │ z.discriminatedUnion("type",[…]) │ riot.discriminatedUnion("type",.{…}) │
│ Pipeline     │ v.pipe(v.string(),v.trim()) │ z.string().trim()                │ riot.pipe(.{riot.string(),riot.trim()│
│ Parse        │ v.parse(schema, input)      │ schema.parse(input)              │ riot.parse(schema, input, ally)      │
│ Safe parse   │ v.safeParse(schema, input)  │ schema.safeParse(input)          │ schema.safeParse(input, ally)        │
│ Type infer   │ v.InferOutput<typeof S>     │ z.infer<typeof S>                │ @TypeOf(schema).Output               │
│ Pick fields  │ v.pick(schema, ["a"])       │ schema.pick({a: true})           │ riot.pick(schema, .{"a"})            │
│ Omit fields  │ v.omit(schema, ["a"])       │ schema.omit({a: true})           │ riot.omit(schema, .{"a"})            │
│ Partial      │ v.partial(schema)           │ schema.partial()                 │ riot.partial(schema)                 │
│ Refinement   │ v.check(fn, msg)            │ schema.refine(fn, msg)           │ riot.check(fn, msg)                  │
│ Transform    │ v.transform(fn)             │ schema.transform(fn)             │ riot.transform(fn)                   │
│ Fallback     │ v.fallback(schema, val)     │ schema.default(val)              │ riot.fallback(schema, val)           │
│ Codec        │ —                           │ z.codec(in, out, {dec, enc})     │ riot.codec(in, out, .{dec, enc})     │
│ Recursive    │ v.lazy(() => schema)        │ getter on object key             │ riot.lazy(Type, fn)                  │
│ Custom       │ v.custom(fn)                │ z.custom(fn)                     │ riot.custom(Type, fn)                │
└──────────────┴─────────────────────────────┴──────────────────────────────────┴──────────────────────────────────────┘
```

### Philosophical Differences

| Aspect | Valibot / Zod | Riot |
|--------|---------------|------|
| **When schemas are evaluated** | Runtime | Comptime (schemas) + Runtime (validation) |
| **Why schemas exist** | Bridge runtime ↔ types | Validate untrusted input, generate types |
| **Modularity motivation** | Smaller bundle size | Clean composition (bundle size is free) |
| **Error model** | Exceptions / Result objects | Error unions / tagged union Result |
| **Memory model** | GC handles everything | Explicit allocator, arena pattern |
| **Type output** | TypeScript type inference | `@Type` comptime struct generation |
| **Ecosystem** | Huge (forms, ORMs, tRPC) | None yet |

---

## 17. Full API Reference Sketch

### Schemas

```zig
// Primitives
riot.string() -> StringSchema
riot.int(comptime T: type) -> IntSchema(T)         // T must be an integer type
riot.float(comptime T: type) -> FloatSchema(T)      // T must be a float type
riot.boolean() -> BoolSchema
riot.enumeration(comptime E: type) -> EnumSchema(E)  // E must be an enum type

// Compound
riot.object(comptime fields: anytype) -> ObjectSchema(...)
riot.array(comptime element: anytype) -> ArraySchema(...)
riot.tuple(comptime schemas: anytype) -> TupleSchema(...)
riot.map(comptime key: anytype, comptime val: anytype) -> MapSchema(...)

// Special
riot.optional(comptime inner: anytype) -> OptionalSchema(...)
riot.nullable(comptime inner: anytype) -> NullableSchema(...)    // for ?T where null is meaningful
riot.literal(comptime value: anytype) -> LiteralSchema(...)
riot.unionOf(comptime schemas: anytype) -> UnionSchema(...)
riot.discriminatedUnion(comptime disc: []const u8, comptime variants: anytype) -> DiscUnionSchema(...)
riot.picklist(comptime values: anytype) -> PicklistSchema(...)
riot.custom(comptime T: type, comptime fn: ...) -> CustomSchema(...)
riot.lazy(comptime T: type, comptime resolver: ...) -> LazySchema(T)
riot.any() -> AnySchema    // accepts anything, Output = anyopaque (use with caution)
```

### Actions (for use in pipe)

```zig
// String validations
riot.minLength(comptime n: usize)
riot.maxLength(comptime n: usize)
riot.length(comptime n: usize)
riot.email()
riot.url()
riot.uuid()
riot.regex(comptime pattern: []const u8)
riot.startsWith(comptime prefix: []const u8)
riot.endsWith(comptime suffix: []const u8)
riot.includes(comptime substr: []const u8)
riot.nonEmpty()
riot.ip()
riot.ipv4()
riot.ipv6()
riot.isoDate()
riot.isoDateTime()
riot.hexColor()
riot.slug()
riot.base64()

// String transformations
riot.trim()
riot.trimStart()
riot.trimEnd()
riot.toLower()     // requires allocator
riot.toUpper()     // requires allocator

// Numeric validations
riot.minValue(comptime n: anytype)
riot.maxValue(comptime n: anytype)
riot.multipleOf(comptime n: anytype)
riot.positive()
riot.negative()
riot.nonNegative()
riot.finite()

// Numeric transformations
riot.toMinValue(comptime n: anytype)   // clamp
riot.toMaxValue(comptime n: anytype)   // clamp

// Array validations
riot.minItems(comptime n: usize)
riot.maxItems(comptime n: usize)
riot.itemCount(comptime n: usize)

// Custom
riot.check(comptime fn: anytype, comptime message: []const u8)
riot.transform(comptime fn: anytype)

// Composition
riot.pipe(comptime stages: anytype) -> PipeSchema(...)
```

### Methods

```zig
// Top-level functions
riot.parse(comptime schema, input, ally) -> Output!error{ValidationFailed}
riot.safeParse(comptime schema, input, ally) -> Result(Output)
riot.is(comptime schema, input) -> bool
riot.assert(comptime schema, input) -> void

// JSON-specific
riot.coerce(comptime schema, json_value: std.json.Value, ally) -> Result(Output)
riot.parseJson(comptime schema, raw: []const u8, ally) -> Result(Output)

// Codec methods
riot.decode(comptime codec, input, ally) -> Result(codec.Output)
riot.encode(comptime codec, value, ally) -> Result(codec.Input)

// Struct utilities
riot.pick(comptime schema, comptime fields) -> ObjectSchema(...)
riot.omit(comptime schema, comptime fields) -> ObjectSchema(...)
riot.partial(comptime schema) -> ObjectSchema(...)
riot.partial(comptime schema, comptime fields) -> ObjectSchema(...)
riot.required(comptime schema) -> ObjectSchema(...)
riot.merge(comptime a, comptime b) -> ObjectSchema(...)
riot.extend(comptime base, comptime extra) -> ObjectSchema(...)

// Error utilities
riot.flatten(issues: Issues, ally: Allocator) -> StringHashMap([][]const u8)

// Type extraction
@TypeOf(schema).Output   // the Zig type produced by successful parsing
```

---

## 18. Example: Complete Usage

### HTTP Server Request Validation

```zig
const std = @import("std");
const riot = @import("riot");

// ── Schema Definitions ─────────────────────────────────────────────

const AddressSchema = riot.object(.{
    .street = riot.pipe(.{ riot.string(), riot.nonEmpty() }),
    .city   = riot.pipe(.{ riot.string(), riot.nonEmpty() }),
    .state  = riot.pipe(.{ riot.string(), riot.length(2) }),
    .zip    = riot.pipe(.{ riot.string(), riot.regex("[0-9]{5}(-[0-9]{4})?") }),
});

const CreateUserSchema = riot.object(.{
    .name     = riot.pipe(.{ riot.string(), riot.trim(), riot.minLength(1), riot.maxLength(100) }),
    .email    = riot.pipe(.{ riot.string(), riot.trim(), riot.email(), riot.maxLength(255) }),
    .age      = riot.pipe(.{ riot.int(u32), riot.minValue(13), riot.maxValue(150) }),
    .role     = riot.picklist(.{ "admin", "user", "moderator" }),
    .address  = riot.optional(AddressSchema),
    .tags     = riot.pipe(.{ riot.array(riot.string()), riot.maxItems(10) }),
});

// Comptime-inferred type:
const CreateUserInput = @TypeOf(CreateUserSchema).Output;
// struct {
//     name: []const u8,
//     email: []const u8,
//     age: u32,
//     role: enum { admin, user, moderator },
//     address: ?struct { street: []const u8, city: []const u8, state: []const u8, zip: []const u8 },
//     tags: []const []const u8,
// }

// Partial schema for PATCH updates
const UpdateUserSchema = riot.partial(riot.omit(CreateUserSchema, .{"role"}));

// ── Request Handler ─────────────────────────────────────────────

fn handleCreateUser(raw_body: []const u8) !void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const ally = arena.allocator();

    // Parse JSON
    const json = std.json.parseFromSlice(std.json.Value, ally, raw_body, .{}) catch {
        return sendResponse(400, "Invalid JSON");
    };

    // Validate against schema
    switch (riot.coerce(CreateUserSchema, json.value, ally)) {
        .ok => |user| {
            // `user` is fully typed as CreateUserInput
            try createUserInDatabase(user.name, user.email, user.age);
            return sendResponse(201, "User created");
        },
        .err => |issues| {
            // Flatten errors for API response
            const flat = riot.flatten(issues, ally);

            // Build error response JSON
            var response = std.ArrayList(u8).init(ally);
            try std.json.stringify(.{
                .@"error" = "Validation failed",
                .fields = flat,
            }, .{}, response.writer());

            return sendResponse(422, response.items);
        },
    }
}

// ── Config File Validation ─────────────────────────────────────

const DatabaseConfig = riot.object(.{
    .host     = riot.pipe(.{ riot.string(), riot.nonEmpty() }),
    .port     = riot.pipe(.{ riot.int(u16), riot.minValue(1), riot.maxValue(65535) }),
    .name     = riot.pipe(.{ riot.string(), riot.nonEmpty() }),
    .pool_size = riot.pipe(.{ riot.int(u32), riot.minValue(1), riot.maxValue(100) }),
});

const AppConfig = riot.object(.{
    .database   = DatabaseConfig,
    .log_level  = riot.picklist(.{ "debug", "info", "warn", "error" }),
    .port       = riot.fallback(riot.pipe(.{ riot.int(u16), riot.minValue(1) }), 8080),
    .cors_origins = riot.pipe(.{
        riot.array(riot.pipe(.{ riot.string(), riot.url() })),
        riot.maxItems(20),
    }),
});

fn loadConfig(path: []const u8) !@TypeOf(AppConfig).Output {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const ally = arena.allocator();

    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();
    const raw = try file.readToEndAlloc(ally, 1024 * 1024);

    const json = try std.json.parseFromSlice(std.json.Value, ally, raw, .{});

    switch (riot.coerce(AppConfig, json.value, ally)) {
        .ok => |config| return config,
        .err => |issues| {
            std.log.err("Invalid config file:\n", .{});
            issues.format(std.io.getStdErr().writer()) catch {};
            return error.InvalidConfig;
        },
    }
}
```

### CLI Argument Validation

```zig
const CliArgs = riot.object(.{
    .input_file  = riot.pipe(.{ riot.string(), riot.endsWith(".csv") }),
    .output_file = riot.pipe(.{ riot.string(), riot.endsWith(".json") }),
    .verbose     = riot.fallback(riot.boolean(), false),
    .max_rows    = riot.fallback(riot.pipe(.{ riot.int(u32), riot.minValue(1) }), 1000),
});

fn parseArgs(args: anytype) !@TypeOf(CliArgs).Output {
    // ... parse from args into struct, then validate
}
```

### Form Validation with Password Confirmation

```zig
const RegisterSchema = riot.pipe(.{
    riot.object(.{
        .username = riot.pipe(.{
            riot.string(),
            riot.trim(),
            riot.minLength(3),
            riot.maxLength(20),
            riot.regex("[a-zA-Z0-9_]+"),
        }),
        .email = riot.pipe(.{ riot.string(), riot.trim(), riot.email() }),
        .password = riot.pipe(.{ riot.string(), riot.minLength(8) }),
        .password_confirm = riot.string(),
    }),
    // Cross-field validation: passwords must match
    riot.forward(
        riot.check(struct {
            fn validate(input: anytype) bool {
                return std.mem.eql(u8, input.password, input.password_confirm);
            }
        }.validate, "Passwords do not match"),
        .{"password_confirm"},
    ),
});
```

---

## Appendix A: Comptime Utility Functions

These helpers are needed throughout the implementation:

```zig
/// Format a comptime integer into a string literal
fn comptimeIntToString(comptime n: anytype) []const u8 {
    return std.fmt.comptimePrint("{d}", .{n});
}

/// Prepend a path segment to an existing path
fn prependPath(segment: PathSegment, existing: []const PathSegment, ally: Allocator) []const PathSegment {
    var new_path = ally.alloc(PathSegment, existing.len + 1);
    new_path[0] = segment;
    @memcpy(new_path[1..], existing);
    return new_path;
}

/// Check if a comptime type is a valid Riot schema (has Output and safeParse)
fn isSchema(comptime T: type) bool {
    return @hasDecl(T, "Output") and @hasDecl(T, "safeParse");
}

/// Check if a comptime type is a validation action (has validate)
fn isValidator(comptime T: type) bool {
    return @hasDecl(T, "validate");
}

/// Check if a comptime type is a transformation action (has transform)
fn isTransformer(comptime T: type) bool {
    return @hasDecl(T, "transform");
}
```

## Appendix B: File Structure

```
riot/
├── build.zig
├── build.zig.zon
├── src/
│   ├── root.zig              # Public API — re-exports everything
│   ├── schemas/
│   │   ├── string.zig
│   │   ├── int.zig
│   │   ├── float.zig
│   │   ├── boolean.zig
│   │   ├── enumeration.zig
│   │   ├── object.zig
│   │   ├── array.zig
│   │   ├── tuple.zig
│   │   ├── map.zig
│   │   ├── optional.zig
│   │   ├── literal.zig
│   │   ├── union.zig
│   │   ├── picklist.zig
│   │   ├── custom.zig
│   │   └── lazy.zig
│   ├── actions/
│   │   ├── string_validations.zig   # email, url, uuid, regex, etc.
│   │   ├── numeric_validations.zig  # minValue, maxValue, etc.
│   │   ├── array_validations.zig    # minItems, maxItems, etc.
│   │   ├── string_transforms.zig    # trim, toLower, toUpper
│   │   ├── numeric_transforms.zig   # toMinValue, toMaxValue
│   │   ├── check.zig               # custom validation
│   │   └── transform.zig           # custom transformation
│   ├── pipe.zig                     # Pipeline composition
│   ├── methods.zig                  # parse, safeParse, is, assert
│   ├── struct_methods.zig           # pick, omit, partial, merge
│   ├── coerce.zig                   # JSON coercion
│   ├── codec.zig                    # Bidirectional transforms
│   ├── issue.zig                    # Issue, IssueCode, PathSegment
│   ├── result.zig                   # Result(T) type
│   └── utils.zig                    # Comptime helpers
├── tests/
│   ├── schema_tests.zig
│   ├── pipe_tests.zig
│   ├── action_tests.zig
│   ├── object_tests.zig
│   ├── json_tests.zig
│   └── integration_tests.zig
└── DESIGN.md                        # This document
```

## Appendix C: Design Decisions Log

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Schema representation | Comptime struct instances | Runtime vtable objects, comptime types (not instances) | Struct instances can carry configuration (e.g., min length = 5) as comptime fields |
| Pipeline API | `pipe(.{ schema, action, action })` (Valibot-style) | Method chaining `schema.trim().email()` (Zod-style) | Method chaining requires each schema to know about all possible actions; pipe is more modular |
| Error model | Tagged union `Result(T)` | Error unions `!T`, panics | Result is most explicit; error unions lose issue details; panics are harsh |
| Output type extraction | `@TypeOf(schema).Output` | `riot.Infer(schema)` function | Using `@TypeOf` + decl access is idiomatic Zig |
| Allocator passing | At parse time, not in schema | Store allocator in schema | Schemas are comptime; storing runtime allocator would force them to be runtime |
| JSON integration | Dual approach (coerce + parseJson) | Only coerce from `std.json.Value` | parseJson is faster when structure is known; coerce is more flexible |
| Struct methods | Free functions `riot.pick(schema, ...)` | Methods on schema `schema.pick(...)` | Free functions are more composable and don't bloat each schema type |
| Recursive schemas | `lazy(Type, resolver_fn)` | Automatic detection | Zig comptime can't resolve cycles; explicit `lazy()` makes it clear |

---

*This document is a living design reference. Update it as the library evolves.*
