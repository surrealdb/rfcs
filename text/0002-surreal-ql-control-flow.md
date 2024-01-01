# SurrealQL: Revised Control Flow

## Glossary

- SurQL: An abbreviation for SurrealQL.
- Scope: A layer that can be used to control the validity of name bindings.
- Phantom state: A value from a canceled transaction.
- Query: A collection of statements.
- Statement: A value or statement.

(I felt it was necessary to define Query and Statement, because there is another thing called a Subquery, which is a better name for what a Statement is, but I didn't make the names)

# 1. Summary
The current state of control flow in SurrealQL is quite convoluted.
Complex queries are subject to excessive nesting, transactions are subject to leaking phantom states, and transactions cannot be nested.

This RFC proposes changes to SurrealQL to improve usability in complex queries while still maintaining flexibility to write simple queries:
- Make `RETURN` statements break execution.
- Introduce block expressions.
- Change the semantics for transactions to allow for greater usability.
- Make `RETURN` semantics more consistent.

<details>
  <summary>Why not use JS?</summary>
  While JavaScript allows for better control flow, it is substantially slower (in my limited testing). This could be improved, but as it stands, a custom DSL will always be faster; SurQL just needs some changes to be more flexible.
</details>

# 2. Motivation
## The initial problem
Currently, the only way to return early is to `THROW` an exception.
This is limited to only returning a string.
And if one tries to use the error message as control flow in application code, it requires a two step process (attempting to deserialize the result and then trying to deserialize the error if that fails) (at least in Rust).
I have found in practice that it is more convenient to return an object with a property that is either `Ok` or `Err` and then deserialize a `Result<T, E>` in Rust.
However, this turns out be a bit messy in the SurQL side. The following that uses `THROW`:
```sql
{
  IF !$foo.id {
    THROW "NotFoo";
  };

  IF !$bar.id {
    THROW "NotBar";
  };

  RETURN SELECT bar[WHERE name = $bar.name] FROM ONLY $foo;
}
```
suddenly becomes:
```sql
RETURN IF !$foo.id {
  RETURN { Err: "NotFoo" };
} ELSE {
  RETURN IF !$bar.id {
    RETURN { Err: "NotBar" };
  } ELSE {
    RETURN { Ok: (SELECT bar[WHERE name = $bar.name] FROM ONLY $foo)};
  };
};
```

This quickly becomes messy as more conditions are added.
But maybe this is just the wrong way of doing things?
Personally, I believe it is the correct route for my particular application, it's just that there are potholes in the road caused by SurrealQL's current limitations.
Creating complex queries like this allows for executing more code on the database and avoids extra queries and the latency cost they incur.

## The problems with `RETURN`
The (current) purpose of the `RETURN` statement is to set the return value for the current scope.
However, the actual semantics are quite convoluted.
To start, what is scope?
In SurrealQL, there is:
- Global scope, which provides query bindings.
- Blocks, which provide a static/lexical scope.
- Transactions, which create a dynamic scope of sorts (essentially, the data can be invalid if the transaction is canceled, but the data is still available; more on this later).

Let's take a look at calling `RETURN` at each of these scopes:

### Global Scope
```sql
RETURN 300;
RETURN CREATE foo;
# Result: [300, {foo}]
```
The semantics here are straightforward: each `RETURN` is treated as a separate statement and so each one is returned.
However, it is also unnecessary: removing the `RETURN` still has the same result, so the `RETURN` is redundant.

When I discuss transactions later on, we'll see how the semantics established here are broken.

### Blocks
```sql
{
  RETURN CREATE foo;
}
# Result: [{foo}]
```
Here, the semantics start to break down.
First, let's remove the `RETURN` to see the result.
```sql
{
  CREATE foo;
}
# Result: [null]
```
Okay, makes sense. If we aren't returning anything, we shouldn't expect anything, right?
```
{
  1 + 2;
}
# Result: [3]
```
Huh.

It turns out that expressions are implicitly returned, while statements must be explicitly returned.
Now, I'm not against implicit returns (with a better syntax), but I am against this sort of inconsistency.

<details>
  <summary>Aside about repetitive <code>RETURN</code> statements</summary>
  As of writing this, in the documentation it claims the <code>RETURN</code> statements don't break execution.
  However, if you test the example, it clearly breaks execution when it encounters the first <code>RETURN</code> statements.
  Anyways, that's what I should expect (just wanted to clear up any potential confusion), so back to the RFC.
</details>

Let's continue by looking into how these semantics play out when nesting blocks.
```sql
{
  {
    RETURN 300;
  }; # Frankly, not a fan of the required semicolon after blocks; it feels redundant

  RETURN CREATE foo;
}
# Result: [{foo}]
```
It returned the expected value.
Let's try it again to see if blocks are subject to being implicitly returned:
```sql
{
  {
    RETURN CREATE foo;
  };
}
# Result: [{foo}]
```
Yep, so that means both blocks and expressions can be implicitly returned.

Whether or not these results were part of an intentional result, I can't say.
What I can say is that these quirks do little to aid in better code.

### Transactions
Before diving into transactions (`BEGIN/CANCEL/COMMIT`) and their syntax, let's first ask ourselves, are they necessary?
What can be done by a transaction that can't be done by moving the condition for cancellation outside of the transaction?

The answer: nothing.
But that doesn't mean transactions aren't a useful abstraction.
The real benefit of transactions is their ability to make code terse.

However, as transactions currently stand, they are only useful in a REPL.
That is because they are only effective when used as top-level statements; otherwise they do nothing.
This means that they can't be used with `IF-ELSE` statements and that there is an entire part of the language that is useless in most contexts.

Anyways, lets take a look at how transactions work:
```sql
BEGIN;
CREATE foo;
RETURN 300;
COMMIT;
# Result: [300]
```
Huh, so `RETURN` is used to set the result of the transaction.
This looks like it breaks the established semantics for the global scope, but I suppose if we think of `BEGIN-COMMIT` as a new scope (using keywords instead of braces) it makes sense.
What happens if we remove that `RETURN`?
```sql
BEGIN;
CREATE foo;
300;
COMMIT;
# Result: [{foo}, 300]
```
Now we are back to global scope semantics...
Well, I suppose you could make it return nothing by using an empty return statement, right?
```sql
BEGIN;
CREATE foo;
300;
RETURN;
COMMIT;
# Result: [{foo}, 300, null]
```
I see, so there needs to be a non-null result in order to return a single value.
```sql
BEGIN;
CREATE foo;
300;
RETURN null; # or RETURN none;
COMMIT;
# Result: [null]
```
Let's move on...

Another problem with transactions is that they leak canceled state:
```sql
BEGIN;
LET $phantom = CREATE foo;
CANCEL;
RETURN $phantom;
# Result: [canceled, {foo}]
```
What is returned is a record that would have existed if the transaction was commited.
While it may not have a significant impact,
it would be more consistent that when the transaction is canceled,
the changes to the query environment would be reversed along with the changes to the database.

## Recap
So just to recap:
- Current `RETURN` semantics don't allow for early returns.
- `RETURN` semantics changes depending on the context.
- `RETURN` semantics can be quite convoluted and inconsistent even within the same scope.

**tl;dr: there are issues with `RETURN` semantics**

# 3. Proposal
## Early returns and block expressions
First, in order to solve the issue with heavy nesting, returns should break execution all the way up to the current statement.
In order to allow for blocks to return values, block expressions should be introduced.
This means a new syntax, where the last statement in a block is returned if not suffixed with a semicolon.

For example, the first example will now look like:
```sql
{
  IF !$foo.id {
    RETURN { Err: "NotFoo" };
  };

  IF !$bar.id {
    RETURN { Err: "NotBar" };
  };

  { Ok: (SELECT bar[WHERE name = $bar.name] FROM ONLY $foo)}
}
```
