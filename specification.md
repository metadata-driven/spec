# Metadata configuration format

> **Status: draft.** This document and the canonical configuration example
> define the MDD metadata format independently of any implementation.

Created by [Denis Glazkov (@denisskin)](https://github.com/denisskin).
Licensed under [CC BY 4.0](LICENSE).

Metadata-Driven Development (MDD) is an architectural approach to designing
information systems based on metadata. The metadata describes a shared model
of the system that serves as the basis for its subsystems and applications.

The metadata document declares logical models, their attributes and queryable
sets, presentation metadata, storage sources, indexes, and predefined data.

The canonical example models Mega Game, a fictional multiplayer gaming
platform with an online store. See [`examples/canonical.yaml`](examples/canonical.yaml).

## Design principles

- Metadata is model-centered. Attributes, indexes, and predefined items
  stay with their owning model. Enums may be declared at the root, within a
  model, or inline on an attribute.
- Logical models are independent from physical storage. One logical model may
  be assembled from attributes stored in several sources.
- A model describes one element. Its `set` property names the queryable
  population of those elements. A collection is a materialized attribute value.
- Repeated objects are YAML sequences. Their declaration order is preserved.
- An item may declare an explicit `id` alongside `name`. Otherwise its one-based
  sequence position is used as its effective ID.
- `kind` is normalized by the parser and is not normally written by an author.
- Stable machine names and user-facing presentation expressions are separate.

## Document structure

```yaml
version: 1.0
name: mega_game
title: Mega Game
description: Mega Game sample application with multiplayer game servers and an online store
icon: 🕹️

domains: []
enums: []
models: []
sources: []
```

| Field | Meaning |
| --- | --- |
| `version` | Optional version of the metadata format. |
| `name` | Optional stable application name. Defaults to `project`. |
| `title` | Optional application display title. |
| `description` | Optional application description. |
| `icon` | Optional application icon. |
| `options` | Optional application settings as defined in [Common metadata object](#common-metadata-object). |
| `domains` | Optional logical classifications referenced by models and attributes. |
| `enums` | Ordered root enum declarations. |
| `models` | Ordered model declarations; this section exists only at the document root. |
| `sources` | Storage, search, file, API, composite, or other data backends. |

The order of top-level mapping keys has no meaning. The order of models,
attributes, enums, enum values, indexes, and predefined items is preserved and
may be used by editors and generators. For an item without an explicit `id`,
this order also determines its effective ID.

## Format version

The metadata format defined here is version `1.0`. The root `version` field
consists of two non-negative decimal integers separated by a dot (`major.minor`).
Both `version: 1.10` and `version: "1.10"` represent the string `"1.10"`, with
major version `1` and minor version `10`. The parser preserves the digits of the
YAML scalar. If the field is absent, the parser uses its current format version.

Parsers must check the version before interpreting metadata. A parser accepts
any minor version of a supported major version, including a later minor version.
It must reject a document that specifies an unsupported major version or a
malformed version.

A minor revision may add optional features that an earlier parser can safely
ignore. An addition or change that would cause an earlier parser to build a
different schema or change the meaning of existing metadata requires a new
major version.

## Common metadata object

Most named objects support the same presentation fields:

```yaml
name: stable_name
title: Human-readable title
description: Human-readable description
icon: 🔹
options:
  mysql:
    charset: utf8mb4
```

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | Stable identifier inside the owning scope. |
| `title` | no | Primary display value. Defaults to a humanized `name`. |
| `description` | no | Longer user-facing presentation. |
| `icon` | no | Icon name, emoji, URI, or provider-specific icon value. |
| `options` | no | Arbitrary JSON-compatible settings for schema builders and drivers. |

`options` is available on every metadata node. MDD preserves its nested values
without interpreting provider-specific keys. Schema builders may copy the
object into the corresponding `dbschema` object for consumption by a driver.

`title`, `description`, and `icon` are strings. At the document root, they are
literal values describing the application. On a model, they are literal values
describing the model definition. For other objects, presentation strings can be
literal or computed according to the context defined by the object kind. Sources
normally use literal presentation values.

Template syntax, including delimiters, expressions, variable and method names,
and their case sensitivity, is outside this specification. These rules depend
on the template engine chosen by an implementation. The examples below and
elsewhere in this document use one possible syntax for illustration; they do
not prescribe that syntax. They illustrate the Go template engine
[xtpl](https://github.com/goldic/xtpl) configured with `{...}` delimiters. Other
template engines may be used. The following model fields describe an instance.

```yaml
to_title: '{login}'
to_description: '{display_name}'
to_icon: '{type.icon}'
```

The metadata parser stores template expressions as strings without interpreting
or validating their language. An implementation may compile templates when an
application opens and evaluates them in the appropriate runtime context.

`name` is never computed. When `title` is missing or requires a runtime
context that is unavailable, a renderer humanizes `name` to produce a label
(`valid_for_user_type` becomes `Valid for user type`). Other missing common
presentation fields are empty. A template language may provide fallback
operators, and UI or control metadata may override presentation.

Names should use lowercase `snake_case`. Renaming a `name` changes metadata
identity and may require a migration. Presentation fields may change without
changing identity.

Names must be unique inside their owning scope. The parser does not infer
singular or plural variants; irregular pairs such as `person` and `people` must be
written explicitly.

## Names, sets, and paths

Model names are singular. Set names are normally plural:

```yaml
models:
  - name: user
    set: users
```

`user` identifies the model and its value type. `users` is the value of the
model's `set` property and is used as a query root. It is not a metadata object
or path. A language binding may derive a type name such as `User` from the
metadata name `user`.

Metadata lookup treats the set name as an alias for its model. Looking up
either `user` or `users` resolves to the same node, with canonical path
`user` and kind `model`.

Canonical logical paths include:

| Node                        | Path or expression      |
|-----------------------------|-------------------------|
| Model                       | `user`                  |
| Model set name (not a path) | `users`                 |
| Attribute                   | `user.login`, `user.type` |
| Attribute group             | `user.groups.general`   |
| Root enum type              | `status`                |
| Model-scoped enum type      | `user.type`             |
| Enum item                   | `user.type.admin`       |
| Predefined model item       | `user.super_admin`      |
| Named index                 | `user.indexes.location` |

Model and attribute paths use singular model identities. Queries use the set
name stored on the target model:

```text
type: user
from(users)
```

Neither source names nor set names become part of logical metadata paths.
The SQL table `mega_game.users` belongs to the `maindb` source.

## Attribute groups

Attributes may be grouped for presentation in an editor. A group is declared
inside `attrs` and contains its attributes in a nested `attrs` list:

```yaml
models:
  - name: user
    attrs:
      - name: created
        type: datetime

      - group: general
        title: General
        icon: 👤
        description: Basic user information
        attrs:
          - name: login
            type: string(25)
          - name: avatar
            type: file
```

Groups are metadata nodes with kind `group` and a path such as
`user.groups.general`. Groups and attributes are both direct children of the
model. The group references the same `user.login` node as the model, and the
attribute's parent remains `user`. A group has no child nodes. Group and
attribute order follows the YAML declaration. The model includes all its
attributes. An editor may show ungrouped attributes, including implicit
identifiers, on a default tab. Groups do not affect storage or generated
model structure.

A group declaration has `group` instead of `name`; an attribute with a `name`
may still use `group` as a custom property. Attribute names cannot contain dots
and must be unique across the model, including all groups. Group names must be
unique within the model. Groups support `title`, `icon`, `description`, and
`options`.

## Domains

Domains classify models and attributes by subject area. Their metadata paths
are based on the model and attribute names.

```yaml
domains:
  - name: shop
    title: Shop
    icon: 🛒

  - name: game
    title: Game
    icon: 🎮

  - name: website
    title: Website
    icon: 🌐
```

A model and each of its attributes can declare domain references as a single
name or an ordered list of names. `domains: game` is equivalent to
`domains: [game]`.

```yaml
models:
  - name: user
    domains: [shop, game, website]
    attrs:
      - name: login
        type: string

      - name: score
        type: int
        domains: game
```

The parser resolves either form to a list of domain objects declared at the
document root. An attribute that omits `domains` inherits the list from its
model declaration. An explicit value replaces that default for the attribute,
and `domains: []` gives the attribute no domain membership. In this example,
`user.login` belongs to all three domains and `user.score` belongs to `game`.

## Sources

A source is an abstract data provider. It may represent a relational database,
key-value store, memory database, file system, HTTP API, website, blockchain,
search engine, or another storage mechanism.

```yaml
sources:
  - name: maindb
    title: Accounts and orders
    description: Stores player accounts and storefront orders
    icon: 🗄️
    conn: 'mysql://mega_game_app@/mega_game'

  - name: realtime
    title: Realtime storage
    conn: 'redis:localhost:6379/0'
```

`conn` is an opaque provider connection string. Its scheme selects or assists
in selecting the appropriate adapter. Credentials and deployment-specific
values may later be supplied through an external configuration mechanism.

A source declaration itself has no fixed physical shape. Its adapter may combine
several resources, such as a database and an internal cache, and may eventually
reference other sources. Source composition is intentionally not specified yet.

### Authoritative storage

`model.source` is the default authoritative source for the model's set and
attributes:

```yaml
- name: user
  source: maindb
```

An attribute may override it:

```yaml
- name: online
  type: bool
  source: realtime
```

The effective source is:

```text
attribute.source ?? model.source
```

This allows one logical `user` value and one logical `users` set to be assembled
from physical records in several sources. Every physical fragment uses the same
logical identifier. Cross-source reads, writes, consistency, and transactions
depend on runtime and provider capabilities.

## Models

A model describes one logical element. Its definition can have its own
presentation in metadata tools and type selectors.

```yaml
models:
  - name: user
    title: Player account
    description: A registered Mega Game player
    icon: 👤
    to_title: '{login}'
    to_description: '{UID} @{login} ({display_name})'
    to_icon: '{type.icon}'
    source: maindb

    domains: [shop, game, website]

    set: users
    set_title: Users
    set_description: All application users
    set_icon: 👥

    attrs: []
    enums: []
    indexes: []
    items: []
```

### Model fields

| Field | Meaning |
| --- | --- |
| Common fields | `title`, `description`, and `icon` are literal presentation values for the model definition. |
| `to_title` | Optional instance title as a literal or template. |
| `to_description` | Optional instance description as a literal or template. |
| `to_icon` | Optional instance icon as a literal or template. |
| `source` | Default authoritative source. |
| `domains` | Domain name or ordered list of names. Provides the default for attributes. |
| `set` | Canonical queryable set name. |
| `set_title` | Set title. |
| `set_description` | Set description. |
| `set_icon` | Set icon. |
| `attrs` | Ordered attribute declarations. |
| `enums` | Ordered enum declarations scoped to this model. |
| `indexes` | Ordered compound or external index declarations. |
| `items` | Ordered predefined, read-only elements. |

`set`, `set_title`, `set_description`, and `set_icon` are ordinary model
properties. They do not create a nested or synthetic metadata object. An editor
may group them visually without changing the metadata graph.

### Element presentation

The optional `to_title`, `to_description`, and `to_icon` fields present a
model instance. Each field accepts a literal string or a template.

```yaml
to_title: '{login}'
to_description: '{UID} @{login} ({display_name})'
to_icon: '{type.icon}'
```

Templates are evaluated in the context of the current model instance. Access to
its attributes, references, and methods depends on the template implementation.

Use `to_description` for details about the instance, such as its status or
last update time. Omit it when the title provides enough context. Use the
model's `description` for information about the type.

A nonempty `to_title` supplies the instance title through its literal value
or template result. When `to_title` is omitted or set to an empty string,
the runtime selects a default once per model using the first applicable rule.

1. Use the instance's `title` attribute if the model declares it as a stored
   scalar string.
2. Use the instance's `name` attribute if the model declares it as a stored
   scalar string.
3. Use the instance UID, which joins the model path and instance ID with `-`.

The selected template supplies the title, including an empty result.
Resolving a UID during rendering may assign an ID and persist pending instance
data. The UID is empty if no ID is available.

The Go runtime compiles the selected expression (`{title}`, `{name}`, or
`{UID}`) with the same text parser as an explicit `to_title`. With its default
parser, `{UID}` produces a title such as `user-123`.

An omitted or empty `to_description` or `to_icon` leaves the corresponding
instance value empty. The collection has its own presentation through
`set_title`, `set_description`, and `set_icon`.

### Identifiers

The parser adds an unsigned 64-bit `id` attribute to every model that declares
`set`. Models without a set have no identifier attribute. The name `id` is
reserved in model attribute declarations. Model references, indexes, queries,
and templates can use the generated attribute.

## Attributes

Attributes are declared as an ordered list:

```yaml
attrs:
  - name: login
    title: Login
    type: string(25)
    index: unique
```

### Attribute fields

| Field | Meaning |
| --- | --- |
| Common fields | Attribute identity and presentation. |
| `type` | Complete value type expression. |
| `format` | Optional type-specific input and display pattern. |
| `unit` | Optional trailing-unit presentation. |
| `source` | Authoritative source override. |
| `domains` | Optional domain name or ordered list of names. Defaults to the model declaration's domains. |
| `index` | Shorthand single-attribute index type (`regular`, `unique`, or `fulltext`). |
| `lookup` | Query used to present selectable reference candidates. |
| `query` | Query used to compute a `set<T>` value. |
| `items` | Values declared by an inline `enum` or `enum[]`. |

### Automatic timestamps

An element model may declare scalar `datetime` attributes named `created` and
`updated`. The runtime fills an empty `created` value when the element is first
written, before `OnNew`, and leaves an explicitly supplied value unchanged. It
sets `updated` to the current time before `OnCommit` for every new or changed
element. Attributes with these names and other types have no automatic behavior.

### Attribute input presentation

An attribute uses the same presentation fields directly. In a generated input,
`title` is the input caption; the standard renderer uses it as a placeholder or
floating label. `description` is supporting or help text, `icon` is the leading
adornment, and `unit` is the trailing adornment (the right side in a
left-to-right interface).

```yaml
- name: currency
  title: Currency
  type: enum
  items:
    - name: usd
      title: USD
      icon: '$'

    - name: eur
      title: EUR
      icon: '€'

- name: amount
  title: 'Enter amount in {currency.title}'
  description: 'Order amount in {currency.title}'
  icon: '{currency.icon}'
  unit: '{currency.title}'
  type: number(12,2)
  format: '#,##0.00'
```

Attribute presentation fields are evaluated against the current owning model
element, so they can follow peer attributes and references. Literal values are
valid: `unit: 'kg'` describes a fixed unit without another property.

If `title` is absent or cannot be evaluated without an element, metadata tools
use the humanized attribute name as its definition label. An absent or empty
`icon` renders no leading adornment; an absent or empty `unit` renders no
trailing adornment.

The model layer deliberately does not distinguish a general field label from a
field placeholder. If a UI needs both, or needs different captions, that is a
view/control-level override rather than a second attribute property.

`format` describes the textual representation of the value inside the input;
`unit` is a separate UI adornment and is never appended to the stored value.
A unit or currency that affects calculations, validation, or conversion must
also exist as model data. `unit` only derives its user-facing representation.

The authoring format normally omits `kind`. The parser infers it:

| Type or structure | Normalized kind |
| --- | --- |
| Scalar, reference, or `T[]` | `KindAttr` |
| `enum` or `enum[]` declaration | `KindAttr` for the field and `KindEnum` for its type |
| `set<T>` | `KindSet` |

## Type expressions

Cardinality is part of `type`; there is no separate `multiple`, `type_kind`, or
`is_collection` property.

```text
scalar              string
reference           user
materialized list   user[]
set                 set<user>
inline enum         enum
inline enum list    enum[]
```

The intended grammar is:

```text
type          = atom | atom "[]" | "set<" reference ">"
atom          = scalar | model-path | enum-path | "enum"
reference     = model-path | enum-path
```

`set<enum>` is not a valid inline declaration. A queryable view of an existing
enum uses its path, for example `set<user.type>`.
`bool`, `json`, and `file` attributes are scalar: `bool[]`, `json[]`, and `file[]` are not supported. Binary attributes may be collections, for example `binary[]`.

### Scalar types

| Type | Meaning |
| --- | --- |
| `id` | Unsigned 64-bit identifier. |
| `bool` | Boolean value. |
| `int` | Signed integer. |
| `number` | Exact numeric value with provider defaults. |
| `number(p,s)` | Exact numeric value with precision and scale. |
| `datetime` | Date and time value. |
| `string` | String with provider defaults. |
| `string(n)` | String with a maximum length of `n` characters; `0` means no declared length limit. |
| `binary` | Binary data represented as an in-memory byte sequence, without a declared length limit. |
| `binary(n)` | Binary data with a maximum length of `n` bytes; `0` means no declared length limit. |
| `file` | Binary data accessed through a stream. |
| `json` | JSON-compatible structured value. |

Attribute length limits are specified by the author. The metadata format does not
impose a universal maximum; providers and databases may impose their own limits.

Examples:

```yaml
- name: active
  type: bool

- name: amount
  type: number(12,6)

- name: birthday
  type: datetime
  format: 'dd.MM.yyyy'

- name: created
  type: datetime

- name: login
  type: string(25)

- name: payload
  type: binary

- name: checksum
  type: binary(32)

- name: invoice
  type: file
```

### Input and display formats

`format` is a type-specific pattern used by generated fields to render a value
and parse user input. It does not change the logical type, precision, storage,
or business meaning of an attribute.

Date-time and decimal patterns use Unicode/CLDR syntax:

```yaml
- name: birthday
  type: datetime
  format: 'dd.MM.yyyy'

- name: created
  type: datetime
  format: 'dd.MM.yyyy HH:mm:ss'

- name: amount
  type: number(12,2)
  format: '#,##0.00'

- name: ratio
  type: number(8,4)
  format: '0.00%'
```

Date-time patterns are case-sensitive: `MM` means month, while `mm` means
minute. Common fields include `yyyy`, `MM`, `dd`, `HH`, `mm`, and `ss`.
Decimal patterns use `0` for a required digit, `#` for an optional digit, `,`
for grouping, and `.` for the decimal position. Runtime locale determines the
localized decimal and grouping symbols.

A `datetime` pattern containing only date fields may produce a date-only input,
while the normalized value remains `datetime`. Units and currency suffixes are
provided separately through `unit`, not encoded in `format`.

Formats should be quoted in YAML, especially number patterns beginning with
`#`. An absent `format` uses the runtime default for the value type and locale.
Invalid or type-incompatible patterns are configuration errors.

String masks, validation expressions, widget choice, timezone conversion,
and collection layout are separate concerns and are not defined by `format`.

### Model references

A model name is also its reference type:

```yaml
- name: manager
  type: user
```

The runtime exposes a `user` reference. A storage adapter persists the target
model's generated identifier and resolves it through model metadata.

### Materialized collections

The postfix `[]` creates a finite value stored as one logical attribute:

```yaml
- name: best_friends
  type: user[]

- name: tags
  type: string[]

- name: checksums
  type: binary[]
```

Its physical representation may be an array, JSON value, serialized blob, or a
provider-native equivalent. When binary collections are stored as JSON, each
byte sequence is represented as a base64 string. The length of `binary(n)[]`
applies to each byte sequence, not to the encoded string or the number of items. It is not a queryable set.

### Set attributes

`set<T>` represents a potentially large or unbounded queryable result:

```yaml
- name: orders
  type: set<order>
  query: 'from(orders) where(user={id}) order(ts)'
```

It is not stored as one attribute value. The runtime exposes query/set behavior
rather than a materialized `T[]` value.

In the canonical configuration, `order` is an ordinary root model with the
`orders` set name and a stored `user` reference. `user.orders` is a filtered
set over that population; it does not declare another storage object or imply
ownership semantics.

A set may also represent the complete population. When `query` is absent, its
root is inferred from the target model:

```yaml
- name: all_orders
  type: set<order>
```

This normalizes to `from(orders)`. Both complete and filtered sets have
`KindSet`.

## Item identity

Every object declared in an `items` sequence has a required `name` and an
effective identifier. An optional `id` sits alongside `name`.

```yaml
items:
  - name: admin
    id: 10
    title: Administrator

  - name: user
    title: User
```

`name` is the stable metadata and code identifier. It participates in paths
such as `user.type.admin` and in readable assignments such as `type: admin`.
If `id` is omitted, normalization assigns the item's one-based position in its
own sequence. The first item receives `1`, the second receives `2`, and so on.

```text
effective_id(items[i]) = items[i].id if present, otherwise uint64(i + 1)
```

An explicit ID does not affect the positional IDs of other items, and the
normalizer never searches for the next unused value. It validates uniqueness
after every effective ID has been calculated.

`name` and an explicit `id` are literal values. An explicit `id` must be
non-null. Effective IDs must be unique in their immediate `items` scope, even
when explicit and derived IDs are mixed. An item name must be unique among
children that share the same flattened metadata scope.

Changing `name` changes metadata paths and generated names. Reordering items,
or inserting or removing an item before one with a derived ID, changes that
item's data identity and may require a migration. Items whose order may change,
or whose IDs have already been persisted, should therefore declare `id`
explicitly.

For an enum, the effective ID is an unsigned 64-bit integer. Different
enums may reuse the same IDs. A readable enum assignment is resolved by `name`,
while a storage adapter persists the effective ID.

For a predefined item of a set-backed model, the effective ID is an unsigned
64-bit integer that supplies the element's generated `id` attribute. The
effective ID must be unique across the model's canonical set. The parser can
detect collisions between declared items. The source adapter must also reject
a conflicting existing row during bootstrap. Normalized item metadata exposes
the effective ID even when the raw YAML omitted the `id` key.

## Enumerations

An enumeration defines an ordered set of values. The parser represents each
root, model, or inline declaration as a `KindEnum` node. The enum node owns its
`items` and has predefined string attributes `name`, `title`, `description`,
and `icon`. The enum and its read-only items use the memory source.

### Named enums

The root `enums` sequence defines types such as `status`. A model's `enums`
sequence defines types such as `user.type`. Append the item name to the type
path to address a value, for example `status.done` or `user.type.admin`.
References to a model-scoped enum use the qualified path, including from
attributes of the owning model.

```yaml
enums:
  - name: status
    title: Statuses
    items:
      - name: created
      - name: done
      - name: canceled

models:
  - name: user
    enums:
      - name: type
        title: Account types
        items:
          - name: admin
            title: Administrator

    attrs:
      - name: type
        title: Current account type
        type: user.type

      - name: status
        type: status
```

A named enum requires a `name` and accepts its own common metadata fields.
Its optional `items` sequence follows the [item identity rules](#item-identity).
The name is a single type path segment. It starts with a lowercase ASCII
letter and may contain lowercase ASCII letters, digits, and underscores.
Singular names are conventional. Root enum names must be distinct from model
names, model set names, primitive names, and the reserved token `enum`.

The parser collects all enum definitions before resolving attribute types.
Within a model, named and inline declarations share the type namespace. Two
such declarations of `user.type` are an error. Attributes have their own
namespace, so the named enum and the attribute `user.type` can coexist.

In Go, `LookupType("user.type")` and `LookupEnum("user.type")` return the enum
definition. `LookupAttr("user.type")` returns the field. The generic
`Lookup("user.type")` resolves to the enum at this shared path.

### Inline enums

An attribute declared with `type: enum` or `type: enum[]` creates a separate
model-scoped enum node at the attribute's path. Presentation fields on the
inline declaration describe the field, and the enum uses its name as its
default title. Moving the same `items` sequence to a named declaration inside
the same model preserves the enum path, item paths, and effective IDs.

#### Scalar enum

```yaml
- name: type
  title: User type
  type: enum
  items:
    - name: admin
      title: Administrator
      icon: 🤠

    - name: user
      title: User
      icon: 👤

    - name: guest
      title: Guest
      icon: 👻
```

This declaration creates the enum type `user.type` and items such as
`user.type.admin`.

#### Enum collection

```yaml
- name: permissions
  title: Permissions
  type: enum[]
  items:
    - name: manage_users
      title: Manage users

    - name: manage_orders
      title: Manage orders
```

The declared enum type is `user.permissions`; the attribute value type is
`user.permissions[]`.

### Enum reuse

An attribute can use a root or model-scoped enum by its type path:

```yaml
- name: status
  type: status

- name: previous_type
  type: user.type

- name: inherited_permissions
  type: user.permissions[]
```

On an attribute, `items` belongs to the inline declaration forms `enum` and
`enum[]`. An attribute that references an existing enum uses the values from
that enum definition.

## Lookups and queries

### Reference lookup

`lookup` restricts or orders the candidates offered for an ordinary reference:

```yaml
- name: manager
  type: user
  lookup: 'from(users) order(login)'
```

It does not compute the stored value of `manager`.

### Collection membership in queries

Use `&=` to require **all** supplied values or `|=` to require **at least one**:

```text
from(users) where(permissions &= manage_users, manage_orders)
from(users) where(permissions |= manage_users, manage_orders)
```

The first query matches users with both permissions; the second matches users
with either permission or both. Additional collection elements do not affect
a match. A single value checks membership with either operator. Empty
collections never match, and duplicate candidates do not affect the result.

Both operators require a collection field and at least one candidate. An empty
candidate list or a scalar field is an error. Use `=` with a list to match a
scalar field against any listed value. For reference and enum collections,
candidates identify elements of the corresponding model or enum. Both operators
test element membership, not substring matching or equality of the whole
collection. Query execution remains subject to provider support.

### Set query

`query` computes a `set<T>` attribute:

```yaml
- name: orders
  type: set<order>
  query: 'from(orders) where(user={id}) order(ts)'
```

For a model, `from(...)` names the value of its `set` property, never the model
metadata path. For an enum, it may name the enum type path directly. The query
parser treats the source name as a single token: ASCII letters, digits,
underscores, hyphens, and dots are allowed; slashes are not. It does not split
the name at dots. No set metadata node is created. A query template in an
attribute is evaluated in the context of the model instance that owns the
attribute. The template syntax in the examples is illustrative.

When `set<T>` has one unambiguous canonical set, `from(...)` may be omitted:

```yaml
- name: recent_orders
  type: set<order>
  query: 'where(user={id}) order(ts)'
```

The query is kept as written in metadata. When it is executed, `orders` from
`order.set` is the default source. An explicit `from(...)` is required when
the query name cannot be inferred.

The query grammar shown here is preliminary. Regardless of its eventual parser,
the query result must be compatible with the declared `set<T>` type.

## Predefined items

`items` at model level declares elements created with the model and treated as
read-only by ordinary application operations:

```yaml
items:
  - name: super_admin
    id: 1
    title: Super administrator
    description: Built-in administrator account
    icon: 👑
    data:
      login: root
      active: true
      type: admin
      permissions: [manage_users, manage_orders]
```

The metadata path is `user.super_admin`. Its explicit `id` is both the stable
item ID and the value of the model identifier. The `data` object supplies the
remaining attribute values. The explicit ID preserves its identity when the
item moves within the sequence.

Enum values are short in an assignment because the target attribute supplies
the enum context:

```text
type: admin
    -> user.type.admin

permissions: [manage_users, manage_orders]
    -> user.permissions.manage_users
    -> user.permissions.manage_orders
```

Outside an attribute assignment context, use the full metadata path when an
enum item must be addressed unambiguously.

## Indexes

| Type | Meaning |
| --- | --- |
| `regular` | General-purpose index. Default for explicit declarations. |
| `unique` | Enforces uniqueness of the indexed values in the index's source. |
| `fulltext` | Full-text search over attributes. |

A unique index over multiple attributes enforces uniqueness of the value tuple
across records. Individual attribute values may repeat.

### Single-attribute shorthand

```yaml
- name: login
  type: string
  index: unique
```

The `index` field normalizes to a single-attribute index of the specified type
in the attribute's effective source. Omit `index` when no shorthand index is
needed. A matching explicit index on the same attribute, of the same type and
in the same effective source, takes precedence.

### Explicit indexes

```yaml
indexes:
  - attrs: [type, active]

  - name: location
    source: realtime
    attrs: [latitude, longitude]

  - name: search
    type: fulltext
    source: search
    attrs: [login, display_name]
```

| Field | Required | Meaning |
| --- | --- | --- |
| `attrs` | yes | Ordered model attribute names. |
| `name` | no | Stable model-local index name. |
| `type` | no | `regular` (default), `unique`, or `fulltext`. |
| `source` | no | Native or derived index target. |

A `session` model can require each `sid` to be unique within its `host`.

```yaml
indexes:
  - type: unique
    attrs: [host, sid]
```

Attribute order is significant. `[latitude, longitude]` and
`[longitude, latitude]` describe different indexes.

Index declarations may refer to stored attributes of any type, including JSON
and collections. Provider or database capabilities determine which index types
and attribute types can be implemented; unsupported combinations are reported
by the provider.

When `name` is omitted, normalization joins the ordered attribute names with
underscores. It adds `_unique` for a unique index and `_fulltext` for a
full-text index. For example, an index on `login` is named `login`,
`login_unique`, or `login_fulltext`, depending on its type.

Without `source`, the index is created in the common effective source of its
attributes. An explicit different source, such as `search`, describes a derived
index that the runtime or provider integration must keep synchronized.

## Normalization and validation

A conforming parser or compiler should:

1. Preserve the order of every declared sequence.
2. Require unique names inside each scope.
3. Reject an explicit `id` in model `attrs` and add the generated `id`
   attribute of type `id` to every model declaring `set`.
4. Derive a missing item ID from its one-based sequence position and validate
   the uniqueness and type of every effective ID.
5. Resolve all domain, source, model, enum, attribute, index, and model-set-name
   references without creating set metadata nodes.
6. Resolve model reference storage through the target model's generated identifier.
7. Normalize `enum` and `enum[]` to attributes that reference separate
   model-local enum type nodes.
8. Read enum `items` from a named `enums` entry or an inline `enum` or
   `enum[]` attribute declaration.
9. Normalize the attribute's `index` shorthand to explicit index metadata.
10. Resolve source inheritance and each attribute's effective domains.
11. Validate an attribute format's syntax and compatibility with its value type.
12. Validate predefined values against model attributes and enum types.
13. Preserve presentation expressions as strings without interpreting or
    validating the template language in metadata.
14. Permit `unit` only on attributes.
15. Validate query result types when executing the query.
16. Require an explicit or unambiguously inferable query root at execution.
17. Let the selected provider validate its ability to implement or materialize
    requested indexes and cross-source behavior.

## Canonical syntax decisions

The following metadata syntax choices are intentionally canonical. The
template expressions shown in `to_title` and `unit` are illustrative.

```yaml
type: user[]          # not []user or user.collection
type: set<user>       # queryable set, not a stored collection
type: enum[]          # inline enum collection

type: datetime        # one temporal value type
format: 'dd.MM.yyyy'  # concrete input and display pattern

title: Player account # literal model title
to_title: '{login}'  # literal or computed instance title
set: users           # ordinary model property; creates no metadata node
set_title: Users

unit: '{currency.title}' # attribute presentation

items:
  - name: super_admin
    id: 1             # optional, keeps the ID stable when items are reordered
```

The format uses a scalar `set` name and expresses cardinality through `type`.

## Implementation scope

Implementations may support different provider capabilities. The query
grammar in this draft is preliminary. Query execution and provider capability
checks depend on the runtime and its storage adapters.
