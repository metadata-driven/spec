# Proposal for named and inline enumerations

**Status — Accepted for enum declarations.** See the [specification](../specification.md#enumerations).

An enumeration defines an ordered set of values that any compatible attribute
can use. A document can declare an enumeration at the root, within a model,
or inline on an attribute. All three forms produce the same kind of type
definition in the normalized metadata.

## Named declarations

The root `enums` sequence declares types with names such as `status`. A model's
`enums` sequence declares types with qualified names such as `user.type`. A
type's path uses the singular model name and the declared enum name. An element
adds its own name to that path, for example `status.done` or `user.type.admin`.

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
        items:
          - name: admin
            id: 1
            title: Administrator

    attrs:
      - name: type
        type: user.type
        title: User type

      - name: status
        type: status
```

An explicit enum can have its own `title`, `description`, and `icon`. An
attribute that uses it has its own presentation fields. Singular names are a
convention for explicit declarations. Name validation follows the usual rules
for metadata identifiers.

## Inline declarations

The following model produces the same `user.type` attribute and enum as the
named declaration above:

```yaml
models:
  - name: user
    attrs:
      - name: type
        type: enum
        title: User type
        items:
          - name: admin
            id: 1
            title: Administrator
```

The parser creates an enum in the owning model using the attribute's name
verbatim, moves `items` into that enum, and replaces the attribute's `type`
with the qualified type name. The field keeps its own `title`, `description`,
`icon`, and other attribute properties. The enum uses the common presentation
defaults. Converting this syntax to an explicit declaration preserves the enum
path, element paths, and effective IDs when the `items` sequence is carried
over as written.

`enum[]` follows the same rule and preserves the collection suffix on the
attribute. For example, an inline `permissions` field declared as `enum[]`
creates the enum `user.permissions` and the field type `user.permissions[]`.
An explicit declaration allows the same field to use a type named
`user.permission` instead.

## Type and attribute lookup

Type references resolve in a namespace shared by models and enumerations.
Attributes have their own namespace. A model and a root enum named `status`
conflict in type lookup. An attribute and an enum named `user.type` can coexist.
An enum named `type` in the `user` model's `enums` sequence and an inline
enum on the `user.type` attribute are duplicate declarations and cause a
validation error. The parser collects type declarations before resolving
references. An attribute reuses an existing enumeration by referencing its
name in `type`.

The Go metadata API exposes the distinction with typed lookups:

| Lookup | Result |
| --- | --- |
| `Meta.LookupType("user.type")` | The enum definition with its own presentation and `items`. |
| `Meta.LookupAttr("user.type")` | The attribute definition with `user.type` as its resolved value type. |

An editor or other caller addressing either kind of node can use the namespace
along with the path. The Go model generator works from the normalized type and
attribute definitions regardless of the syntax used to declare them.

## Values and queries

Enum items use the existing [item identity rules](../specification.md#item-identity).
Effective IDs are scoped to the enumeration. A short assignment such as
`type: admin` resolves through the attribute's type. The full path
`user.type.admin` identifies the item when that context is unavailable.
`set<user.type>` and `from(user.type)` use the same qualified type name for
queries over the enumeration.
