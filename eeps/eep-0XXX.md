    Author: Adam Lindberg <hello(at)alind(dot)io>
    Status: Draft
    Type: Standards Track
    Created: 14-Feb-2025
    Erlang-Version: OTP-28.0
    Post-History:
****
EEP XXX: Deep Map Access
----

Abstract
========

Maps in Erlang are frequently used to represent nested structures, such as JSON
data or configuration settings. This EEP proposes new library functions in the
`maps` module for accessing nested maps, which will make it easier to access and
modify such data.

> [!NOTE]
> This document uses the terms deep and nested interchangeably to denote a map
> that can potentially contain other maps as values. The suffix 'deep' is used in
> function names for brevity. This should preferably be resolved before this EEP
> is finalized.

Motivation
==========

There is currently no good way to access or modify nested map structures
dynamically (or otherwise) in Erlang, and the ways that do exist are limited and
can be cumbersome to implement.

Assume the following JSON document:

```erlang
1> Document = json:decode(~"""
{
    "Image": {
        "Width": 800,
        "Height": 600,
        "Title": "View from 15th Floor",
        "Thumbnail": {
            "Url": "http://www.example.com/image/481989943",
            "Height": 125,
            "Width": "100"
        },
        "IDs": [116, 943, 234, 38793]
    }
}
""").
#{<<"Image">> =>
      #{<<"Height">> => 600,<<"IDs">> => "tίê鞉",
        <<"Thumbnail">> =>
            #{<<"Height">> => 125,
              <<"Url">> => <<"http://www.example.com/image/481989943">>,
              <<"Width">> => <<"100">>},
        <<"Title">> => <<"View from 15th Floor">>,
        <<"Width">> => 800}}
```

Accessing nested map structures in Erlang today can be achieved using one of
three methods:

1. Accessing a hard-coded path using pattern matching:
   ```erlang
   2> #{<<"Image">> := #{<<"Thumbnail">> := #{<<"Url">> := Url}}} = Document.
   #{<<"Image">> =>
         #{<<"Height">> => 600,<<"IDs">> => "tίê鞉",
           <<"Thumbnail">> =>
               #{<<"Height">> => 125,
                 <<"Url">> => <<"http://www.example.com/image/481989943">>,
                 <<"Width">> => <<"100">>},
           <<"Title">> => <<"View from 15th Floor">>,
           <<"Width">> => 800}}
   3> Url.
   <<"http://www.example.com/image/481989943">>
   ```
2. Recursing with custom functions:
   ```erlang
   4> lists:foldl(
       fun(Key, Acc) -> maps:get(Key, Acc) end,
       Document,
       [<<"Image">>, <<"Thumbnail">>, <<"Url">>]
   ).
   <<"http://www.example.com/image/481989943">>
   % or
   5> Get = fun
       Get([], Value) -> Value;
       Get([Key | Path], Map) -> Get(Path, maps:get(Key, Map))
   end.
   #Fun<erl_eval.18.18682967>
   6> Get([<<"Image">>, <<"Thumbnail">>, <<"Url">>], Document).
   <<"http://www.example.com/image/481989943">>
   ```
3. Repeatedly calling `maps:get/2`:
   ```erlang
   7> maps:get(<<"Url">>,
          maps:get(<<"Thumbnail">>,
              maps:get(<<"Image">>, Document))).
   <<"http://www.example.com/image/481989943">>
   ```

All of the above methods have their drawbacks, commonly that they are either not
dynamic (meaning the path is hard-coded at compile time) or have poor failure
modes in case of unexpected input leading to hard to debug errors.

However, the major drawback shared by all methods described above is that they
require custom implementations everytime the functionality is needed. If they
would exist in the standard library, it would reduce community duplication of
this common functionality.

In contrast, the functionality in this EEP would let the user get the URL by the
following call:

```erlang
get_thumbnail_url(Document) ->
    maps:deep_get([<<"Image">>, <<"Thumbnail">>, <<"Url">>], Document).
```

In addition it is easy to implement other functionality that modify the
structure, that would otherwise require custom traversal code:

```erlang
set_thumbnail_url(Document, Url) ->
    maps:deep_put([<<"Image">>, <<"Thumbnail">>, <<"Url">>], Url, Document).

update_document(Document, NewValues) ->
    maps:deep_merge(Document, NewValues).

get_heights(Document) ->
    get_heights(maps:deep_next(maps:deep_iterator(Document)), []).

get_heights(none, Heights) ->
    Heights;
get_heights({Path, Value, Iterator}, Heights) ->
    case lists:last(Path) of
        <<"Height">> when is_integer(Value) ->
            get_heights(Iterator, [Value|Heights]);
        _ ->
            get_heights(Iterator, Heights)
    end.
```

Specification
=============

This section details new types, errors and functions that are introduced by this
EEP. All functions are purely functional and side-effect free, thus no thread
safety considerations are necessary.

Types
-----

The following new types are introduced as part of the new API:

### `path/0`

```erlang
-type path() :: [term()].
```

A path is a sequence of keys that leads to a value in a nested map. A path must
always be a list of terms. As with normal maps, keys inside a path list can be
any term, including lists.

If an argument to a function that takes a path is not a list, the function will
raise an `{badpath, Arg}` error.

### `deep_iterator/0`

```erlang
-opaque deep_iterator() :: term().
```

A deep iterator is an opaque type is returned from the new [deep iterator
function](#deep_iterator-1), and can be used by the [deep next
function](#deep_next-2) to traverse a nested map.

### `combiner/0`

```erlang
-type combiner() ::
    fun((Path :: path(), Old :: term(), New :: term()) -> term()).
```

A combiner is a function of arity 3 that takes a path and two values and returns
a new value. It is used to decide the final value when two values are
conflicting at the same path.

Errors
------

The following new errors are introduced as part of the new API, specific to
situations encountered while operating on deep maps:

### `badpath`

The exception `{badpath, Path}` of type `error` is raised when a path is not a
list of terms. The exception always carries the supplied bad value in a tuple.

### `badvalue`

The exception `{badvalue, PartialPath, Value}` of type `error` is raised when a
value `Value` is not a map at an intermediate key in the path `PartialPath`.
This happens when a function tries to traverse a path and encounters an
intermediary value that is not a map.

Functions
---------

This proposal introduces new functions to the `maps` module for accessing nested
maps in five different areas:

* Getters
    * [`deep_get/1`](#deep_get-1)
    * [`deep_get/3`](#deep_get-3)
    * [`deep_find/2`](#deep_find-2)
    * [`deep_search/2`](#deep_search-2)
* Setters
    * [`deep_put/3`](#deep_put-3)
    * [`deep_update/3`](#deep_update-3)
    * [`deep_update_with/3`](#deep_update_with-3)
    * [`deep_update_with/4`](#deep_update_with-4)
    * [`deep_remove/2`](#deep_remove-2)
* Traversal
    * [`deep_iterator/1`](#deep_iterator-1)
    * [`deep_next/1`](#deep_next-1)
* Combinators
    * [`deep_merge/1`](#deep_merge-1)
    * [`deep_merge/2`](#deep_merge-2)
    * [`deep_merge_with/2`](#deep_merge_with-2)
    * [`deep_merge_with/3`](#deep_merge_with-3)
    * [`deep_intersect/2`](#deep_intersect-2)
    * [`deep_intersect_with/3`](#deep_intersect_with-3)
* Transformations
    * [`inverse/1`](#inverse-1)
    * [`inverse/2`](#inverse-2)

### Getters

#### `deep_get/1`

```erlang
-spec deep_get(Path :: path(), Map :: map()) -> Value :: term().
```

Returns value `Value` associated with `Path` if `Map` contains `Path`.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`
* `{badkey,Path}` if no value is associated with path `Path`

#### `deep_get/3`

```erlang
-spec deep_get(Path :: path(), Map :: map(), Default :: term()) ->
    Value :: term().
```

Returns value `Value` associated with `Path` if `Map` contains `Path`. If
no value is associated with `Path`, `Default` is returned.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`

#### `deep_find/2`

```erlang
-spec deep_find(Path :: path(), Map :: map()) -> {ok, Value :: term()} | error.
```

Returns a tuple `{ok,Value}`, where Value is the value associated with `Path`,
or `error` if no value is associated with `Path` in `Map`.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `{badpath,Path}` if `Path` is not a path

#### `deep_search/2`

```erlang
-spec deep_search(Path :: path(), Map :: map()) ->
    {ok, Value :: term()}
    | {error, PartialPath :: path(), FoundValue :: term()}.
```

Returns a tuple `{ok,Value}` where `Value` is the value associated with `Path`,
or `{error, PartialPath, Value}` if no value is associated with `Path` in `Map`,
where `PartialPath` represents the path to the last found element in `Map` and
`Value` is the value found at that path.

When no key in `Path` exists in `Map`, `{error, [], Map}` is returned.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `{badpath,Path}` if `Path` is not a path

### Setters

#### `deep_put/3`

```erlang
-spec deep_put(Path :: path(), Value :: term(), Map1 :: map()) -> Map2 :: map().
```

Associates `Path` with value `Value` and inserts the association into map
`Map2`. If path `Path` already exists in map `Map1`, the old associated value
is replaced by value `Value`. The function returns a new map `Map2` containing
the new association and the old associations in `Map1`.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map1` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`

#### `deep_update/3`

```erlang
-spec deep_update(Path :: path(), Value :: term(), Map1 :: map()) ->
    Map2 :: map().
```

If `Path` exists in `Map1`, the old associated value is replaced by value
`Value`. The function returns a new map `Map2` containing the new associated
value.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map1` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`
* `{badkey,Path}` if no value is associated with path `Path`

#### `deep_update_with/3`

```erlang
-spec deep_update_with(
    Path :: path(), Fun :: fun((term()) -> term()), Map1 :: map()
) ->
    Map2 :: map().
```

Update a value in a `Map1` associated with `Path` by calling `Fun` on the old
value to get a new value.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map1` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`
* `{badkey,Path}` if no value is associated with path `Path`
* `badarg` if `Fun` is not a function of arity 1

#### `deep_update_with/4`

```erlang
-spec deep_update_with(
    Path :: path(), Fun :: fun((term()) -> term()), Init :: any(), Map1 :: map()
) -> Map2 :: map().
```

Update a value in a `Map1` associated with `Path` by calling `Fun` on the old
value to get a new value. If `Path` is not present in `Map1` then `Init` will be
associated with `Path`.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map1` is not a map
* `{badpath,Path}` if `Path` is not a path
* `{badvalue,P}` if a term that is not a map exists as a intermediate key at the
  path `P`
* `badarg` if `Fun` is not a function of arity 1

#### `deep_remove/2`

```erlang
-spec deep_remove(Path :: path(), Map1 :: map()) -> Map2 :: map().
```

Removes the last existing key of `Path`, and its associated value from
`Map1` and returns a new map `Map2` without that key. Any deeper non-existing
keys are ignored.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `{badpath,Path}` if `Path` is not a path

### Traversal

#### `deep_iterator/1`

```erlang
-spec deep_iterator(Map :: map()) -> Iterator :: deep_iterator().
```

Returns a map iterator `Iterator` that can be used by `deep_next/1` to
recursively traverse the path-value associations in a deep map structure.

The call fails with a `{badmap,Map}` exception if Map is not a map.

#### `deep_next/1`

```erlang
-spec deep_next(Iterator1 :: deep_iterator()) ->
    {Path :: path(), Value :: term(), Iterator2 :: deep_iterator()}
    | none.
```

Returns the next path-value association in `Iterator` and a new iterator for the
remaining associations in the iterator.

If the value is another map the iterator will first return the map as a value
with its path. Only on the next call the inner value with its path is returned.
That is, first `{Path, map(), iterator()}` and then `{InnerPath, Value,
iterator()}`.

If there are no more associations in the iterator, `none` is returned.

### Combinators

#### `deep_merge/1`

```erlang
-spec deep_merge(Maps :: [map()]) -> Map :: map().
```

Merges a list of maps recursively into a single map.

If a path exist in several maps, the value in the first nested map is superseded
by the value in a following nested map.

That is, `deep_merge/2` behaves as if it had been defined as follows:

```erlang
deep_merge(Maps) when is_list(Maps) ->
    deep_merge_with(fun(_Path, _V1, V2) -> V2 end, Maps).
```

The call can raise the following exceptions:

* `{badmap,Map}` exception if any of the maps is not a map

#### `deep_merge/2`

```erlang
-spec deep_merge(Map1 :: map(), Map2 :: map()) -> Map :: map().
```

Equivalent to `deep_merge([Map1, Map2])`.

#### `deep_merge_with/2`

```erlang
-spec deep_merge_with(Fun :: combiner(), Maps :: [map()]) -> Map :: map().
```

Merges a list of maps `Maps` recursively into a single map.

If a path exist in several maps, the function `Fun` is called with the path, the
previous and the conflicting value to resolve the conflict. The return value
from the function is put into the resulting map.

The call can raise the following exceptions:

* `{badmap,Map}` exception if any of the maps is not a map
* `badarg` if `Fun` is not a function of arity 3

#### `deep_merge_with/3`

```erlang
-spec deep_merge_with(Fun :: combiner(), Map1 :: map(), Map2 :: map()) ->
    Map3 :: map().
```

Merges two maps `Map1` and `Map2` recursively into a single map.

If a path exist in several maps, the function `Fun` is called with the path, the
previous and the conflicting value to resolve the conflict. The return value
from the function is put into the resulting map.

The call can raise the following exceptions:

* `{badmap,Map}` exception if any of the maps is not a map
* `badarg` if `Fun` is not a function of arity 3

#### `deep_intersect/2`

```erlang
-spec deep_intersect(Map1 :: map(), Map2 :: map()) -> Map3 :: map().
```

Intersects two maps into a single map `Map3`.

If a path exists in both maps, the value in `Map1` is superseded by the value in
`Map2`.

That is, `deep_intersect/2` behaves as if it had been defined as follows:

```erlang
deep_intersect(A, B) ->
    deep_intersect_with(fun(_Path, _V1, V2) -> V2 end, A, B).
```

The call can raise the following exceptions:

* `{badmap,Map}` exception if any of the maps is not a map

#### `deep_intersect_with/3`

```erlang
-spec deep_intersect_with(Fun :: combiner(), Map1 :: map(), Map2 :: map()) ->
    Map3 :: map().
```

Intersects two maps into a single map `Map3`.

If a path exists in both maps, the value in `Map1` is combined with the value in
`Map2` by the `Combiner` fun. When `Combiner` is applied the path that exists in
both maps is the first parameter, the value from `Map1` is the second parameter,
and the value from `Map2` is the third parameter.

The call can raise the following exceptions:

* `{badmap,Map}` exception if any of the maps is not a map
* `badarg` if `Fun` is not a function of arity 3

### Transformations

#### `inverse/1`

```erlang
-spec inverse(Map1 :: map()) -> Map2 :: map().
```

Inverts a map by inserting each value as the key with its corresponding key as
the value. If two keys have the same value, the value for the first key in map
order will take precedence.

That is, `inverse/1` behaves as if it had been defined as follows:

```erlang
inverse(Map) -> inverse(Map, fun(Old, _New) -> Old end).
```

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map

#### `inverse/2`

```erlang
-spec inverse(
    Map1 :: map(),
    Fun :: fun((Old :: term(), New :: term()) -> Value:: term())
) ->
    Map2 :: map().
```

Inverts a map by inserting each value as the key with its corresponding key as
the value. If two keys have the same value in `Map`, `Fun` is called with the
old and new key to determine the resulting value.

The call can raise the following exceptions:

* `{badmap,Map}` if `Map` is not a map
* `badarg` if `Fun` is not a function of arity 2

Backwards Compatibility
=======================

Since this EEP only introduces new functions, there are no backwards
incompatible changes.

Rationale
=========

The above implementation is chosen to be familiar to Erlang developers that have
experience with the existing `maps` module. It retains similarity between
arguments and errors as far as possible.

Scope
-----

The scope of this EEP is strictly limited to providing a way to access nested
maps structures in a concise and consistent manner. Concise in that it should
represent a generic but powerful set of functions covering most common use
cases, and consistent in that it should be familiar to OTP developers in that it
mimics the existing `maps` module API as closely as possible.

### Not in Scope

#### Support for any other nested data structure

This includes, but is not limited to, lists and their subtype proplists. It is
out of scope of this EEP to provide a generic access API to be able to traverse
paths where a list is encountered.

One could imagine accessing an element in a list by its index, using a path like
`[foo, 1, bar]` but this conflicts with maps that could also have integers as
keys. For example, what should be the result of the following expression:

```erlang
maps:deep_merge(#{1 => [true]}, [#{1 => false}]).
```

Proplists pose further problems since the elements allowed in a proplist are not
consistent. Some elements are key-value tuple pairs, while others are just
values or even tuples bigger than two elements. This inconsistency makes it
difficult to define a generic access API for some of the functions proposed in
this EEP. For example, what should be the result of the following expression:

```erlang
maps:merge(#{a => 1}, [{a, 2, extra}, b]).
```

These are ambiguities that this EEP does not intend to solve (see the
[Lenses](#lenses) chapter under [Alternatives
Considered](#alternatives-considered)).

Alternatives Considered
-----------------------

### Separate Module

The functions proposed above could all be placed in a separate module to avoid
blowing up the standard `maps` module API too much. For example, `maps_nested`
or `maps_deep`. This EEP proposes to add these functions to the `maps` module
for ease of access to all functionality related to maps.

#### Prior Art

There are several similar community packages that solve this problem. A
non-exhaustive list:

* [`maps_in`][]
* [`nested`][]
* [`mapz`][] (the reference implementation of this EEP)

### Pattern Matching

In [EEP 43][] where maps were first proposed, a pattern matching syntax for
accessing values was introduced:

> Consider the following,
>
>     V = M#{ K }
>
> is shorter than
>
>     #{ K := V } = M
>
> It also allows for easy access of associated values in deep structures.
>
> The syntax for single value access is the least developed (and contemplated)
> feature in this proposal and certainly could use some input.

Although it is unclear what is meant by "easy access of associated values in
deep structures," one could imagine some hypothetical syntax support for this:

```erlang
Url = Document#{<<"Image">>}{<<"Thumbnail">>}{<<"Url">>}
```

As an alternative implementation it is interesting as a way to access static
paths (defined at compile time) but it does not solve the problem of supporting
dynamic paths built at runtime and is therefore not suitable for the problems
intended to be solved by this EEP.

### Lenses

There has been previous discussions of support for lenses in Erlang. Lenses
provide a way to access and modify values within nested data structures,
regardless of the surrounding structure. That is, a lens implementation could
support setting a deep value inside a tuple containing a list containing a map.

Since there are many potential ambiguities when mixed data structures are
accessed, this EEP only proposes an API for maps that is consistent with the
existing `maps` API and that does not impose any limitations on the data
structures being accessed (such as allowing that maps can have any Erlang term
as keys). See the [Scope](#scope) section for further discussion.

#### Prior Art

Lens implementations in Erlang:

* [`erl-lenses`][]

Non-exhaustive list of implementations in other languages:

* [Haskell's `lens` library][]
* [PureScript's `profunctor-lenses` library][]

### Protocols

Protocols have been previously proposed for Erlang[[1]] and could partially
solve the problem of accessing nested data structures. The example closest to
Erlang would be Elixir's `Access` protocol. It allows the syntax `map[:a][:b]`
to work with any data structure implement this protocol. Currently this is
supported by maps and keyword lists in Elixir.

The implementation comes in two parts:

* The protocol specification together with the actual traversal
  algorithm implementation
* Implementation of the necessary API by the respective data structures

The functionality of this EEP can be seen as a hybrid of the two concepts into
one single implementation that only supports the maps data structure.

Protocols have similar limitations to those of lenses, in that they deal poorly
with ambiguous directives regarding path traversal and manipulation. See the
[Scope](#scope) section for further discussion.

Reference Implementation
========================

The reference implementation for this proposal is the `mapz` library that can be
found on [Hex][mapz_on_hex] and on [GitHub][mapz_on_github]. The documentation
for the library is available at [HexDocs][mapz_documentation]. The reference
implementation has 100% test coverage of all functions defined in this EEP.

To try it out, you can clone the repository and start a shell session:

```console
$ git clone git@github.com:eproxus/mapz.git
Cloning into 'mapz'...
remote: Enumerating objects: 388, done.
remote: Counting objects: 100% (253/253), done.
remote: Compressing objects: 100% (142/142), done.
remote: Total 388 (delta 128), reused 195 (delta 81), pack-reused 135 (from 1)
Receiving objects: 100% (388/388), 99.11 KiB | 554.00 KiB/s, done.
Resolving deltas: 100% (187/187), done.
$ cd mapz
$ rebar3 eunit
...
===> Compiling mapz
===> Performing EUnit tests...
..............................................................................................
94 tests passed
$ rebar3 shell
===> Verifying dependencies...
===> Analyzing applications...
===> Compiling mapz
Erlang/OTP 27 [erts-15.2.2] [source] [64-bit] [smp:12:12] [ds:12:12:10] [async-threads:1] [jit]

Eshell V15.2.2 (press Ctrl+G to abort, type help(). for help)
1> Document = json:decode(~"""
   {
       "Image": {
           "Width": 800,
           "Height": 600,
           "Title": "View from 15th Floor",
           "Thumbnail": {
               "Url": "http://www.example.com/image/481989943",
               "Height": 125,
               "Width": "100"
           },
           "IDs": [116, 943, 234, 38793]
       }
   }
   """).
#{<<"Image">> =>
      #{<<"Height">> => 600,<<"IDs">> => "tίê鞉",
        <<"Thumbnail">> =>
            #{<<"Height">> => 125,
              <<"Url">> => <<"http://www.example.com/image/481989943">>,
              <<"Width">> => <<"100">>},
        <<"Title">> => <<"View from 15th Floor">>,
        <<"Width">> => 800}}
2> mapz:deep_get([<<"Image">>, <<"Thumbnail">>, <<"Url">>], Document).
<<"http://www.example.com/image/481989943">>
```

Performance
-----------

The reference implementation is focused on correctness first, and performance
second. Internally it does implement some generic protocol-like traversal
functionality used to build the high-level API. Alternatively custom
implementation could be considered for each API function if that would increase
performance. This would of course come with the cost of increased duplication
and a higher maintenance burden.

Open Questions
==============

* [ ] Settle terminology between 'deep' or 'nested'.
* [ ] Any other significant prior art that should be referenced?
* [ ] Is `Access` a 'protocol' in Elixir (or a 'behaviour')?
* [ ] What should the final implementation look like? Is the reference
      implementation good enough?
* [ ] Should `inverse/1` and `inverse/2` be introduced as part of the new API
      or be removed from this EEP?

[EEP 43]: eep-0043.md
    "EEP-43: Maps"

[`maps_in`]: https://github.com/williamthome/maps_in
    "maps_in library by William Fank Thomé"

[`nested`]: https://github.com/odo/nested
    "nested library by Odronitz, Bader & Huning"

[`mapz`]: https://github.com/eproxus/mapz
    "mapz library by Adam Lindberg"

[`erl-lenses`]: https://github.com/jlouis/erl-lenses
    "erl-lenses library by Jesper Louis Andersen"

[Haskell's `lens` library]: https://hackage.haskell.org/package/lens
    "Haskell's lens library"

[PureScript's `profunctor-lenses` library]: https://pursuit.purescript.org/packages/purescript-profunctor-lenses
    "PureScript's profunctor-lenses library"

[mapz_on_hex]: https://hex.pm/packages/mapz
    "mapz library on Hex.pm"

[mapz_on_github]: https://github.com/eproxus/mapz
    "mapz library on GitHub"

[mapz_documentation]: https://hexdocs.pm/mapz
    "mapz library documentation on HexDocs.pm"

Copyright
=========

This document is placed in the public domain or under the CC0-1.0-Universal
license, whichever is more permissive.

[1]: https://erlangforums.com/t/the-need-for-protocols-in-erlang/2973

[EmacsVar]: <> "Local Variables:"
[EmacsVar]: <> "mode: indented-text"
[EmacsVar]: <> "indent-tabs-mode: nil"
[EmacsVar]: <> "sentence-end-double-space: t"
[EmacsVar]: <> "fill-column: 70"
[EmacsVar]: <> "coding: utf-8"
[EmacsVar]: <> "End:"
[VimVar]: <> " vim: set fileencoding=utf-8 expandtab shiftwidth=4 softtabstop=4: "
