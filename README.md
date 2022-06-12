# weight-balanced-tree

A [persistent](https://en.wikipedia.org/wiki/Persistent_data_structure)
weight-balanced ([bounded balance](https://en.wikipedia.org/wiki/Weight-balanced_tree))
tree for JavaScript.

 * Zero dependencies
 * Usable as a persistent map or set
 * Works in Node.js and the browser
 * Flow and TypeScript definitions included

## Installation

This software is released under the [MIT license](LICENSE).

It's published on npm as [weight-balanced-tree](https://www.npmjs.com/package/weight-balanced-tree),
so you can install it using `yarn` or `npm`.

## API

To create a tree, see `create`, `insert`, or `fromDistinctAscArray` below.

A tree consists of at least one value.  There's no tree of size 0; an empty
tree is represented by `null`.

Although there's only one datum stored per node, for maps you can store a
`[key, value]` tuple, or store `key` directly on `value` if it's an object.

```
type ImmutableTree<+T> = {
  +left: ImmutableTree<T> | null,
  +right: ImmutableTree<T> | null,
  +size: number,
  +value: T,
};

type InsertConflictHandler<T, K> =
  (existingTreeValue: T, key: K) => T;

type InsertNotFoundHandler<T, K> =
  (key: T) => T;
```

### size

How many values are contained in the tree.

### insert()

```
insert<T>(
    tree: ImmutableTree<T> | null,
    value: T,
    cmp: (T, T) => number,
    onConflict?: InsertConflictHandler<T, T>,
): ImmutableTree<T>;
```

Returns a new version of `tree` with `value` inserted.

Pass `null` for `tree` to create a new tree.  (This module has no such thing
as a tree of size 0.)

The `cmp` (comparator) function is used to order the values.  It works the
same as the comparator you'd pass to `Array.prototype.sort`, and should
never change for a particular tree.

`onConflict` allows you to configure what happens when `value` already exists
in the tree according to `cmp`.  It receives the existing tree value as its
first argument, and the value you passed to `insert` as its second argument.

`onConflict` is expected to return a final value to be inserted, or throw an
error if the value shouldn't exist.  This allows you to merge both values in
some way if needed.  However, the returned value must have the same sort
order as before.

If you return `existingTreeValue` from `onConflict`, `insert` will return the
same `tree` reference back.  `Object.is` is used to determine if the value
you return is the same as `existingTreeValue`.

If `onConflict` is not specified, the default action is to throw an error.
There are several exports in [insert.mjs](insert.mjs) that can be used here:

 * `onConflictThrowError` (the default), which throws an exception.
 * `onConflictKeepTreeValue`, which just returns the existing tree value back
   unmodified.  In this case, `insert` will also return the same tree
   reference back.
 * `onConflictUseGivenValue`, which replaces the existing tree value with the
   value given to `insert`.

There are also some convenience functions available that call `insert` with
these options for you:

 * `insertIfNotExists` (passes `onConflictKeepTreeValue` for you)
 * `insertOrReplaceIfExists` (passes `onConflictUseGivenValue` for you)

`insertOrThrowIfExists` is an alias of `insert`.

### insertByKey()

```
insertByKey<T, K>(
    tree: ImmutableTree<T> | null,
    key: K,
    cmp: (key: K, treeValue: T) => number,
    onConflict: InsertConflictHandler<T, K>,
    onNotFound: InsertNotFoundHandler<T, K>,
): ImmutableTree<T>;
```

This is a generalized version of `insert` (which calls `insertByKey` under
the hood).  The main difference is that it takes a `key` of type `K` instead
of a `value` of type `T`, and allows you to construct a value when `key` is
not found.

This is particularly convenient where you're using the tree as a map. and the
keys are properties of the items being mapped.

`onNotFound` handles the key-not-found case.  It only receives one argument,
the `key` you passed to `insertByKey`.  Like `onConflict`, you are expected to
return a final value of type `T` to be inserted.

`onNotFound` is useful in at least a couple scenarios:

  * You want to create the value to insert lazily, only if it doesn't exist.

  * You want to throw an error if the value doesn't exist (because you expect
    to replace it).

The following exports in [insert.mjs](insert.mjs) can be used for
`onNotFound` instead of defining your own:

 * `onNotFoundUseGivenValue`, which is what `insert` and all associated
   helpers default to.  Note that the given value in this case is the `key`,
   so this only works in cases where `K` is a subtype of `T`.
 * `onNotFoundDoNothing`, which causes `insertByKey` to do nothing and return
   the same `tree` reference back if the key doesn't exist.
 * `onNotFoundThrowError`, which throws a `ValueNotFoundError` if the key
   doesn't exist.

Here's an example of sorting/mapping objects by a simple key property and
lazily creating them:

```TypeScript
interface Item {
  readonly key: number;
}

function compareKeyWithItemKey(key: number, item: Item): number {
  return key - item.key;
}

function onNotFoundCreateItemFromKey(key: number): Item {
  return {key};
}

insertByKey<Item, number>(
  tree,
  /* key = */ 1,
  compareKeyWithItemKey,
  onConflictThrowError,
  onNotFoundCreateItemFromKey,
);

// a "find or insert" implementation:

let item2;
insertByKey<Item, number>(
  tree,
  /* key = */ 1,
  compareKeyWithItemKey,
  (existingItem: Item) => {
    item2 = existingItem;
  },
  (key: number) => {
    item2 = onNotFoundCreateItemFromKey(key);
    return item2;
  },
);
```

### remove()

```
remove<T>(
    tree: ImmutableTree<T> | null,
    value: T,
    cmp: (T, T) => number,
): ImmutableTree<T> | null;
```

Returns a new version of `tree` with `value` removed.

If `value` is not found in the tree, the same tree reference is returned
back.

If this was the last value in `tree`, `null` is returned.

The `cmp` (comparator) function is the same as used for `insert`.

`removeIfExists` is an alias of `remove`.

### removeOrThrowIfNotExists

Like `remove`, but throws an error if `value` does not exist in the tree.

This simply checks if the tree returned from `remove` is the same reference.

### equals()

```
equals<T>(
  a: ImmutableTree<T> | null,
  b: ImmutableTree<T> | null,
  cmp: (a: T, b: T) => number,
): boolean;
```

Returns `true` if two trees contain the same values, or `false` otherwise.

### find()

```
find<T>(
    tree: ImmutableTree<T> | null,
    value: T,
    cmp: (T, T) => number,
): ImmutableTree<T> | null;
```

Finds the branch of `tree` containing `value` and returns it, or `null` if not
found.

The `cmp` (comparator) function is the same as used for `insert`.

### findNext()

```
findNext<T, V = T>(
  tree: ImmutableTree<T> | null,
  value: V,
  cmp: (V, T) => number,
): ImmutableTree<T> | null;
```

Finds the branch of `tree` that follows `value` and returns it, or `null` if
there is no such node.  `value` does not have to exist in the tree: if a set
has 1 & 3, the next value from 2 is 3.

### findPrev()

```
findNext<T, V = T>(
  tree: ImmutableTree<T> | null,
  value: V,
  cmp: (V, T) => number,
): ImmutableTree<T> | null;
```

Finds the branch of `tree` that precedes `value` and returns it, or `null` if
there is no such node.  `value` does not have to exist in the tree: if a set
has 1 & 3, the previous value from 2 is 1.

### fromDistinctAscArray()

```
fromDistinctAscArray<T>(
  array: $ReadOnlyArray<T>,
): ImmutableTree<T> | null;
```

If `array` is sorted and contains only unique values, then this returns a new,
valid, and balanced tree with the values from `array`.  (This is faster than
building the tree value-by-value with `insert`.)

If `array` is not sorted or contains duplicate values, then this returns an
invalid tree.  (Do not do this.)

### iterate()

```
iterate<T>(tree: ImmutableTree<T> | null): Generator<T, void, void>;
```

Returns a JS iterator that traverses the values of the tree in order.

### reverseIterate()

```
reverseIterate<T>(tree: ImmutableTree<T> | null): Generator<T, void, void>;
```

Returns a JS iterator that traverses the values of the tree in reverse order.

### minNode()

```
minNode<T>(tree: ImmutableTree<T>): ImmutableTree<T>;
```

Returns the "smallest" (left-most) node in `tree`.

### minValue()

```
minValue<T>(tree: ImmutableTree<T>): T;
```

Returns the "smallest" (left-most) value in `tree`.

This is equivalent to `minNode(tree).value`.

### maxNode()

```
maxNode<T>(tree: ImmutableTree<T>): ImmutableTree<T>;
```

Returns the "largest" (right-most) node in `tree`.

### maxValue()

```
maxValue<T>(tree: ImmutableTree<T>): T;
```

Returns the "largest" (right-most) value in `tree`.

This is equivalent to `maxNode(tree).value`.

### toArray()

```
toArray<T>(
  tree: ImmutableTree<T> | null,
): $ReadOnlyArray<T>;
```

Flattens `tree` into an array of values.

### union()

```
union<T>(
  t1: ImmutableTree<T> | null,
  t2: ImmutableTree<T> | null,
  cmp: (a: T, b: T) => number,
  onConflict?: (v1: T, v2: T) => T,
): ImmutableTree<T> | null;
```

Merges two trees together using the comparator `cmp`.  `onConflict` handles
the case where an equivalent value appears in both trees, and is expected to
return the final value to use in the union (though it must have the same
relative sort order as `v1` and `v2`).  If not specified, by default `union`
will prefer values in `t2` when resolving conflicts.

### zip()

```
zip<T, U>(
  t1: ImmutableTree<T> | null,
  t2: ImmutableTree<U> | null,
): Generator<[T | void, U | void], void, void>;
```

Zips two trees together, returning an iterable of tuples: the first tuple
contains the first values of both trees, the second tuple contains the second
values of both trees, and so on.  If the trees are of different sizes,
`undefined` is used within a tuple where a corresponding value is missing.

## Performance

Performance will largely depend on the size of your data and the cost of your
comparator function.  [benchmark.mjs](benchmark.mjs) tests an ASCII table with
uniform-length string keys and a simple string comparator function.

Comparisons against [Immutable.List](https://immutable-js.com/) and plain
arrays are included for insertions and removals.

You can run `node benchmark.mjs` yourself.

## Tests

Run `./node_modules/.bin/c8 node test.mjs`.

To test the .d.ts files, run `./node_modules/.bin/tsd`.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## References

 1. Adams, Stephen.
    "Implementing Sets Efficiently in a Functional Language."
    University of Southampton, n.d.
    Accessed at http://groups.csail.mit.edu/mac/users/adams/BB/92-10.ps

 2. [GHC's Data.Map.Internal](https://gitlab.haskell.org/ghc/packages/containers/-/blob/f00aa02/containers/src/Data/Map/Internal.hs).
