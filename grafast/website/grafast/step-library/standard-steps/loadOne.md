---
toc_max_heading_level: 4
---

import Mermaid from "@theme/Mermaid";

# loadOne

Similar to [DataLoader][]'s load method, uses the given callback function to
read a single result from your business logic layer. To load a list, see
[`loadMany`](./loadMany.md).

## Enhancements over DataLoader

Thanks to the planning system in Gra*fast*, `loadOne` can expose features that
are not possible in DataLoader.

### Attribute and parameter tracking

A `loadOne` step (technically a `LoadedRecordStep`) keeps track of the
attribute names accessed via `.get(attrName)` and any parameters set via
`.setParam(key, value)`. This information will be passed through to your
callback function such that you may make more optimal calls to your backend
business logic, only retrieving the data you need.

### Input/output equivalence

If you (optionally) pass an `ioEquivalence` parameter to `loadOne` (the second
parameter) then you can use it to indicate which field(s) on the output is
equivalent to the input(s). This enables an optimization where a chained fetch
can instead be performed in parallel if the child only depends on an output
which is equivalent to an input. Hopefully an example will make this clearer...

Imagine you're loading a user and their organization:

```graphql
{
  currentUser {
    id
    name
    friends {
      id
      name
    }
  }
}
```

You might have plan resolvers such as:

```ts
const objects = {
  Query: {
    plans: {
      currentUser() {
        const $currentUserId = context().get("userId");
        return loadOne($currentUserId, batchGetUserById);
      },
    },
  },
  User: {
    plans: {
      friends($user) {
        const $userId = $user.get("id");
        return loadMany($userId, batchGetFriendsByUserId);
      },
    },
  },
};
```

In its current state the system doesn't know that the `$user.get("id")` is
equivalent to the `context().get("userId")`, so this would result in a chained
fetch:

<Mermaid chart={`\
stateDiagram
  direction LR
  state "batchGetUserById" as currentUser
  state "batchGetFriendsByUserId" as friends
  [*] --> currentUser
  currentUser --> friends
`} />

However, we can indicate that the output of the `loadOne` step's `id` property
(`$user.get("id")`) is equivalent to its input (`context().get("userId")`):

```diff {6-7}
 const objects = {
   Query: {
     plans: {
       currentUser() {
         const $currentUserId = context().get("userId");
-        return loadOne($currentUserId, batchGetUserById);
+        return loadOne($currentUserId, 'id', batchGetUserById);
       },
     },
   },
   User: {
     plans: {
       friends($user) {
         const $userId = $user.get("id");
         return loadMany($userId, batchGetFriendsByUserId);
       },
     },
   },
 };
```

Now the access to `$user.get("id")` will be equivalent to
`context().get("userId")` - we no longer need to wait for the `$user` to load
in order to fetch the friends:

<Mermaid chart={`\
stateDiagram
  direction LR
  state "batchGetUserById" as currentUser
  state "batchGetFriendsByUserId" as friends
  [*] --> currentUser
  [*] --> friends
`} />

## Usage

```
loadOne($spec, [$unaryStep,] [ioEquivalence,] callback)
```

`loadOne` accepts two to four arguments. The first is the step that specifies
which records to load (the _specifier step_), the last is the callback function called with these
specs responsible for loading them.

```ts
// Basic usage:
const $record = loadOne($spec, callback);

// Advanced usage:
const $record = loadOne($spec, $unaryStep, ioEquivalence, callback);
const $record = loadOne($spec, $unaryStep, callback);
const $record = loadOne($spec, ioEquivalence, callback);
```

Where:

- `$spec` is any step
- `$unaryStep` is any _unary_ step - see [Unary step usage](#unary-step-usage) below
- `ioEquivalence` is either `null`, a string, an array of strings, or a string-string object map - see [ioEquivalence usage](#ioequivalence-usage) below
- and `callback` is a callback function responsible for fetching the data.

### Callback

The `callback` function is called with two arguments, the first is
a list of the values from the _specifier step_ `$spec` and the second is options that
may affect the fetching of the records.

```ts
function callback(
  specs: ReadonlyArray<unknown>,
  options: {
    unary: unknown;
    attributes: ReadonlyArray<string>;
    params: Record<string, unknown>;
  },
): PromiseOrDirect<ReadonlyArray<unknown>>;
```

:::tip

For optimal results, we strongly recommend that the callback function is defined
in a common location so that it can be reused over and over again, rather than
defined inline. This will allow the underlying steps to optimize calls to this function.

:::

Within this definition of `callback`:

- `specs` is the runtime values of each value that `$spec` represented
- `options` is an object containing:
  - `unary`: the runtime value that `$unaryStep` (if any) represented
  - `attributes`: the list of keys that have been accessed via
    `$record.get('<key>')`
  - `params`: the params set via `$record.setParam('<key>', <value>)`

`specs` is deduplicated using strict equality; so it is best to keep `$spec`
simple - typically it should only represent a single scalar value - which is
why `$unaryStep` exists.

`options.unary` is very useful to keep specs simple (so that fetch
deduplication can work optimally) whilst passing in global values that you may
need such as a database or API client.

`options.attributes` is useful for optimizing your fetch - e.g. if the user
only ever requested `$record.get('id')` and `$record.get('avatarUrl')` then
there's no need to fetch all the other attributes from your datasource.

`options.params` can be used to pass additional context to your callback
function, perhaps options like "should we include archived records" or "should
we expand 'customer' into a full object rather than just returning the
identifier".

### Basic usage

```ts
const $userId = $post.get("author_id");
const $user = loadOne($userId, batchGetUserById);
// OR: const $user = loadOne($userId, 'id', batchGetUserById);
```

An example of the callback function might be:

```ts
async function batchGetUserById(ids, { attributes }) {
  // Your business logic would be called here; e.g. this might be the same
  // function that your DataLoaders would call, except we can pass additional
  // information to it.

  // For example, load from the database
  const rows = await db.query(
    sql`SELECT id, ${columnsToSql(attributes)} FROM users WHERE id = ANY($1);`,
    [ids],
  );

  // Ensure you return the same number of results, and in the same order!
  return ids.map((id) => rows.find((row) => row.id === id));
}
```

### Unary step usage

:::info

A unary step is a step that only ever represents one value, e.g. simple derivatives of `context()`, `fieldArgs`, or `constant()`.

:::

In addition to the forms seen in "Basic usage" above, you can pass a second
step to `loadOne`. This second step must be a [**unary
step**](../../step-classes.md#addunarydependency), meaning that it must represent
exactly one value across the entire request (not a batch of values like most
steps).

```ts
const $userId = $post.get("author_id");
const $dbClient = context().get("dbClient");
const $user = loadOne($userId, $dbClient, "id", batchGetUserFromDbById);
// OR: const $user = loadOne($userId, $dbClient, batchGetUserFromDbById);
```

Since we know it will have exactly one value, we can pass it into the
callback as a single value and our callback will be able to use it directly
without having to perform any manual grouping.

This unary dependency is useful for fixed values (for example, those from
GraphQL field arguments) and values on the GraphQL context such as clients to
various APIs and other data sources.

An example of the callback function might be:

```ts
async function batchGetUserFromDbById(ids, { attributes, unary }) {
  const dbClient = unary;

  const rows = await dbClient.query(
    sql`SELECT id, ${columnsToSql(attributes)} FROM users WHERE id = ANY($1);`,
    [ids],
  );

  return ids.map((id) => rows.find((row) => row.id === id));
}
```

### ioEquivalence usage

The `ioEquivalence` optional parameter can accept the following values:

- `null` to indicate no input/output equivalence
- a string to indicate that the same named property on the output is equivalent
  to the entire input plan
- if the step is a `list()` (or similar) plan, an array containing a list of
  keys (or null for no relation) on the output that are equivalent to the same
  entry in the input
- if the step is a `object()` (or similar) plan, an object that maps between
  the attributes of the object and the key(s) in the output that are equivalent
  to the given entry on the input

```ts title="Example for a list step"
const $member = loadOne(
  list([$organizationId, $userId]),
  ["organization_id", "user_id"],
  batchGetMemberByOrganizationIdAndUserId,
);

// - batchGetMemberByOrganizationIdAndUserId will be called with a list of
//   2-tuples, the first value in each tuple being the organizationId and the
//   second the userId.
// - Due to the io equivalence (2nd argument):
//   - `$member.get("organization_id")` will return `$organizationId` directly
//   - `$member.get("user_id")` will return `$userId` directly
```

```ts title="Example for an object step"
const $member = loadOne(
  object({ oid: $organizationId, uid: $userId }),
  { oid: "organization_id", uid: "user_id" },
  batchGetMemberByOrganizationIdAndUserId,
);

// - batchGetMemberByOrganizationIdAndUserId will be called with a list of
//   objects; each object will have the key `oid` set to an organization id,
//   and the key `uid` set to the user ID.
// - Due to the io equivalence (2nd argument):
//   - `$member.get("organization_id")` will return the step used for `oid`
//     (i.e. `$organizationId`) directly
//   - Similarly `$member.get("user_id")` will return `$userId` directly
```

### Passing multiple steps

The [`list()`](./list) or [`object()`](./object) step can be used if you need
to pass the value of more than one step into your callback:

```ts
const $isAdmin = $user.get("admin");
const $stripeId = $customer.get("stripe_id");
const $last4 = loadOne(list([$isAdmin, $stripeId]), getLast4FromStripeIfAdmin);
```

The first argument to the `getLast4FromStripeIfAdmin` callback will then be an
array of all the tuples of values from these plans: `ReadonlyArray<readonly [isAdmin:
boolean, stripeId: string]>`. The callback might look something like:

```ts
async function getLast4FromStripeIfAdmin(tuples) {
  const stripeIds = uniq(
    tuples
      .filter(([isAdmin, stripeId]) => isAdmin)
      .map(([isAdmin, stripeId]) => stripeId),
  );
  const last4s = await getLast4FromStripeIds(stripeIds);

  return tuples.map(([isAdmin, stripeId]) => {
    if (!isAdmin) return null;
    const index = stripeIds.indexOf(stripeId);
    return last4s[index];
  });
}
```

This technique can also be used with the unary step in advanced usage.

:::tip Performance impact from using list/object

Using `list()` / `object()` like this will likely reduce the effectiveness of
`loadOne`'s built in deduplication; to address this a stable object/list is
required - please track this issue:
https://github.com/graphile/crystal/issues/2170

:::

[dataloader]: https://github.com/graphql/dataloader
