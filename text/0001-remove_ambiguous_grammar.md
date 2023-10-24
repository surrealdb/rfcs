# SurrealQL: Remove ambiguity in grammar

Status: Open for suggestion
Creator: Mees Delzenne
Owner: Mees Delzenne
Last edited time: October 11, 2023 3:56 PM
tag: SurrealQL

### 1. Summary

The current version of SurrealQL grammar, as defined by what the parser currently accepts, contains several ambiguous productions. These productions are parsed differently depending on the context or can seem very similar to each other, but subtle differences can result in completely different semantics.

These ambiguities in the grammar complicate parser design, limit possible future extensions to SurrealQL that don't break, and could be confusing when using the language.

This RFC proposes several changes to the grammar to limit the present ambiguity:

- Disallow a value from starting with an identifier which could, in its current position in the grammar, start a statement as a keyword.
- Require that raw identifiers don't start with a digit.
- Introduce strand prefixes for the specific types of strands
- Introduce a syntax error for block record-id object ambiguity
- Change the KNN operator from `<3>` to `knn<3>` or similar

#### Glossary

- Production: A branch or leaf of the parse tree.
- Identifier: Names used in code which don't belong to the structure of a statement.
- Raw Identifier: A identifier without being surrounded by ` or `⟨ ⟩`
- Keyword: Any identifier like production which belongs to the structure of a statement.
- Reserved word: A keyword which can’t be used as a identifier.
- Surrounded Identifier: An identifier which is surrounded by a delimiter, allowing otherwise disallowed identifiers to be used. Example: ``foo\\nbar``
- Strand: A string like production i.e. something like `"hello world"`

### 2. Motivation

What motivates this proposal and why is it important?

#### Context

The first implementation of the SurrealQL parser was implemented in Nom. Nom is a useful library for building a parser; however, its flexibility can also be a downside. Nom functions combine to form a tree of smaller sub-parsers, which together define a full parser. In Nom, whenever the grammar to be parsed contains a branch, Nom decides which branch to follow by taking whichever branch first parses without error. For example, see the following source code:

```
a - 1
```

and the following parser:

```jsx
alt((
	tuple((ident, char('+'), number)),
	tuple((ident, char('-'), number))
))
```

In this case Nom will first try to parse the first branch, fail when it encounters `char('+')` sub-parser and then back up to try the second example.  Nom imposes no limits on when and how far the parser will back up whenever an error happens, only failing when it has completed a full sweep of the entire parse tree.

This ability to backup and try again allows one to produce quite powerful parsers by composing simple functions. Nom is able to parse basically any grammar if sometimes with exponential complexity.

SurrealQL was build with this parser and the flexibility of the parser can be noticed in the current flexibility of the language. The SurrealQL grammer uses backup to define a grammar which has ambiguity which is only resolved by the order by which the parser parses a production.

#### Flexibility vs Clear semantic meaning.

The current grammar flexibility does allow for a more expressive syntax: more code will parse without errors, and you will often need fewer characters to write a query than if the language were more rigid.

However, I feel that this flexibility is limiting other goals that I would consider more desirable, such as the ability to precisely understand the semantics of a query. Currently, a full understanding of what a query exactly means requires quite deep knowledge of how the parser functions, not just if a token is allowed, but also what other possible tokens are allowed and how the parser will retry parsing when a parse branch fails to match.

To illustrate I have two examples.

Consider this first example:

```sql
if CONTAINS { 3 }
```

The above source is completely ambiguous and can both be parsed as an `IF` statement with as condition `CONTAINS` and as its `THEN` block `{ 3 }`, or as an expression where we check if the identifier `if` contains `{ 3 }`. The current parser will parse it as an if statement only because that is the first branch in the parser which matches.

This could result in confusion, consider the following code:

```sql
DELETE some_data FROM values WHERE if CONTAINS true
```

The above query checks for deletion if the identifier `if` contains true. It is not parsed as an `IF` statement since the parser fails when trying to parse a code block when it finds the `true` token, and therefore falls back to parsing a the production as a value.

Now suppose the user wanted to make a change and add a expression block:

```sql
DELETE some_data FROM values WHERE if CONTAINS { true }
```

Now the statement is suddenly parsed as an IF statement and, if the table happens to have a `CONTAINS` field, it will still run but with a completely different meaning.

A different type of ambiguity in today's grammar is token ambiguity.
Most languages try to limit the possible meaning of tokens as much as possible.
As apposed to most languages in SurrealQL raw Identifiers can start with a digit. This means that a number is a valid identifier, and a number with a fraction could also be parsed as a identifier and its field. This can make it hard to determine what is an identifier. Consider the following example:

```sql
UPDATE table SET 10dec = "a value" WHERE 10dec = "a different value";
```

In the above query the field `10dec` will be updated to the value "a value" if the number 10 is equal to "a different value". In SET you are not allowed to have any numbers so `10dec` which is a valid identifier will be parsed as an identifier. In the where clause however numbers are allowed so `10dec` will in that position be parsed as number.

More examples of these types of confusing grammar can probably be found.

#### Hard to specify

As we are working towards a production ready database we should probably eventually create an official specification for the SurrealQL language. Most languages are either Context-Free or almost Context-Free and therefore allow specifying the grammar in Context-Free grammar. See for example the JavaScript spec: [https://tc39.es/ecma262/#prod-Statement](https://tc39.es/ecma262/#prod-Statement)

The current ambiguities in the language make it impossible to specify a grammar for SurrealQL in the same way.

#### Extendibility

Because a lot of the current syntax is defined in part by the failure to parse other productions, productions which today are parsed one why could in the future silently change to have a completely different semantic meaning.

Consider the following query:

```sql
SELECT count FROM table GROUP count
```

Currently the values from this select will be grouped by the field key. But where we to every introduce some feature which would require the `COUNT` keyword after `GROUP` then this code will quietly change meaning.

### 3. Proposal

#### Location specific reserved words.

The first change proposed is to disallow raw identifiers in places where the same identifier could also be parsed as a keyword. For example, the following code would no longer be allowed:

```sql
use = 32;
```

`use` in this case could be the start of a USE statement in this position and would therefore not be allowed. In order to get the same code one would need to surround the identifier.

```sql
`use` = 32;
```

Another example would be:

```sql
SELECT field FROM use;
```

The part after `FROM` can also have statements, however the USE statement is disallowed in subqueries, therefore allowing `use` would not be a problem.

This change would completely remove existing ambiguity wherever an expression could possibly be parsed as a statement. It would simplify the parser a great deal. This changes also does not require that raw identifiers are completely distinct.

#### Disallow raw identifiers to start with a digit

This is a common way to distinguish between a digit and number. By disallowing the first character from being a digit we can be sure that Identifiers aren’t numbers. This would resolve the same text from being used as an identifier or a number depending on the place in the grammar.

#### Introduce strand prefixes

The only way can currently be distinguished is if one strand fails to parse as any of the other types. This can lead to problems where a use wanted a certain value to be a plain strand but it happened to match another type of strand and is thus converted into that specific type.

And example is `"5:00"` which brought up in an issue. The user wanted to store this value as a plain strand but it happened to match a thing strand.

I propose we introduce specific strand prefixes for specific strand types:

- `t"2012-05-26T12-00-00Z"` for date-time strands
- `u'7c3f4ce8-83c4-458f-b1d7-b28352dea93c'` for UUID’s
- `r"5:00"` for record id strings

#### Syntax Error for Block-RecordId Object ambiguity

The following statement is ambiguous:

```sql
{ a:b.c }
```

This can be parsed either as an object with a field `a` and value `b.c`, or as a block statement taking the field `c` of the record `a:b`. Removing this ambiguity completely would require making significant changes to how record IDs are joined (changing the `:` to something else) or requiring that either blocks or objects always be enclosed by parentheses. Both of these changes are quite drastic for a relatively minor ambiguity.

Therefore, I propose that if the parser encounters `{` `Identifier` `:`, it raises a syntax error to notify the user of the ambiguity. The user would then need to either use a record ID strand if they intended to create a block, or make the identifier a strand if they intended to create an object.

#### 4. Drawbacks

- Some statements now require the use of ` where they previously didn’t

#### 5. Alternatives

### Limited reserved word list

Instead of location specific keywords we could instead specify a limited list of keywords which are disallowed. Keywords which can't start a statement like `EVENT` or `TABLE` would still be allowed, but `USE` would be disallowed everywhere.

This would probably be easier to communicate then location specific keywords

#### 6. Potential Impacts

The proposed changes are breaking changes. They will probably break existing code, if a limited set. As we have guaranteed stability it introducing these changes as the default would require releasing a new major version. Therefore I propose that the new parser will implement this syntax and introduce the new parser as a new experimental feature which in the future will possibly become the default.

#### 7. Unresolved Questions

- What do we change the KNN operator to.

### 8. Conclusion

Here we briefly outline why this is the right decision to make at this time, and move forward!

### Addendum: List of ambiguities

The following is a list of ambiguous SurrealQL statements I encountered. Some of these are solved by the current proposal. Some will still remain.

- KNN operator VS relation operator.

```sql
RETURN 1<2>3  
```

- Assignment or equality

```sql
a = b
```

- Object or block statement with a thing

```sql
{ a:b.c } 
```

- Number or operation on identifiers

```sql
123e+123

123e-123
```

- Outputs keyword or field:

```sql
CREATE foo SET before = 3 RETURN before
```
