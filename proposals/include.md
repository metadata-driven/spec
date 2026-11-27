# Proposal for composing metadata with `include`

**Status — Draft for future consideration**

A project can split its metadata across files. For example, billing can
declare invoices, register its own storage source, and add fields to the
shared `user` model. The loader combines all declarations into one project
with one metadata graph. A model keeps the same logical path when its
declaration moves to a different file.

## File layout

The entry document declares application metadata at its root and can declare an
ordered `include` list of local YAML file paths. Paths are relative to the entry
document's directory.
The entry document can also declare `domains`, `sources`, and `models`. Each
included file can contain any of these three sections. A project can, for
example, keep shared connections in `sources.yaml` and business declarations
in files such as `models/shop.yaml` and `models/game.yaml`. Domain membership
comes from `domains` on the model or attribute.

The first version would read the entry document's declarations first, then
each included document in the order listed. A missing file, an invalid YAML
document, or a second reference to the same resolved file is an error.
Duplicate names within one file are still declaration errors.

## Example

`project.yaml`

```yaml
name: mega_game
title: Mega Game

include:
  - billing.yaml

domains:
  - name: website
    title: Website

sources:
  - name: maindb
    conn: 'mysql://mega_game_app@/mega_game'

models:
  - name: user
    set: users
    source: maindb
    domains: website
    attrs:
      - group: profile
        title: Profile
        attrs:
          - name: login
            type: string(25)
            index: unique
```

`billing.yaml`

```yaml
domains:
  - name: billing
    title: Billing

sources:
  - name: billing_db
    conn: 'postgres://mega_game_billing@localhost:5432/mega_game_billing'

models:
  - name: user
    domains: billing
    attrs:
      - group: profile
        attrs:
          - name: billing_email
            type: string

      - name: tax_id
        type: string
        source: billing_db

      - name: invoices
        type: set<invoice>
        query: 'from(invoices) where(user={id})'

    indexes:
      - attrs: [login, billing_email]

  - name: invoice
    set: invoices
    source: billing_db
    domains: billing
    attrs:
      - name: user
        type: user
```

The result contains one `user` model in `website` and `billing`. Its
`profile` group contains `login` and `billing_email`. Both fields use
`maindb`, so the compound index can use that source. `user.tax_id` uses
`billing_db`. Invoices remain separate records linked to the user through
`invoice.user` and accessible through the queryable `user.invoices` set.

## Merge rules

Models, domains, and sources merge by `name` within their respective top-level
sections. The first declaration determines the object's position. A later
fragment can supply a property that earlier fragments omitted. If more than
one fragment supplies the same property, the values must agree unless a rule
below explicitly combines them. An explicitly empty list counts as a supplied
value. Compare `options` as a single value, with mapping key order ignored.

| Declaration | Rule |
| --- | --- |
| Domain | Combine declarations of the same domain. For example, a later file may supply its `icon`. Conflicting presentation fields are errors. |
| Source | Combine declarations of the same source. For example, one file may specify `conn` and another its `title`. Different values for `conn` are a conflict. |
| Model properties | Apply the same agreement rule to `set`, `source`, `title`, `description`, `icon`, `set_*`, and other properties without a specific merge rule. |
| Model `domains` | Normalize a single name to a one-element list and validate each list. Combine lists in declaration order and keep the first occurrence of each domain across fragments. |
| Model `attrs` | Append new attributes in declaration order. Each attribute is declared in full in one fragment. Its name must be unique across the model, including grouped and ungrouped attributes. An inline enum and its `items` belong to its attribute's declaration. |
| Attribute groups | Merge groups with the same `group` in a model. Keep the group's first position and append its new members in declaration order. The agreement rule applies to group properties such as `title`, `description`, and `icon`. Flatten the completed `attrs` list in order, expanding each group at its position. Attribute paths stay model-relative, for example `user.billing_email`. |
| Model `indexes` | Append declarations in order. Indexes can reference fields from different files. Repeated explicit index names are errors. For unnamed indexes, a duplicate can be identified by index type, ordered attribute names, and effective source after resolution. Apply the existing rules for attribute index shorthand to the combined model. |
| Model `items` | Append items in order. Validate names and effective IDs in the final sequence. Implicit IDs are one-based positions in that sequence. Use explicit IDs for persisted items whose positions can change when files are reordered. |

### Domain and source inheritance

Set each attribute's effective domain names **before merging model
fragments**. An attribute with its own `domains` uses that value. An
attribute with `domains: []` has an empty domain list. When the key is omitted,
use the enclosing model fragment's domains, or an empty list if the fragment
omits them too. This also applies to attributes inside groups, even when a
group receives members from several files. In the example, `user.login`
belongs to `website` and `user.billing_email` belongs to `billing`.
The merged model's domain list comes from its model fragments.

Resolve **source inheritance after merging**. An attribute uses its explicit
`source` or the assembled model's default source. A source used by an
attribute, a model, or an index can be declared in any of the files.
An index without `source` uses the common effective source of its attributes
under the existing index rules.

### Resolution and errors

Resolve metadata references against the whole project after collecting
declarations. This allows forward references across files. Query roots also
refer to the combined project and follow the existing query resolution rules.
Apply existing normalization and validation to the merged result, including
implicit identifiers, index shorthand, and duplicate item ID checks.

Keep track of the file and location of each declaration. Report conflicts
using the logical metadata path, such as `user.source` or
`user.groups.profile.title`, and both declaration locations. Report an
unresolved reference at the location that supplied it.

## Open questions

- Whether to support nested includes, globs, or remote sources in a later
  version. Nested includes would need their own relative-path and cycle rules.
- Whether use cases emerge for deliberately overriding properties,
  merging individual keys in `options`, or extending an already declared
  attribute or inline enum. The proposed agreement rule handles additive
  declarations.
- Which domains to give an implicit `id` attribute when the model spans
  several fragments. One option is to use the merged model's domain list,
  since the identifier is shared by all fragments.
