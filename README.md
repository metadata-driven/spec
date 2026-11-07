# MDD Metadata Specification

Metadata-Driven Development (MDD) describes a project's structure and its
supporting infrastructure in one YAML document. You can see which models the
project has, how they relate, which databases and other sources they use, and
how their data appears in the interface. Models and attributes have stable
names and can carry human-readable titles, descriptions, and icons.

Tools can use these definitions to generate application models and code,
database schemas, and user interfaces such as forms. Business and product
teams, frontend and backend developers, database administrators, DevOps teams,
and AI agents can all work from the same description and refer to the same
model and field names, even when the implementation spans several services.

Created by [Denis Glazkov (@denisskin)](https://github.com/denisskin).

[Full example (YAML)](examples/canonical.yaml)

## Start with a player

Mega Game is a fictional multiplayer gaming platform with an online store. A
player account has a login and a display name. The `invited_by` field references
the player who sent the invitation, and `friends` stores a list of players.

```yaml
project:
  name: mega_game
  title: Mega Game
  description: Mega Game sample application with multiplayer game servers and an online store
  icon: 🕹️

models:
  - name: user
    title: Player account
    description: A registered Mega Game player
    icon: 👤

    set: users
    set_title: Users
    set_description: All application users
    set_icon: 👥

    attrs:
      - name: login
        title: Login
        type: string(25)
        index: unique

      - name: name
        title: Display name
        type: string

      - name: bio
        title: Bio
        description: A short introduction written by the player
        type: string
        index: fulltext

      - name: birthday
        title: Birthday
        type: datetime
        format: 'dd.MM.yyyy'

      - name: invited_by
        title: Invited by
        description: The player who sent the invitation
        type: user

      - name: friends
        title: Friends
        description: List of friends
        type: user[]

      - name: score
        type: int

      - name: preferences
        type: json

      - name: avatar
        title: Avatar
        description: The player's profile image
        icon: 🖼️
        type: file
        source: files

      - name: latitude
        title: Latitude
        type: number(9,6)
        source: realtime_db

      - name: longitude
        title: Longitude
        type: number(9,6)
        source: realtime_db

```

`user` names the model, `users` names the set you can query, and `user.login`
identifies the login field in the metadata. Declaring `set: users` gives the
model an `id` field by default. The full-text index on `bio` supports profile
search by the text of a player's introduction. The `avatar` field exposes the
profile image as a stream.


## Present each record

At model level, `title`, `description`, and `icon` describe an individual
record. Mega Game can show `Alex (@alex)` as the title and use the icon from
Alex's role. Values in braces are resolved against the current player.

```yaml
- name: user
  title: '{name} (@{login})'
  description: '{role.title} · Account #{id}'
  icon: '{role.icon}'
```

## Declare enumerations

An inline `enum` gives a field a defined set of choices. Each role has a name
used in assignments and can also have a display title and an icon.

```yaml
attrs:
  - name: role
    title: Player role
    type: enum
    items:
      - name: player
        title: Player
        icon: 🎮

      - name: moderator
        id: 2
        title: Moderator
        icon: 🛡️
```

## Query related records as sets

Mega Game stores sessions in Redis. Each session references its player and the
game host, and a compound index on `user` and `ts` supports looking up sessions
by player and time.

```yaml
models:
  - name: session
    set: sessions
    source: realtime
    attrs:
      - name: user
        type: user

      - name: host
        type: host

      - name: ts
        type: datetime

    indexes:
      - attrs: [user, ts]
```

Add `sessions` under `user.attrs` to query a player's sessions in creation
order. The query resolves `{id}` against the current player.

```yaml
attrs:
  - name: sessions
    type: set<session>
    query: 'from(sessions) where(user={id}) orderby(ts)'
```

## Declare predefined records

A model's `items` can declare records that come with the application. The
`game_guide` account has a fixed ID and is read-only in ordinary application
operations. Its `data` supplies the values of the `user` fields.

```yaml
items:
  - name: game_guide
    id: 1
    title: Mega Game Guide
    description: Built-in community guide account
    icon: 🧭
    data:
      login: game_guide
      name: Mega Game Guide
      role: moderator
```

## Group fields for forms

Moving `name` and `invited_by` into a `profile` group under `user.attrs` lets
an editor render them together as a section of the player form. Their paths
are still `user.name` and `user.invited_by`.

```yaml
attrs:
  - group: profile
    title: Player profile
    description: Display name and invitation details
    icon: 👤
    attrs:
      - name: name
        title: Display name
        type: string

      - name: invited_by
        title: Invited by
        type: user
```

## Format values in forms

In the `order` model, `format` controls how the amount is displayed and how the
form parses input. `unit` shows the selected currency next to the amount. For
example, an order for `1250` with `currency: eur` can display as `1,250.00 EUR`
in an English locale.

```yaml
attrs:
  - name: currency
    title: Currency
    type: enum
    items:
      - name: usd
        title: USD
      - name: eur
        title: EUR

  - name: amount
    title: Amount
    type: number(12,2)
    format: '#,##0.00'
    unit: '{currency.title}'
```

## Connect databases and add indexes

Mega Game stores player accounts in MySQL and its game server registry in
PostgreSQL. It also uses Redis for frequently updated data. Each model selects
its default provider through `source`. The unique index on `login` enforces
distinct player logins.

```yaml
sources:
  - name: maindb
    conn: 'mysql://mega_game_app@/mega_game'

  - name: gamedb
    conn: 'postgres://mega_game_ops@localhost:5432/mega_game_hosts'

  - name: realtime
    conn: 'redis:localhost:6379/0'

  - name: files
    conn: 'file:./var/mega-game/uploads'

models:
  - name: user
    set: users
    source: maindb
    attrs:
      - name: login
        type: string(25)
        index: unique

  - name: host
    set: hosts
    source: gamedb
    attrs:
      - name: id
        type: string(64)

      - name: ip
        type: string(45)
        index: regular

```

## Store one model across sources

The model's `source` is the default for its fields. In `user.attrs`, the
`online` and `last_seen` fields can use `realtime` for frequently updated
presence data while the other stored account fields use `maindb`.

```yaml
attrs:
  - name: online
    type: bool
    source: realtime

  - name: last_seen
    type: datetime
    source: realtime
```

The application still works with one `user`. Both sources use the same record
ID, and the runtime coordinates reads and writes. Consistency and transaction
guarantees depend on the runtime and storage adapters.

## Store collections of values

An attribute with a `T[]` type stores a list. A player can have a list of tags
and a list of references to other players.

```yaml
attrs:
  - name: tags
    type: string[]

  - name: best_friends
    type: user[]
```


## Reference

- [Metadata format specification](specification.md)

## Status

The specification is a draft. Changes to format rules should update the
specification and any affected examples in the same pull request.

## License

The specification and its examples are licensed under [CC BY 4.0](LICENSE).
