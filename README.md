# Zod: Complete Guide

A practical guide to **Zod**, the TypeScript-first schema validation library: setup, all schema types, parsing, refinements, transforms, error handling, type inference, JSON Schema, and real-world integrations.

> **Version note:** This guide targets **Zod 4** (the current major version). Where Zod 3 is different, a note is added. Always check the official docs at <https://zod.dev> for your installed version.

---

## Table of Contents

1. [What is Zod?](#1-what-is-zod)
2. [Installation and Imports](#2-installation-and-imports)
3. [Quick Start](#3-quick-start)
4. [Primitives](#4-primitives)
5. [Coercion](#5-coercion)
6. [Strings](#6-strings)
7. [String Formats](#7-string-formats)
8. [Numbers and BigInt](#8-numbers-and-bigint)
9. [Dates and ISO Formats](#9-dates-and-iso-formats)
10. [Booleans and stringbool](#10-booleans-and-stringbool)
11. [Literals and Enums](#11-literals-and-enums)
12. [Objects](#12-objects)
13. [Arrays, Tuples, Sets, Maps, Records](#13-arrays-tuples-sets-maps-records)
14. [Unions, Discriminated Unions, Intersections](#14-unions-discriminated-unions-intersections)
15. [Optional, Nullable, Default, Catch](#15-optional-nullable-default-catch)
16. [Refinements and Custom Validation](#16-refinements-and-custom-validation)
17. [Transforms, Pipes, and Codecs](#17-transforms-pipes-and-codecs)
18. [Parsing Methods](#18-parsing-methods)
19. [Error Handling and Custom Messages](#19-error-handling-and-custom-messages)
20. [Type Inference](#20-type-inference)
21. [Recursive Schemas](#21-recursive-schemas)
22. [Brand, Readonly, Instanceof, File, JSON](#22-brand-readonly-instanceof-file-json)
23. [Template Literal Types](#23-template-literal-types)
24. [Functions](#24-functions)
25. [Metadata and Registries](#25-metadata-and-registries)
26. [JSON Schema](#26-json-schema)
27. [Zod Mini](#27-zod-mini)
28. [Localization](#28-localization)
29. [Real-World Use Cases](#29-real-world-use-cases)
30. [Migration from Zod 3](#30-migration-from-zod-3)
31. [Best Practices](#31-best-practices)
32. [Cheat Sheet](#32-cheat-sheet)
33. [Troubleshooting](#33-troubleshooting)

---

## 1. What is Zod?

Zod lets you **declare a schema once**, then:

- **Validate** data at runtime (API bodies, forms, env variables, config files, external APIs).
- **Infer** the static TypeScript type from the same schema (no duplicate type definitions).
- **Transform** data while parsing (trim, coerce, rename, normalize).

Key points:

- Zero dependencies, works in Node.js, Bun, Deno, browsers, and edge runtimes.
- Immutable API: every method returns a new schema.
- Works with TypeScript strict mode (required).

---

## 2. Installation and Imports

```bash
npm install zod
pnpm add zod
yarn add zod
bun add zod
deno add npm:zod
```

Requirements: **TypeScript 5.5+** and `"strict": true` in the "tsconfig.json" file.

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true
  }
}
```

Imports:

```ts
import { z } from "zod";          // Zod 4 (default in zod@4)
import * as z from "zod";         // also valid

import { z } from "zod/v4";       // explicit v4 subpath
import { z } from "zod/v3";       // legacy v3 API (available in zod@4 package)
import * as z from "zod/mini";    // tree-shakable functional API
```

---

## 3. Quick Start

```ts
import { z } from "zod";

// 1. define schema
const User = z.object({
  name: z.string().min(2),
  email: z.email(),
  age: z.number().int().positive().optional(),
});

// 2. infer type
type User = z.infer<typeof User>;
// { name: string; email: string; age?: number | undefined }

// 3. parse (throws on error)
const user = User.parse({ name: "Ana", email: "ana@example.com" });

// 4. safeParse (does not throw)
const result = User.safeParse({ name: "A", email: "bad" });

if (!result.success) {
  console.log(result.error.issues);   // array of problems
} else {
  console.log(result.data);           // typed data
}
```

---

## 4. Primitives

```ts
z.string();
z.number();
z.bigint();
z.boolean();
z.date();
z.symbol();
z.undefined();
z.null();
z.void();      // accepts undefined
z.any();       // anything (type: any)
z.unknown();   // anything (type: unknown)
z.never();     // nothing is valid
z.nan();
```

Examples:

```ts
z.string().parse("hello");        // "hello"
z.string().parse(123);            // throws ZodError
z.number().parse(42);             // 42
z.number().parse(NaN);            // throws (NaN is not accepted by z.number())
z.null().parse(null);             // null
```

---

## 5. Coercion

Coercion converts input using JavaScript built-ins (`String()`, `Number()`, `Boolean()`, `BigInt()`, `new Date()`).

```ts
z.coerce.string().parse(123);            // "123"
z.coerce.number().parse("42");           // 42
z.coerce.number().parse("");             // 0  (careful!)
z.coerce.boolean().parse("false");       // true (any truthy value is true!)
z.coerce.bigint().parse("123");          // 123n
z.coerce.date().parse("2026-01-15");     // Date object
```

Typical use: query strings and form data, where everything is a string.

```ts
const Query = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

Query.parse({ page: "2" });   // { page: 2, limit: 20 }
```

> **Warning:** `z.coerce.boolean()` is true for any non-empty string, including `"false"`. Use [`z.stringbool()`](#10-booleans-and-stringbool) instead for environment variables and form values.

---

## 6. Strings

```ts
z.string().min(5);                    // min length
z.string().max(20);                   // max length
z.string().length(10);                // exact length
z.string().regex(/^[a-z]+$/);         // regex
z.string().startsWith("https://");
z.string().endsWith(".com");
z.string().includes("@");

// transformations (these change the value)
z.string().trim();
z.string().toLowerCase();
z.string().toUpperCase();
z.string().normalize();               // Unicode normalization

// checks (these validate only)
z.string().lowercase();               // must already be lowercase
z.string().uppercase();               // must already be uppercase
```

With custom error messages:

```ts
z.string().min(8, "Password must be at least 8 characters");
z.string().min(8, { error: "Password must be at least 8 characters" });   // Zod 4
z.string({ error: "Name is required" });
```

Chaining example:

```ts
const Username = z
  .string()
  .trim()
  .toLowerCase()
  .min(3)
  .max(20)
  .regex(/^[a-z0-9_]+$/, "Only letters, numbers, and underscore");

Username.parse("  Ana_01 ");   // "ana_01"
```

---

## 7. String Formats

In Zod 4, string formats are **top-level functions**. The old method style (`z.string().email()`) still works but is deprecated.

```ts
z.email();               // email address
z.uuid();                // UUID (RFC 9562/4122, strict)
z.guid();                // any UUID-like 8-4-4-4-12 hex string (loose)
z.uuid({ version: "v4" });
z.url();                 // URL
z.httpUrl();             // http or https URL only
z.emoji();               // single emoji
z.nanoid();
z.cuid();
z.cuid2();
z.ulid();
z.ipv4();
z.ipv6();
z.cidrv4();              // e.g. "192.168.0.0/24"
z.cidrv6();
z.base64();
z.base64url();
z.e164();                // phone number, e.g. "+14155552671"
z.jwt();                 // JWT shape (does not verify signature)
z.hash("sha256");        // hex hash string: md5, sha1, sha256, sha384, sha512
z.hostname();            // hostname
```

Examples:

```ts
z.email().parse("ana@example.com");           // ok
z.email().parse("not-an-email");              // throws

z.url().parse("https://zod.dev/docs");        // ok

// restrict URL protocol and hostname
z.url({
  protocol: /^https$/,
  hostname: /^example\.com$/,
});

// custom email regex
z.email({ pattern: z.regexes.html5Email });
```

Available regexes for reuse: `z.regexes.email`, `z.regexes.html5Email`, `z.regexes.rfc5322Email`, `z.regexes.unicodeEmail`, and others.

---

## 8. Numbers and BigInt

```ts
z.number();
z.number().gt(5);            // > 5
z.number().gte(5);           // >= 5  (alias: .min(5))
z.number().lt(10);           // < 10
z.number().lte(10);          // <= 10 (alias: .max(10))
z.number().positive();       // > 0
z.number().nonnegative();    // >= 0
z.number().negative();       // < 0
z.number().nonpositive();    // <= 0
z.number().multipleOf(5);    // alias: .step(5)
z.number().finite();         // (default in v4: infinity is not allowed)
z.number().safe();           // within Number.MIN_SAFE_INTEGER..MAX_SAFE_INTEGER
```

Integer and float helpers (Zod 4):

```ts
z.int();         // safe integer
z.int32();       // -2147483648 .. 2147483647
z.uint32();      // 0 .. 4294967295
z.float32();
z.float64();
```

> In Zod 4, `z.number()` does not accept `Infinity`. `z.number().int()` is still available, but `z.int()` is the preferred form.

BigInt:

```ts
z.bigint();
z.bigint().min(5n);
z.bigint().max(100n);
z.bigint().positive();
z.int64();       // bigint in signed 64-bit range
z.uint64();
```

---

## 9. Dates and ISO Formats

```ts
z.date();                                  // JavaScript Date object (valid date)
z.date().min(new Date("2020-01-01"));
z.date().max(new Date());

z.date({ error: (issue) => (issue.input === undefined ? "Required" : "Invalid date") });
```

ISO string formats (under `z.iso`):

```ts
z.iso.date();         // "2026-01-15"
z.iso.time();         // "14:30:00"
z.iso.datetime();     // "2026-01-15T14:30:00Z"
z.iso.duration();     // "P3Y6M4DT12H30M5S"

z.iso.datetime({ offset: true });      // allow "+02:00" offsets
z.iso.datetime({ local: true });       // allow no timezone
z.iso.datetime({ precision: 3 });      // require milliseconds
z.iso.time({ precision: -1 });         // minutes only: "14:30"
```

Common pattern: ISO string in, `Date` out:

```ts
const DateFromString = z.iso.datetime().transform((s) => new Date(s));
// or use z.coerce.date()
```

---

## 10. Booleans and stringbool

```ts
z.boolean().parse(true);    // true

// env-style strings
const Flag = z.stringbool();
Flag.parse("true");    // true   (also: "1", "yes", "on", "y", "enabled")
Flag.parse("false");   // false  (also: "0", "no", "off", "n", "disabled")
Flag.parse("maybe");   // throws

// custom values
z.stringbool({
  truthy: ["yes", "true"],
  falsy: ["no", "false"],
  case: "sensitive",       // default is "insensitive"
});
```

---

## 11. Literals and Enums

### Literals

```ts
z.literal("admin");
z.literal(42);
z.literal(true);
z.literal(null);
z.literal(["red", "green", "blue"]);   // Zod 4: multiple literal values (union)
```

### Enums

```ts
const Role = z.enum(["admin", "editor", "viewer"]);
type Role = z.infer<typeof Role>;      // "admin" | "editor" | "viewer"

Role.parse("admin");                   // ok
Role.options;                          // ["admin", "editor", "viewer"]
Role.enum;                             // { admin: "admin", editor: "editor", viewer: "viewer" }
Role.enum.admin;                       // "admin"

Role.exclude(["admin"]);               // z.enum(["editor", "viewer"])
Role.extract(["admin", "editor"]);     // z.enum(["admin", "editor"])
```

TypeScript enums (Zod 4: use `z.enum` directly, `z.nativeEnum` is deprecated):

```ts
enum Status { Active = "active", Inactive = "inactive" }
const StatusSchema = z.enum(Status);

const Direction = { Up: "UP", Down: "DOWN" } as const;
const DirectionSchema = z.enum(Direction);
```

---

## 12. Objects

### Basics

```ts
const Person = z.object({
  name: z.string(),
  age: z.number(),
});

Person.parse({ name: "Ana", age: 30, extra: "ignored" });   // { name: "Ana", age: 30 }
```

By default, **unknown keys are stripped**.

### Unknown key handling

```ts
z.strictObject({ name: z.string() });   // unknown keys cause an error
z.looseObject({ name: z.string() });    // unknown keys are kept (passthrough)

// deprecated method style (still works):
Person.strict();
Person.passthrough();

// catch-all for extra keys
z.object({ name: z.string() }).catchall(z.number());
```

### Shape helpers

```ts
const User = z.object({
  id: z.uuid(),
  name: z.string(),
  email: z.email(),
  password: z.string(),
});

User.shape.name;                       // access a field schema
User.keyof();                          // z.enum(["id", "name", "email", "password"])

const PublicUser = User.omit({ password: true });
const Credentials = User.pick({ email: true, password: true });

const UserPatch = User.partial();                          // all optional
const UserPatch2 = User.partial({ name: true });           // only name optional
const UserStrict = UserPatch.required();                   // back to required
const UserStrict2 = UserPatch.required({ email: true });

const Admin = User.extend({ role: z.literal("admin") });   // add / override fields
const Merged = z.object({ ...User.shape, ...Other.shape }); // merge (v4 preferred over .merge)
```

> `.extend()` on schemas with refinements: in Zod 4.1+, use `.safeExtend()` if the schema already has refinements, so they are kept.

### Nested objects

```ts
const Order = z.object({
  id: z.string(),
  customer: z.object({
    name: z.string(),
    address: z.object({
      street: z.string(),
      city: z.string(),
    }),
  }),
});
```

---

## 13. Arrays, Tuples, Sets, Maps, Records

### Arrays

```ts
z.array(z.string());                   // string[]
z.string().array();                    // same
z.array(z.number()).min(1);            // at least 1 item
z.array(z.number()).max(10);
z.array(z.number()).length(3);
z.array(z.string()).nonempty();        // [string, ...string[]]
```

### Tuples

```ts
const Point = z.tuple([z.number(), z.number()]);        // [number, number]
const Entry = z.tuple([z.string(), z.number()], z.boolean());   // rest: [string, number, ...boolean[]]
```

### Sets and Maps

```ts
z.set(z.string());                     // Set<string>
z.set(z.number()).min(1).max(5);
z.map(z.string(), z.number());         // Map<string, number>
```

### Records

```ts
z.record(z.string(), z.number());               // Record<string, number>
z.record(z.enum(["a", "b"]), z.number());       // exhaustive: { a: number; b: number }
z.partialRecord(z.enum(["a", "b"]), z.number());// { a?: number; b?: number }
z.looseRecord(z.string(), z.number());          // keeps non-matching keys
```

> In Zod 4, `z.record()` requires **two** arguments (key and value schema).

---

## 14. Unions, Discriminated Unions, Intersections

### Union

```ts
const StringOrNumber = z.union([z.string(), z.number()]);
z.string().or(z.number());              // same

StringOrNumber.parse("a");              // ok
StringOrNumber.parse(1);                // ok
StringOrNumber.parse(true);             // throws
```

### Discriminated union (faster, better errors)

```ts
const Event = z.discriminatedUnion("type", [
  z.object({ type: z.literal("click"), x: z.number(), y: z.number() }),
  z.object({ type: z.literal("keypress"), key: z.string() }),
  z.object({ type: z.literal("scroll"), delta: z.number() }),
]);

type Event = z.infer<typeof Event>;

Event.parse({ type: "click", x: 1, y: 2 });   // ok
Event.parse({ type: "keypress" });            // error points to missing "key"
```

### Exclusive union (exactly one must match)

```ts
z.xor([z.string(), z.number()]);              // available in newer Zod 4 versions
```

### Intersection

```ts
const A = z.object({ name: z.string() });
const B = z.object({ age: z.number() });

const Both = z.intersection(A, B);
A.and(B);                                      // same
// For objects, prefer: z.object({ ...A.shape, ...B.shape })
```

---

## 15. Optional, Nullable, Default, Catch

```ts
z.string().optional();        // string | undefined
z.string().nullable();        // string | null
z.string().nullish();         // string | null | undefined
z.optional(z.string());       // function form
z.nullable(z.string());
```

### Default

```ts
z.string().default("guest");
z.number().default(() => Date.now());          // function default

z.string().default("guest").parse(undefined);  // "guest"
z.string().default("guest").parse(null);       // throws (null is not undefined)
```

In Zod 4, `.default()` short-circuits: the default value must match the **output** type and is returned as is (no parsing, no transform). Use `.prefault()` to provide a default that is parsed through the schema:

```ts
const A = z.string().trim().default("  hi ");     // result: "  hi " (not trimmed)
const B = z.string().trim().prefault("  hi ");    // result: "hi"    (parsed)
```

### Catch (fallback on any error)

```ts
const Port = z.coerce.number().int().min(1).max(65535).catch(3000);

Port.parse("abc");        // 3000
Port.parse("8080");       // 8080

// access the error
z.number().catch((ctx) => {
  console.log(ctx.error);
  return 0;
});
```

### Optional keys in objects

```ts
const S = z.object({
  a: z.string(),                 // required
  b: z.string().optional(),      // key can be missing or undefined
  c: z.string().nullable(),      // key required, value can be null
  d: z.string().default("x"),    // optional on input, always present on output
});
```

---

## 16. Refinements and Custom Validation

### `.refine()`: simple custom check

```ts
const Even = z.number().refine((n) => n % 2 === 0, {
  error: "Number must be even",
});

// async refine (use parseAsync)
const UniqueEmail = z.email().refine(async (email) => !(await emailExists(email)), {
  error: "Email already in use",
});
await UniqueEmail.parseAsync("a@b.com");
```

Refine options:

```ts
.refine(fn, {
  error: "message",
  path: ["confirmPassword"],     // attach error to a field
  abort: true,                   // stop later checks if this fails
  when: (payload) => /* only run if true */ true,
})
```

### Cross-field validation on objects

```ts
const SignUp = z
  .object({
    password: z.string().min(8),
    confirm: z.string(),
  })
  .refine((d) => d.password === d.confirm, {
    error: "Passwords do not match",
    path: ["confirm"],
  });
```

### `.check()` and `.superRefine()`: multiple custom issues

```ts
const Password = z.string().check((ctx) => {
  if (ctx.value.length < 8) {
    ctx.issues.push({
      code: "too_small",
      origin: "string",
      minimum: 8,
      inclusive: true,
      input: ctx.value,
      message: "Too short",
    });
  }
  if (!/[A-Z]/.test(ctx.value)) {
    ctx.issues.push({
      code: "custom",
      input: ctx.value,
      message: "Needs an uppercase letter",
    });
  }
});

// Zod 3 style, still supported in v4 (deprecated):
z.string().superRefine((val, ctx) => {
  if (val.length < 8) ctx.addIssue({ code: "custom", message: "Too short" });
});
```

### `z.custom()`: custom type

```ts
const PxString = z.custom<`${number}px`>((v) => typeof v === "string" && /^\d+px$/.test(v));
PxString.parse("20px");   // ok
```

### Reusable checks

```ts
const noSpaces = z.refine((s: string) => !s.includes(" "), "No spaces allowed");
z.string().check(noSpaces);   // using zod/mini style check helpers also works with core
```

---

## 17. Transforms, Pipes, and Codecs

### `.transform()`

```ts
const Length = z.string().transform((s) => s.length);
Length.parse("hello");     // 5

type In = z.input<typeof Length>;    // string
type Out = z.output<typeof Length>;  // number

// transform with ctx for custom issues
const Num = z.string().transform((val, ctx) => {
  const n = Number(val);
  if (Number.isNaN(n)) {
    ctx.issues.push({ code: "custom", message: "Not a number", input: val });
    return z.NEVER;
  }
  return n;
});
```

### `.pipe()`: chain schemas

```ts
const StringToNumber = z.string().transform(Number).pipe(z.number().int().positive());
StringToNumber.parse("42");    // 42
StringToNumber.parse("-1");    // throws
```

### `.overwrite()`: same-type transform

`.overwrite()` changes a value without changing its type, so the schema stays a `ZodString` and JSON Schema output still works.

```ts
z.number().overwrite((n) => Math.round(n));
```

### `z.preprocess()`

```ts
const TrimmedString = z.preprocess(
  (v) => (typeof v === "string" ? v.trim() : v),
  z.string().min(1),
);
```

### Codecs (bidirectional transforms, Zod 4.1+)

```ts
const DateCodec = z.codec(
  z.iso.datetime(),          // input schema (string)
  z.date(),                  // output schema (Date)
  {
    decode: (iso) => new Date(iso),
    encode: (date) => date.toISOString(),
  },
);

DateCodec.decode("2026-01-15T10:00:00Z");   // Date
DateCodec.encode(new Date());               // string

// also: .decodeAsync, .encodeAsync, .safeDecode, .safeEncode
```

Use cases: store `Date` in memory but send ISO string over the network; parse and serialize `bigint`, `URL`, or JSON.

---

## 18. Parsing Methods

| Method | Behavior |
| --- | --- |
| `.parse(data)` | Returns typed data or **throws** `ZodError` |
| `.safeParse(data)` | Returns `{ success: true, data }` or `{ success: false, error }` |
| `.parseAsync(data)` | Async version of `parse` (needed for async refine/transform) |
| `.safeParseAsync(data)` | Async version of `safeParse` |
| `.decode(data)` / `.encode(data)` | For codecs (strongly typed input) |

```ts
const schema = z.string();

try {
  schema.parse(123);
} catch (err) {
  if (err instanceof z.ZodError) {
    console.log(err.issues);
  }
}

const r = schema.safeParse(123);
if (!r.success) {
  r.error;        // ZodError
} else {
  r.data;         // string
}

// async
const data = await schema.parseAsync(input);
const r2 = await schema.safeParseAsync(input);
```

Parse options:

```ts
schema.parse(data, {
  error: (issue) => "custom message",   // per-parse error map
  reportInput: true,                    // include input in issues
  jitless: true,                        // disable JIT (e.g. for CSP restrictions)
});
```

---

## 19. Error Handling and Custom Messages

### ZodError structure

```ts
const r = z.object({ name: z.string(), age: z.number() }).safeParse({ name: 1, age: "x" });

if (!r.success) {
  r.error.issues;
  // [
  //   { code: "invalid_type", expected: "string", path: ["name"], message: "Invalid input: expected string, received number" },
  //   { code: "invalid_type", expected: "number", path: ["age"],  message: "Invalid input: expected number, received string" }
  // ]
}
```

### Formatting helpers (Zod 4 top-level functions)

```ts
z.prettifyError(r.error);
// ✖ Invalid input: expected string, received number
//   → at name
// ✖ Invalid input: expected number, received string
//   → at age

z.treeifyError(r.error);
// { errors: [], properties: { name: { errors: ["..."] }, age: { errors: ["..."] } } }

z.flattenError(r.error);
// { formErrors: [], fieldErrors: { name: ["..."], age: ["..."] } }
// great for forms
```

> `error.format()` and `error.flatten()` still work in Zod 4 but are deprecated. Use the top-level functions.

### Custom messages

```ts
// 1. string
z.string().min(5, "Too short");

// 2. object with error
z.string().min(5, { error: "Too short" });

// 3. function: full control per issue
z.string({
  error: (issue) => {
    if (issue.input === undefined) return "This field is required";
    return "Must be a string";
  },
});

// 4. on object fields
z.object({
  email: z.email({ error: "Please enter a valid email" }),
  age: z.number({ error: "Age must be a number" }).min(18, "Must be 18 or older"),
});
```

### Global custom error map

```ts
z.config({
  customError: (issue) => {
    if (issue.code === "invalid_type") return `Expected ${issue.expected}`;
    return undefined;     // fall back to default message
  },
});
```

### Per-field errors for forms

```ts
const result = Form.safeParse(values);
if (!result.success) {
  const { fieldErrors, formErrors } = z.flattenError(result.error);
  // fieldErrors.email?.[0]
}
```

---

## 20. Type Inference

```ts
const User = z.object({
  name: z.string(),
  age: z.string().transform((s) => parseInt(s, 10)),
});

type UserOutput = z.infer<typeof User>;     // { name: string; age: number }   (same as z.output)
type UserOutput2 = z.output<typeof User>;   // same
type UserInput = z.input<typeof User>;      // { name: string; age: string }
```

Use the types:

```ts
function createUser(data: z.input<typeof User>): z.output<typeof User> {
  return User.parse(data);
}
```

Generic helper functions:

```ts
function parseWith<T extends z.ZodType>(schema: T, data: unknown): z.output<T> {
  return schema.parse(data);
}
```

Extra helpers: `z.ZodType<Output, Input>` is the base type of all schemas.

---

## 21. Recursive Schemas

Zod 4 uses **getters** for recursive types, so no manual type annotation is needed:

```ts
const Category = z.object({
  name: z.string(),
  get subcategories() {
    return z.array(Category);
  },
});

type Category = z.infer<typeof Category>;
// { name: string; subcategories: Category[] }

// mutually recursive
const User = z.object({
  email: z.email(),
  get posts() { return z.array(Post); },
});
const Post = z.object({
  title: z.string(),
  get author() { return User; },
});
```

`z.lazy()` is still available:

```ts
const Node: z.ZodType<{ value: number; next?: any }> = z.lazy(() =>
  z.object({ value: z.number(), next: Node.optional() }),
);
```

> Warning: cyclic **data** (an object that references itself) causes an infinite loop in parsing.

---

## 22. Brand, Readonly, Instanceof, File, JSON

### Branded types

```ts
const UserId = z.string().uuid().brand<"UserId">();
type UserId = z.infer<typeof UserId>;     // string & z.$brand<"UserId">

const id = UserId.parse("550e8400-e29b-41d4-a716-446655440000");

function getUser(id: UserId) {}
getUser(id);           // ok
getUser("abc");        // TS error
```

### Readonly

```ts
const Config = z.object({ host: z.string() }).readonly();
// Readonly<{ host: string }> and Object.freeze is applied to parsed result
```

### instanceof

```ts
class Foo {}
z.instanceof(Foo).parse(new Foo());
```

### File

```ts
z.file()
  .min(1)                              // min bytes
  .max(5 * 1024 * 1024)                // max 5MB
  .mime(["image/png", "image/jpeg"]);
```

### JSON

```ts
z.json();     // any valid JSON value (string, number, boolean, null, array, object)

// parse a JSON string into typed data
const FromJson = <T extends z.ZodType>(schema: T) =>
  z.string().transform((s, ctx) => {
    try { return JSON.parse(s); }
    catch { ctx.issues.push({ code: "custom", message: "Invalid JSON", input: s }); return z.NEVER; }
  }).pipe(schema);
```

---

## 23. Template Literal Types

```ts
const Email = z.templateLiteral([z.string(), "@", z.string(), ".", z.enum(["com", "org"])]);
// `${string}@${string}.com` | `${string}@${string}.org`

const Price = z.templateLiteral(["$", z.number().positive()]);
Price.parse("$10");     // ok

const CssSize = z.templateLiteral([z.number(), z.enum(["px", "em", "rem", "%"])]);
CssSize.parse("12px");  // ok
```

---

## 24. Functions

In Zod 4, `z.function()` is a **function factory**, not a schema.

```ts
const myFunc = z.function({
  input: [z.string(), z.number()],
  output: z.boolean(),
});

const impl = myFunc.implement((name, age) => {
  return age > 18;          // validated input and output
});

impl("Ana", 20);            // true
impl("Ana", "x" as any);    // throws ZodError

// async
const asyncFn = z.function({ input: [z.string()], output: z.number() })
  .implementAsync(async (s) => s.length);
```

To validate that a value is **a function** only: `z.custom<Function>((v) => typeof v === "function")`.

---

## 25. Metadata and Registries

```ts
const Email = z.email().describe("User email address");        // simple description

const Name = z.string().meta({
  id: "user_name",
  title: "Name",
  description: "Full name of the user",
  examples: ["Ana Cruz"],
});

Name.meta();              // read metadata
Name.description;         // "Full name of the user"

// custom registry
const formRegistry = z.registry<{ label: string; placeholder?: string }>();
const Field = z.string();
formRegistry.add(Field, { label: "Full name" });
formRegistry.get(Field);

// global registry (used by toJSONSchema)
z.globalRegistry.add(Field, { description: "..." });
```

---

## 26. JSON Schema

```ts
const User = z.object({
  name: z.string().min(2).meta({ description: "Full name" }),
  email: z.email(),
  age: z.number().int().optional(),
});

const jsonSchema = z.toJSONSchema(User);
/*
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "name": { "type": "string", "minLength": 2, "description": "Full name" },
    "email": { "type": "string", "format": "email", "pattern": "..." },
    "age": { "type": "integer" }
  },
  "required": ["name", "email"],
  "additionalProperties": false
}
*/

// options
z.toJSONSchema(User, {
  target: "draft-7",          // "draft-2020-12" (default) | "draft-7" | "draft-4" | "openapi-3.0"
  io: "input",                // "output" (default) or "input"
  unrepresentable: "any",     // "throw" (default) or "any": handle transforms, dates, etc.
  cycles: "ref",              // handle recursive schemas
  reused: "ref",              // extract reused schemas to $defs
});

// convert all schemas in a registry
z.toJSONSchema(z.globalRegistry);
```

Use cases: OpenAPI documents, LLM structured output / tool schemas, form generators.

Some features cannot be represented in JSON Schema (transforms, `z.date()`, `z.bigint()`, custom refinements). Use `unrepresentable: "any"` or `io: "input"`.

Converting JSON Schema to Zod: check the docs for `z.fromJSONSchema()` support in your version, or use a community package.

---

## 27. Zod Mini

A smaller, tree-shakable, functional API. Good for bundle-size sensitive apps.

```ts
import * as z from "zod/mini";

const User = z.object({
  name: z.string().check(z.minLength(2), z.maxLength(50)),
  email: z.email(),
  age: z.optional(z.number().check(z.gte(18))),
  tags: z.array(z.string()),
});

type User = z.infer<typeof User>;

User.parse({ name: "Ana", email: "ana@example.com", tags: [] });

// wrappers are functions instead of methods
z.optional(z.string());
z.nullable(z.string());
z.pipe(z.string(), z.transform(Number));
z.extend(User, { role: z.string() });
z.pick(User, { name: true });
z.partial(User);
```

Available checks: `z.minLength`, `z.maxLength`, `z.length`, `z.regex`, `z.includes`, `z.startsWith`, `z.endsWith`, `z.lowercase`, `z.uppercase`, `z.gt`, `z.gte`, `z.lt`, `z.lte`, `z.positive`, `z.negative`, `z.multipleOf`, `z.minSize`, `z.maxSize`, `z.refine`, `z.overwrite`, `z.trim`, `z.toLowerCase`, `z.toUpperCase`, and more.

---

## 28. Localization

```ts
import { z } from "zod";

z.config(z.locales.en());       // English (default)
z.config(z.locales.es());       // Spanish
z.config(z.locales.fr());
z.config(z.locales.de());
z.config(z.locales.ja());
z.config(z.locales.zhCN());
// many more locales are available, see z.locales

// lazy-load a locale to save bundle size
import { en } from "zod/locales";
z.config(en());
```

---

## 29. Real-World Use Cases

### 29.1 Environment variables

```ts
// env.ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.url(),
  JWT_SECRET: z.string().min(32),
  ENABLE_CACHE: z.stringbool().default(false),
});

const parsed = EnvSchema.safeParse(process.env);

if (!parsed.success) {
  console.error("Invalid environment variables:\n" + z.prettifyError(parsed.error));
  process.exit(1);
}

export const env = parsed.data;
```

### 29.2 Express request validation middleware

```ts
import type { Request, Response, NextFunction } from "express";
import { z } from "zod";

const validate =
  <T extends z.ZodType>(schema: T) =>
  (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse({ body: req.body, query: req.query, params: req.params });
    if (!result.success) {
      return res.status(400).json({ errors: z.flattenError(result.error) });
    }
    Object.assign(req, result.data);
    next();
  };

const CreateUser = z.object({
  body: z.object({ name: z.string().min(2), email: z.email() }),
  query: z.object({}),
  params: z.object({}),
});

app.post("/users", validate(CreateUser), (req, res) => { /* ... */ });
```

### 29.3 Next.js Server Action

```ts
"use server";
import { z } from "zod";

const Schema = z.object({
  email: z.email("Invalid email"),
  message: z.string().min(10, "Message is too short"),
});

export async function sendMessage(prevState: unknown, formData: FormData) {
  const result = Schema.safeParse(Object.fromEntries(formData));

  if (!result.success) {
    return { ok: false, errors: z.flattenError(result.error).fieldErrors };
  }

  await saveMessage(result.data);
  return { ok: true };
}
```

### 29.4 React Hook Form

```bash
npm i react-hook-form @hookform/resolvers
```

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const Schema = z.object({
  email: z.email(),
  password: z.string().min(8),
});
type FormValues = z.infer<typeof Schema>;

export function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(Schema),
  });

  return (
    <form onSubmit={handleSubmit((data) => console.log(data))}>
      <input {...register("email")} />
      {errors.email && <p>{errors.email.message}</p>}
      <input type="password" {...register("password")} />
      {errors.password && <p>{errors.password.message}</p>}
      <button type="submit">Login</button>
    </form>
  );
}
```

### 29.5 tRPC

```ts
import { initTRPC } from "@trpc/server";
import { z } from "zod";

const t = initTRPC.create();

export const appRouter = t.router({
  getUser: t.procedure
    .input(z.object({ id: z.uuid() }))
    .query(({ input }) => db.user.find(input.id)),

  createUser: t.procedure
    .input(z.object({ name: z.string().min(2), email: z.email() }))
    .mutation(({ input }) => db.user.create(input)),
});
```

### 29.6 Hono validator

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";

const app = new Hono();

app.post(
  "/posts",
  zValidator("json", z.object({ title: z.string(), body: z.string() })),
  (c) => {
    const data = c.req.valid("json");
    return c.json(data);
  },
);
```

### 29.7 Validating external API responses

```ts
const Todo = z.object({
  id: z.number(),
  title: z.string(),
  completed: z.boolean(),
});

async function getTodo(id: number) {
  const res = await fetch("https://jsonplaceholder.typicode.com/todos/" + id);
  return Todo.parse(await res.json());   // typed and validated at runtime
}
```

### 29.8 LLM structured output

```ts
const Recipe = z.object({
  name: z.string(),
  ingredients: z.array(z.object({ item: z.string(), quantity: z.string() })),
  steps: z.array(z.string()),
});

const jsonSchema = z.toJSONSchema(Recipe);      // give this to the LLM tool / response_format
const recipe = Recipe.parse(JSON.parse(llmText)); // validate the response
```

### 29.9 Form data and file upload

```ts
const Upload = z.object({
  title: z.string().min(1),
  file: z
    .file()
    .max(2 * 1024 * 1024, "Max 2MB")
    .mime(["image/png", "image/jpeg"], "Only PNG or JPEG"),
});

Upload.parse({ title: "Photo", file: formData.get("file") });
```

### 29.10 Config files (JSON)

```ts
import { readFileSync } from "node:fs";

const Config = z.object({
  server: z.object({ host: z.string().default("localhost"), port: z.number().default(8080) }),
  features: z.record(z.string(), z.boolean()).default({}),
});

const config = Config.parse(JSON.parse(readFileSync("config.json", "utf8")));
```

### 29.11 Drizzle ORM

```ts
import { createInsertSchema, createSelectSchema } from "drizzle-zod";

const insertUser = createInsertSchema(users, { email: (s) => s.email() });
```

### 29.12 Discriminated API responses

```ts
const ApiResponse = z.discriminatedUnion("status", [
  z.object({ status: z.literal("ok"), data: z.unknown() }),
  z.object({ status: z.literal("error"), message: z.string(), code: z.number() }),
]);
```

### 29.13 Reusable pagination schema

```ts
const Pagination = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(["asc", "desc"]).default("asc"),
});

const Paged = <T extends z.ZodType>(item: T) =>
  z.object({
    items: z.array(item),
    total: z.number().int().nonnegative(),
    page: z.number().int(),
  });
```

---

## 30. Migration from Zod 3

| Zod 3 | Zod 4 |
| --- | --- |
| `z.string().email()` | `z.email()` (method still works, deprecated) |
| `z.string().uuid()` | `z.uuid()` |
| `z.string().url()` | `z.url()` |
| `z.string().ip()` | `z.ipv4()` / `z.ipv6()` |
| `z.string().cidr()` | `z.cidrv4()` / `z.cidrv6()` |
| `z.string().datetime()` | `z.iso.datetime()` |
| `{ message: "..." }` | `{ error: "..." }` (`message` still works, deprecated) |
| `required_error`, `invalid_type_error` | single `error` param (string or function) |
| `errorMap` | `error` function |
| `z.nativeEnum(E)` | `z.enum(E)` |
| `.strict()` / `.passthrough()` | `z.strictObject()` / `z.looseObject()` |
| `.strip()` | removed (default behavior) |
| `.merge(B)` | `.extend(B.shape)` or spread `{ ...A.shape, ...B.shape }` |
| `.deepPartial()` | removed |
| `z.record(valueSchema)` | `z.record(keySchema, valueSchema)` (two arguments required) |
| `error.format()` / `.flatten()` | `z.treeifyError()` / `z.flattenError()` |
| `error.formErrors` | `z.flattenError(error)` |
| `z.promise()` | deprecated: `await` first, then parse |
| `ZodEffects` | removed: refine/transform stay on the same schema class |
| `z.function()` as a schema | function factory with `.implement()` |
| `z.number().int()` | `z.int()` (method still works) |
| `z.number()` allows `Infinity` | not allowed |
| `.default()` parses default | `.default()` returns value as is, use `.prefault()` for old behavior |
| `z.ZodTypeAny` | `z.ZodType` |
| `ctx.addIssue()` | `ctx.issues.push()` (old still works, deprecated) |

Strategy for gradual migration: change imports to `zod/v3` for old code and `zod/v4` for new code inside the same project. A community codemod may be available (search for `zod-v3-to-v4`), but always review the results.

---

## 31. Best Practices

- Define schemas **once** in a shared file, such as "schemas.ts" file, and infer types with `z.infer`.
- Always use `safeParse` at boundaries (HTTP, forms) when you want to handle errors without exceptions.
- Validate **all** external data: request bodies, query strings, env variables, third-party API responses, `localStorage`, and `JSON.parse` results.
- Prefer `z.strictObject` for request bodies where unknown fields should be rejected.
- Use `z.coerce.*` only for string-based input (query, form, env). Do not use it blindly on JSON bodies.
- Use `z.stringbool()` instead of `z.coerce.boolean()`.
- Use `z.discriminatedUnion` instead of `z.union` for tagged objects (faster, clearer errors).
- Use `z.input` for form values and `z.output` for processed data when transforms exist.
- Do not use `z.any()` unless needed. Prefer `z.unknown()`.
- Keep refinements pure and fast. Put database checks in async refinements only when needed.
- Define separate schemas for **create**, **update** (`.partial()`), and **public** (`.omit()`) shapes instead of one schema for everything.
- Use `.meta()` and `z.toJSONSchema()` to keep API docs in sync with validation.
- Do not trust parsed data for authorization. Validation checks shape, not permissions.
- Do not create schemas inside hot loops or request handlers. Create them at module level.
- Use `zod/mini` in bundle-sensitive front-end apps.

---

## 32. Cheat Sheet

| Task | Code |
| --- | --- |
| Install | `npm i zod` |
| Import | `import { z } from "zod"` |
| String | `z.string().min(1).max(100)` |
| Email / URL / UUID | `z.email()`, `z.url()`, `z.uuid()` |
| Number / integer | `z.number()`, `z.int()` |
| From query string | `z.coerce.number()` |
| Env boolean | `z.stringbool()` |
| ISO date string | `z.iso.datetime()` |
| Enum | `z.enum(["a", "b"])` |
| Object | `z.object({ ... })` |
| Strict / loose object | `z.strictObject()`, `z.looseObject()` |
| Partial / pick / omit | `.partial()`, `.pick({})`, `.omit({})` |
| Extend | `.extend({ ... })` |
| Array | `z.array(z.string())` |
| Record | `z.record(z.string(), z.number())` |
| Tuple | `z.tuple([z.string(), z.number()])` |
| Union | `z.union([A, B])` |
| Tagged union | `z.discriminatedUnion("type", [...])` |
| Optional / nullable | `.optional()`, `.nullable()`, `.nullish()` |
| Default | `.default(value)`, `.prefault(input)` |
| Fallback | `.catch(value)` |
| Custom check | `.refine(fn, { error, path })` |
| Transform | `.transform(fn)` |
| Chain | `.pipe(schema)` |
| Brand | `.brand<"Name">()` |
| Parse (throws) | `schema.parse(data)` |
| Parse (no throw) | `schema.safeParse(data)` |
| Async parse | `schema.parseAsync(data)` |
| Infer output type | `z.infer<typeof S>` / `z.output<typeof S>` |
| Infer input type | `z.input<typeof S>` |
| Readable errors | `z.prettifyError(err)` |
| Form errors | `z.flattenError(err).fieldErrors` |
| JSON Schema | `z.toJSONSchema(schema)` |

---

## 33. Troubleshooting

| Problem | Likely cause and fix |
| --- | --- |
| Types become `unknown` or `any` | Turn on `"strict": true` in the "tsconfig.json" file. |
| `z.string().email` is deprecated | Use `z.email()` (Zod 4). |
| `z.record` expects 2 arguments | Add the key schema: `z.record(z.string(), valueSchema)`. |
| `required_error` / `invalid_type_error` not working | Use the `error` parameter (string or function). |
| `Cannot read properties of undefined` with `z.nativeEnum` | Use `z.enum(YourEnum)`. |
| `z.coerce.boolean()` always returns `true` | Use `z.stringbool()`. |
| Optional field still required in form | Empty string is not `undefined`. Transform `""` to `undefined` with `z.preprocess` or `.transform`. |
| Async refine throws "Async refinement encountered during synchronous parse" | Use `parseAsync` or `safeParseAsync`. |
| Default value is not trimmed / transformed | Zod 4 `.default()` skips parsing. Use `.prefault()`. |
| `.extend()` loses refinements | Use `.safeExtend()` or apply refinements after `.extend()`. |
| Recursive type error (TS7022) | Use getter syntax in `z.object`, or annotate with `z.ZodType<T>` for `z.lazy`. |
| `toJSONSchema` throws on transform or date | Pass `unrepresentable: "any"`, or use `io: "input"`. |
| Strict mode rejects extra fields from forms | Use `z.object` (strip) or `z.looseObject`, or remove unknown fields first. |
| Type mismatch between form values and output | Use `z.input<typeof S>` for the form type when transforms or coercion exist. |
| Slow validation with large unions | Use `z.discriminatedUnion`. |
| Error messages are not in my language | Call `z.config(z.locales.xx())` once at app startup. |

---

## Useful Links

- Official docs: <https://zod.dev>
- GitHub: <https://github.com/colinhacks/zod>
- JSON Schema docs: <https://zod.dev/json-schema>
- Migration guide: <https://zod.dev/v4/changelog>
- Ecosystem list: <https://zod.dev/ecosystem>
