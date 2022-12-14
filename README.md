# Data Austerity Notation

A data notation with the bare essentials. ABNF in the repo in the file [dan.abnf](./dan.abnf).

## Design Goals

- Textual data format
- Support API communication a la JSON
- Support config files and document embedding a la YAML
- Support symbols and keywords in lisps a la EDN
- Support tables a la [MTN](https://GitHub.com/djhaskin987/mtn)
- Support incremental parsing better
- Support typed languages better
- Support better schema checking. It doesn't support it better than JSON,
  but it does encourage it more.

## Syntax

## Comments

Comments must be on their own line. The line begins with an arbitrary
number of blankspace characters. The comment itself starts with a semi-colon
(`;`) character and continues to the end of the line. The entire line
is removed as if it never existed.

## Null

Null, nil or nothing is written `#n`.

## Boolean

These are written `#t` for true and `#f` for false.

## Number

These are written just as JSON numbers, with the addition of the `T` or `t`
characters as stand-ins for the exponent:

`[-+](1/0|0/0|(0|[1-9][0-9]*)(.[0-9]+)?([eE][0-9]+)?)`

Where `1/0` means "Inifinity", `0/0` means "Not a Number", and otherwise it's
a normal number.

## Strings

### Quoted strings

Quoted strings are quoted with double quotes (`"`).

Scheme r7rs-small strings:

- Can escape:
  - The pipe `\|`
  - The quote `\"`
  - Blank space up to and including the next line delimiter (escape literal
    newlines)

- Can write:
  - The alarm character using `\a`
  - The backspace character using `\b`
  - The tab character using `\t`
  - The newline character using `\n`
  - The carriage return using `\r`
  - Any unicode character using `\xXX+;`; that is, using an arbitrary
    number of hexadecimal digits delimited by an ending semi-colon
    (may be checked to be within the range of `0-10FFFF` by the implementation)

### Verbatim Strings

Verbatim strings start with a verbatim mark, `#]`. Then blankspace and a line
delimiter follow. Ignoring comment lines, subsequent verbatim marks have
the contents of the line after the mark appended to the multi-line string,
including the line delimiter. The last line delimiter is not
included in the final product.

Thus:

```
    #]
```

Represents the empty string.

This:

```
    #]
    #]
```

Represents the quoted string `"\n"`

This:

```
    #]
    ; This line is very interesting.
    #]12
    ; So is this one.
    #]34
```

Represents the string `"\n12\n34"`.

### Prose strings

Prose strings are the same as verbatim strings, except that unbroken substrings
of blankspace and whitespace characters that are are found between non-space
characters are replaced with a single space (` `).

This:

```
    #>
```

Still represents the empty string.

This:

```
    #>    As I walked
    #>    slowly by
    #>
```

Represents the quoted string `"    As I walked slowly by  \n"`

## Symbols

Symbols are named entities that are unique if their names are unique. They are
named by the characters in their name. They are semantically something like
enum values with names attached to them.

To quote the R7RS standard:


! $ % & * + - . / : < = > ? @ ^ _ ~
> ```
>
> Alternatively, an
> identifier can be represented by a se- quence of zero or more characters
> enclosed within vertical lines (`|`), analogous to string literals.

They can appear bare or quoted. If bare, the following
rules apply:

- As in the r7rs-small standard, symbols can have alphabetic ASCII characters, digits, or one of the following characters in it:
  `! $ % & * + - . / : < = > ? @ ^ _ ~`

- They can't start with a numerical character. They must start with
  one of the alphabetic ASCII characters or one of the symbols above.

- If they start with a period, a plus, or a minus, the second character
  can't be a number.

- Symbols cannot be a single period (`.`).

Quoted symbols are quoted using pipe characters `|` and escaping follows
the same rules as for strings.

### Lists

Lists begin with a `(` character and end with a `)` character. Elements are separated using whitespace.

## Small Example

```
(
    property-type park
    name "Goblin Valley"
    is-desert #t
    current-month-avg-high-temp 92.0
    yearly-visitors 500000
    jurisdiction state
    same-state-parks (
        "Arches"
        ; Very Nice and underrated.
        "Coral Pink Sand Dunes"
        "Grand Canyon"
    )

    ascii-face
    #].-------.
    #]| *   * |
    #]|   U   |
    #]| =   = |
    ; ^^ He needs a shave
    #]|  ---  |
    #]\-------/

    quote
    #>but
    #> why
)
```

Notes about the above example:
  - I can use symbols as either keys or values in the above plist
  - I don't have to escape the backslash in the `ascii-face` prefixed verbatim multi-line string
  - I can put comments just about anywhere, as long as they are on their own line.
  - Clean syntax

## What is a plist?

It is a list with an even number of values in it. This can be thought of as the
"dictionary" of DAN, where the values at the even-numbered spots are the "keys"
and the values at the odd-numbered spots are the "values. Keep in mind, it is
still a list. Multiple keys can be present in the plist, since it's just a
list. What you do with that is up to you. Since order matters, you might
implement a "first wins" parser, for example, where objects earlier in the plist
take precedence, or you might simply implement a multimap and encode its
contents with a plist.

There are several advantages to plists over unordered maps (in a serialization
format, at least):

  - They have order. Imagine being able to guarantee in JSON RPC that the
    `jsonrpc` version key is the first key to appear and that the
    `method` key was right after it, followed by the `params` key.
    All of a sudden, JSON RPC would be much easier to support in typed
    languages. Because incremental parsing is possible, polymorphism
    is made much easier without requiring reflection or untyped/type-erasure
    parsing.
  - They don't require their "keys" to be hashable or even orderable,
    since it is just a list. Keys can be anything.
  - They don't require their key/value pairs to appear only once. When
    converting a plist to a map internally, it is advisable that "first
    wins", but this is not a requirement.

## What Might APIs look like

### Typed languages

#### C

An incremental parser something like:

```
listin(&stream) // Enter a list under cursor
listout(&stream) // Exit current list, skipping all values while doing so
find(&stream, vset) // Move the cursor until it points to one of the values
                    // matching one in the given vset.
skip(&stream, num) // Skip <num> number of values.
expnul(&stream) // Expect a null and consume it if it's there.
expint(&stream) // Expect a int and consume it if it's there.
explng(&stream) // Expect a long and consume it if it's there.
expflt(&stream) // Expect a float and consume it if it's there.
expdbl(&stream) // Expect a double and consume it if it's there.
expstr(&stream) // Expect a string and consume it if it's there.
expsym(&stream) // Expect a symbol and consume it if it's there.
scan(&stream, scanstr, ...) // Scanf-like experience to extract values from the
                            // list.
```

I could add these arguments to the scan function:

`"_ [ %d %d %d ] [ %d %d ] ` or whatever
The brackets would signify that that everything inside is part of a struct
I pass the struct in as another argument between args for 1 2 and 3 but before
4
I pass in the struct, followed by its size
so

```
scan(stream,scan,&bufstruct.a,&bufstruct.b,&bufstruct.c,&bufstruct,sizeof(bufstruct_t), &buf2.x, &buf2, sizeof(buf2_t), &bufstruct_list, &buf2_list)
```
When the list runs out, the loop is jumped out of that is running the scan.

Scan-f could be super good here.

The above function could consume a list of structs using `memcpy()`.

#### Golang

Notably, in golang, nothing will change. DAN will be supported just as well
as YAML or JSON is because of the nature and shape of their
`Unmarshall` and `Marshall` functions. A similar API to those would work
beautifully.

### Untyped Languages

An experience similar to Golang's might be used in Python. The important thing
to note is that, while it is more difficult to parse in typed languages,
it may be advantageous. You have to at least know what you want to extract.
I submit you had to know that anyway. An API should be able to find a way
to make this easy to declare up-front.

```
# TODO
```

Notice, in languages that don't support symbols, they may be safely handled as
strings, or not, depending on the schema provided.

### Clojure, Common Lisp

These lisps don't share the same symbol definition as scheme, and they have
keywords which scheme does not have. The good news is, all valid keywords and
symbols in Clojure and Common Lisp are also scheme symbols. This was chosen
on purpose. Now all lisps are well supported, as well as any other language
using keywords. Just use a colon in the symbol name as normal.

## How is this better?

Since parsers have no way to distinguish between lists, sets, maps, or
vectors -- everything is just a list -- the program must know the schema
before hand and declare it. This is safer anyway. The right API will
make this easy, regardless of the language. The upside is that since
everything has order, typed languages have a much easier time since
incremental parsing is possible. Another huge upside, since everything
is a list, is that parsing and ingestion of enormous documents is well
supported.

## What about tables?

Consider the following example document of tables:

```
(
    tabledoc "v1.0"
    tables (
        columns 3
        data (
           email name     age
           "a@b" "billy"  15
           "x@y" "tay"    23
       )
   )
)
```

    
