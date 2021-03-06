# `ci` Modifier

Introduces a universal modifier into the query language so that case-insensitive forms of various operations can be uniformly expressed.

## Motivation

Text searches and comparisons are a central part of the Seq user experience. In general, every text search or match operation needs both a case-sensitive and case-insensitive variant.

Unfortunately, operators like `=` (equality) have no widely-understood case-insensitive forms.

### Text fragments

The Seq 1.0 "filter language", used for constructing predicates that identify single events in log searches, solved this by supporting two _types_ of text fragment, and dispatching operators based on the types of their operands:

 * `"x"` is a case-insensitive text fragment, while
 * `@"x"` is case sensitive.

This originated with the need to construct searches based on logical expressions including text fragments, e.g. `"http" or @"UI"` will match events with messages that include any variation of HTTP, but only the exact case UI.

The feature was expanded to cover various other comparisons: the expression `A = "x"` uses a case-insensitive test, while `A = @"x"` will test exact equality. Certain functions, like `IndexOf(A, "x")` dispatch to case-insensitive variants based on the types of their arguments.

This solution is still supported in Seq 5, but it suffers from a few problems:

 * the syntax does not extend to case-modified comparisons of non-literal text: `A = B` has no easily-discoverable case-insensitive variant;
 * the concept of using the operand types to select an operator adds additional cognitive overhead;
 * constructs like `["hello", @"world"]` are ambiguous - this is an array of strings, but could also be expected to produce an array of Booleans; and,
 * double-quoted text clashes with the usual interpretation of double quotes as escaped identifiers in SQL-style queries, and the `@` prefix similarly clashes with common SQL-style identifiers.

### Omission of text fragments from the current SQL syntax

Because of thes issues, and a general feeling that this system wasn't the right solution, `"text fragments"` are not supported in Seq's newer SQL-style syntax, where only `'strings'` are allowed. As a workaround, the SQL `like` operator was made case-insensitive, enabling case-insensitive versions of equality, contains, and prefix/suffix comparisons.

Some holes remain:

 * SQL queries lack an idiomatic case-insensitive `IndexOf()`, with the awkward `IndexOfIgnoreCase()` standing in for this.
 * Similarly, there's no case-insensitive variant of `in`.
 * Case-insensitive `distinct()`, `group by`, and `order by` have no natural syntax, and
 * Case-insensitive `like` is surprising for some users.

SQL implementations generally approach this problem with `ToLower()` - for example `ToLower(A) = ToLower(B)`. Seq could support this, too, but without careful optimization, query execution might result in the creation of expensive temporary strings, and given text search is so central to Seq's use case, this solution is thought to be too verbose.

### The universal `ci` modifier

This RFC introduces a syntax extension, `ci`, as a modifier on operations that may work with text, both in expressions:

```
A = B ci
```

And to modify structural SQL clauses:

```
select percentile(Elapsed, 99)
from stream
group by RequestPath ci
```

The `ci` suffix modifies operators, rather than operands, and can be applied to any case-variant operator including calls - for example, `IndexOf(A, 'x') ci`.

## Detailed design

### Case-insensitive expressions

The `ci` modifier is accepted following the right-hand operand of any comparison or function call, including:

 - infix `=`, `<>`, `like`, `in`
 - funtion-call `Equal()`, `NotEqual()`, `StartsWith()`, `EndsWith()`, `Contains()`, `IndexOf()`, `LastIndexOf()`, `ElementAt()`

It binds along with the operator, so precendence works in the normal manner. In the following example, `C = D` is case-insensitive:

```
A = B or C = D ci
```

### Case-insensitive groupings and orderings

The `ci` modifier can follow a grouping or ordering in an SQL-style query:

```
select count(*)
from stream
group by Product ci, Store
```

When specified for an ordering, the modifier must precede the direction (when present):

```
select Name
from stream
order by Name ci desc
```

### Case-insensitive aggregation

The `distinct()` and `count(distinct())` aggregates can support `ci`. While this produces unattractive mixed-case output from `distinct()`, supporting `ci` in a universal fashion is advantageous enough to justify adding this feature.

### Case-insensitive property access

At least one request has been made for a case-insensitive way to access properties on events, e.g. `userId` vs `UserId`. The `@Properties['userid'] ci` option was considered as part of this feature, but it would cause confusing parses of `'Foo' = @Properties['some name'] ci` (intended `ci` comparison, but actually `ci` accessor), and `group by @Properties['some name'] ci` (`ci` accessor instead of `ci` grouping).

Instead, `ci` can be applied to a helper function `ElementAt(@Properties, 'userid') ci` to provide this without the drawbacks of `ci` indexers. `ElementAt()` will support all structured objects, not only `@Properties`.

#### Edge-case

There is a syntactic edge case:

```
select count(*)
from stream
group by Application ci
```

will perform a case-insensitive grouping, because the no comparison is present to consume the `ci`, while:

```
select count(*)
from stream
group by Substring(A, 0, 3) ci
```

will fail because `ci` applies to the function call. The usual strategy of "add parens until it works" applies here:

```
select count(*)
from stream
group by (Substring(A, 0, 3)) ci
```

### Breaking change in `like` evaluation

The `like` operator, previously case-insensitive, will become case-sensitive with this change. Existing data will need to be automatically migrated to preserve semantics.

### Deprecation of case-sensitive text fragments

While `"x" and "y"` text fragment searches are still a valuable part of the filter language, the less-discoverable case-insensitive variants `@"x" and @"y"` can be deprecated along with this change.

The double-quoted forms are intended to be guessable, so that someone searching Seq for the first time can start to form logical filters by guessing at the syntax. The `@`-prefixed case-insensitive forms are not guessable, and are considered to be at a syntactic dead-end, and so they should be removed from the documentation and replaced with examples showing `@Message like '%x%' ci and @Message like '%y%' ci`. While the `ci` version of the syntax is more verbose, it can be applied throughout the query language.

### Deprecation of text-fragments-as-operands

Text fragments on their own are Boolean, while in operand position in filters they're treated like strings. This usage in the place of strings should be deprecated, e.g. in future versions we expect `"x" and "y"` to continue to be supported, but `A = "B"` will not be.

Narrowing the role of text fragments to standalone Boolean conditions improves our chances of finding a way to incorporate them cleanly into the SQL syntax, (for example, as an illustration only, using a mini-language within some kind of delimiters like `where text("x" and "y")`).

## Future directions

### Ordering comparison of string values

Currently the less-than/greater-than operators do not support string values, but should they do so in the future, they can also support the `ci` modifier.

Applying `ci` to `<`, `<=`, `>`, or `>=` will be a hard error, ensuring the syntax can be supported in the future without breaking changes.

## Drawbacks

Edge cases and breaking changes discussed above.

## Alternatives

The primary alternatives to this feature are:

1. Rely on `ToLower()` to emulate case-insensitive comparisons, e.g. `ToLower(A) = 'foo'` or `group by ToLower(A)`. This will be supported through the introduction of `ToLower()` anyway, but is unappealing for reasons mentioned in _Motivation_ above.
2. (In addition to 1.) add `EqualsIgnoreCase()` and similar functions to the language; this does not cover grouping, and is rather verbose.
3. Extend use of `"text fragments"` and embed these further in the SQL language, for example `group by ci(A)` would dynamically construct a case-insensitive text fragment from `A`, as if a literal double-quoted string were used in place of `A`. Drawbacks mentioned in _Motivation_.
4. Prefix expression-level `ci` with suffix structural `ci`, e.g. `ci A = B` and `group by A ci` can be combined into the unambiguous (but ugly) `group by ci A = B ci`. The edge case is removed, but at the expense of ergonomics. (Function-call `ci()` in expressions is a variant of this that was also considered).

## References

 * https://github.com/datalust/seq-tickets/issues/687
 * https://github.com/datalust/seq-tickets/issues/894

