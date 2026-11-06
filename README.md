# MDD Metadata Specification

Metadata-Driven Development (MDD) describes application models, attributes,
relationships, presentation, and data sources in a YAML document. Implementations
can use that document to validate configurations, map models to storage,
generate code, and build editors.

Created by [Denis Glazkov (@denisskin)](https://github.com/denisskin).

## Start here

- [Metadata format specification](specification.md) defines the document structure,
  type expressions, paths, normalization, and validation rules.
- [Canonical configuration](examples/canonical.yaml) shows the format in a single
  application with models, references, enums, indexes, and several data sources.

## Example

```yaml
models:
  - name: user
    set: users
    source: maindb
    attrs:
      - name: login
        type: string(25)
        index: regular

sources:
  - name: maindb
    conn: 'mysql://game_base_app@/game_base'
```

Here `user` names a model, `users` is its query root, and `user.login` is the
canonical metadata path of the attribute. The model's source is also the default
source for its attributes.

## Status

The specification is a draft. Changes to format rules should update the
specification and any affected examples in the same pull request.

## License

The specification and its examples are licensed under [CC BY 4.0](LICENSE).
