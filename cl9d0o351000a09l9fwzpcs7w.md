# Practical Parsing in Rust with nom

If you're a programmer who's done some research on CS fundamentals, or
learned them in a college setting, you've probably heard of the term
"parser-combinator".

As a largely self-taught script kiddie, this term - heard in quiet whispers
and brushes with some tutorials every now and then - was scary to me for the
longest time. Parsers in general are associated with even more esoteric words
like "yacc", "lex", "tokenisation", "parse trees", "analytic grammar"... and too
many more.

There's really no reason for it to be this way, though. Parsing is a hard
problem, for sure, but it's also one of the most fundamental things programs
do. Mastering it, or at least its foundational concepts, is a huge step towards
unlocking higher-order levels of thinking about programming in general, because
they're so widely applicable.

In this article, I want to demonstrate the power of parser-combinators in Rust,
for a real use case that's not too simple and not too complex. Let me know in
the comments if I succeeded!

## Motivation

In my particular case, me and a pairing partner at the
[Recurse Center](https://www.recurse.com/) named [Jeff
Zhang](https://github.com/jeffzh4ng) wanted to make a simple [BitTorrent
protocol](https://www.bittorrent.org/beps/bep_0003.html) client. An important
part of this was parsing the torrent file to obtain information on the files to
download, trackers, and other metadata; in a format we'd never heard of called
Bencode. Bencode is a complex enough format that fumbling our way with manual
implementations or regexes didn't seem like a great idea. I also wanted to
overcome my deep-set fear of parsers without having to go all-in on reading the
[Dragon Book](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)
or something equally dense. So we found `nom`, which is a library
that enables anyone to make simple, type-safe parsers in Rust.

Bencode as defined in [BEP 0003](https://www.bittorrent.org/beps/bep_0003.html)
consists of two primitive types (numbers and strings) and two composite types
(lists and dictionaries). We can model this easily with an enum corresponding to
each of these types, with self-referential portions in the composite types:

```rust
#[derive(Debug, PartialEq, Eq, Clone)]
pub enum Bencode {
    Number(i64),
    ByteString(Vec<u8>),
    List(Vec<Bencode>),
    Dict(BTreeMap<Vec<u8>, Bencode>),
}
```

The basic form of a parser using `nom` is a simple function returning an
`IResult`, which is either an error or a tuple of the remaining input and the
successfully parsed output. For example, a simple number parser could behave
like so:

```rust
assert_eq!(parse_num(b"1"), Ok((b"", 1)));
assert_eq!(parse_num(b"1something"), Ok((b"something", 1)));
assert!(parse_num(b"nothing").is_err())
```

In our case, we need to define a signature for our overall parser like this:

```rust
pub fn parse_bencode(bencode_bytes: &[u8])
    -> IResult<&[u8], Bencode> {
    todo!()
}
```

In `nom`, parsers are created using combinations of other parsers. In our
case, we can create four separate parsers for each of the types and combine
them together easily, as we'll show later. `nom` also has its own primitive
combinators so we don't have to rewrite common logic ourselves. You do
need to import these from their respective modules, but for the sake of
simplicity I'll be leaving out the imports. You can use the search in the [`nom`
docs](https://docs.rs/nom/latest/nom) to find them.

## Numbers

First, we'll implement the simplest parser: numbers. Bencode numbers are
simply numbers wrapped in `i` and `e` on either end. For instance:

```rust
"i3e" // -> 3,
"i-3e" // -> -3
"i10e" // -> 10
"i2562e" // -> 2562
```

```rust
fn parse_number(bencode_bytes: &[u8])
    -> IResult<&[u8], i64> {
    // The `delimited` combinator runs a sequence
    // of three inner parsers and discards the
    // result of the first and third one if
    // they're successful.
    delimited(
        // `tag` is a parser that simply captures
        // the literal string it is passed.
        tag("i"),
        // The `map_res` combinator applies a
        // closure to the output of a parser,
        // converting any errors returned from the
        // closure into nom errors.
        map_res(
            // In this case, our parser is simply
            // capturing anything that isn't the
            // ending delimiter, which should be
            // the number itself.
            is_not("e"),
            // Since our bytes here aren't the raw
            // number but rather a string of the
            // number (for instance, not the byte
            // 0x04 itself but 0x34, which is the
            // ASCII for "4") we must parse it
            // into one.
            |bytes| String::from_utf8_lossy(bytes).parse::<i64>()
        ),
        tag("e")
    )(bencode_bytes)
}
```

The key thing to spot here is that since functions are first-class values in
Rust, all of `nom`'s combinators are in fact functions that return functions:
see how at the very end we're passing `bencode_bytes` to the function that
`delimited` returns. Many of them also *take* functions as arguments. This makes
them very straightforward to compose and the code much better at expressing
its underlying semantics.

## Strings

Bencode strings are a number followed by a colon, then characters equal to
that number. For instance:

```rust
"4:spam" // -> spam
"5:hello" // -> hello
"10:technology" // -> technology
"2:hello" // -> he
```

The string parser is only slightly more complex than the number parser, since
there isn't a delimiter for them as there is for numbers.

```rust
fn parse_string(bencode_bytes: &[u8])
    -> IResult<&[u8], Vec<u8>> {
    // First, we need to capture the number of
    // characters, which is a number ending in a
    // colon. The `terminated` combinator runs its
    // first argument, then its second one and
    // discards the second. Otherwise, this is
    // similar to the number parser above.
    let (remaining, num_characters) = terminated(
        map_res(
            digit1,
            |digits| String::from_utf8_lossy(digits).parse::<usize>()
        ),
        tag(":")
    )(bencode_bytes)?;

    // map is the same as map_res except the
    // closure is infallible instead of returning
    // a Result.
    map(
        // In this case, our parser is simply to
        // take a fixed number of characters.
        take(num_characters),
        |bytestring: &[u8]| bytestring.to_vec()
    )(remaining)
}
```

Notice how we use two separate steps here and return only the results of our
second step, since we don't care about the length of the string once we're done
parsing it.

## Lists

Lists are simply a series of Bencode values in succession, delimited
by `l` and `e`. For example:

```rust
"l 4:spam 5:hello i3e e" // -> [spam, hello, 3]
// they do nest, unfortunately
"l 2:he i3e l i4e i5e e e" // -> [he, 3, [4, 5]]
```

I've separated with spaces to make this clearer, though real Bencode would not
do that.

I initially thought this would be very difficult to implement correctly, since
lists can nest arbitrarily into other lists and dictionaries, but once again,
the simple nature of parsers, and the tools `nom` has to combine them, save us
in a major way. This is the real payoff for parser-combinators, folks!

```rust
fn parse_list(bencode_bytes: &[u8])
    -> IResult<&[u8], Vec<Bencode>> {
    delimited(
        tag("l"),
        // We already defined `parse_bencode`
        // above, which will return Bencode
        // values. So all we need to do here is
        // use the combinator `many0` to run
        // `parse_bencode` over and over on
        // increasingly smaller remaining inputs
        // until it fails! This way, we already
        // handle recursive lists and dictionaries,
        // once we fully implement `parse_bencode`!
        many0(parse_bencode),
        tag("e")
    )(bencode_bytes)
}
```

## Dictionaries

Dictionaries are delimited by `d` and `e`, and then contain a string
followed by a value, in succession. For instance:

```rust
"d 3:cow 3:moo 4:spam 4:eggs e" // -> {cow: moo, spam: eggs}
"d 4:spam l 1:a 1:b e e" // -> {spam: [a, b]}
// yeah, these also nest :(
"d 4:spam d 3:cow l 1:a 1:b e e e" // -> {spam: {cow: [a, b]}}
```

Again, I've separated everything with spaces for clarity only.

Even these beasts turn out to be astonishingly simple to implement, and more
importantly, to read, because of our composition superpowers!

```rust
fn parse_dictionary(bencode_bytes: &[u8])
    -> IResult<&[u8], BTreeMap<Vec<u8>, Bencode>> {
    map(
        delimited(
            tag("d"),
            many0(
                // The `pair` combinator allows us
                // to combine capturing the output
                // of two parsers in succession
                // into a tuple!
                pair(parse_string, parse_bencode)
            ),
            tag("e")
        ),
        // `pair` captures into tuples, and
        // `many0` collects them into `Vec`s. We
        // can simply collect these to a BTreeMap.
        |elements| elements.into_iter().collect()
    )(bencode_bytes)
}
```

## Putting It All Together

Finally, we need to fill in the signature of our `parse_bencode` public-facing
method by combining together all these individual parsers for our types. This
also works beautifully thanks to the `alt` combinator.

```rust
pub fn parse_bencode(bencode_bytes: &[u8])
    -> IResult<&[u8], Bencode> {
    // The `alt` combinator takes a tuple of parsers and 
    // keeps running them in succession until one of them
    // succeeds, or until all of them fail.
    alt((
        // We use `map` in all four cases to wrap
        // the result of the child parsers to the
        // specific enum variant they correspond to.
        map(parse_number, Bencode::Number),
        map(parse_string, Bencode::ByteString),
        map(parse_list, Bencode::List),
        map(parse_dictionary, Bencode::Dict),
    ))(bencode_bytes)
}
```

Just like that, we've got a easy-to-understand, fully type-safe parser for
Bencode, in a tad less than 100 lines of code. It's a little out of scope for
this article, but using
[`nom`'s VerboseError interface](https://docs.rs/nom/latest/nom/error/struct.VerboseError.html),
you can also very easily extend this parser to keep track of where exactly its
errors originate and display user-friendly error messages. Perhaps a topic
for a part 2 - let me know if you're interested.

You can find the full code 
[in the Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=38599ebd80b70497f555f141bfea9af3),
along with an extensive litany of tests to prove its correctness.

Thanks for reading this long! I hope you came to appreciate the coolness of
`nom` and parser-combinators in general the same way I did. If you have any
feedback at all, I'd appreciate leaving some in the comments. If you learned
something from this article, I'd appreciate that, too!

*Cover image courtesy of [XKCD](https://xkcd.com/1171/).*