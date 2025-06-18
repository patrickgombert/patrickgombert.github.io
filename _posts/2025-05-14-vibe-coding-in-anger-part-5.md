---
layout: post
title: "Vibe Coding In Anger: Part 5"
date: 2025-05-14
---

[Previously](https://patrickgombert.com/2025/vibe-coding-in-anger-part-4/) on our quest to build out Vibe Coded Time Series Data Base (VCTSDB) we completed the _Ingestion Pipeline_. This included a pluggable JSON and CSV parser which can read incoming metric events. This time we'll start on _Phase 4: Query Foundation_ from the TODO, beginning with the _Query Parser_. This one I'm especially excited about because Cursor is going to form the syntax and semantics on its own! Looking at the spec it states:

> Query Processor
> - Implement a custom query engine that supports:
>   - SELECT queries with aggregation functions.
>   - Time-based queries for retrieving metrics over specific periods.
>   - Correlations across multiple metrics or series.

Let's see what it comes up with.

> Let's begin on the 'Query Parser' starting with the 'Lexical Analysis' sub section.

It carves out a new module path at _/query/parser_ and generates a _Lexer_. It pretty quickly gets to green. It wasn't able to handle compound keywords like 'ORDER BY' on its first attempt, but was able to fix it on the second iteration which is great to see. Looking at a test should give you a feel for the way it's handling tokens.

```rust
let input = "SELECT * FROM metrics WHERE value > 42.5";
let mut lexer = Lexer::new(input);
let tokens = lexer.tokenize().unwrap();

assert_eq!(tokens, vec![
    Token::Select,
    Token::Star,
    Token::From,
    Token::Identifier("metrics".to_string()),
    Token::Where,
    Token::Identifier("value".to_string()),
    Token::Gt,
    Token::NumberLiteral(42.5),
    Token::EOF,
]);
```

There's also some generic errors

```rust
pub enum LexerError {
    #[error("Unexpected character: {0}")]
    UnexpectedChar(char),
    #[error("Invalid number format: {0}")]
    InvalidNumber(String),
    #[error("Unterminated string literal")]
    UnterminatedString,
    #[error("Invalid identifier: {0}")]
    InvalidIdentifier(String),
}
```

Which we can again see in action in the tests

```rust
let input = "SELECT * FROM metrics WHERE value > @";
let mut lexer = Lexer::new(input);
let result = lexer.tokenize();

assert!(matches!(result, Err(LexerError::UnexpectedChar('@'))));
```

One thing you'll notice about _LexerError_ is that it does not contain the position of the error. There were two TODO items in this subsection and importantly one was _Syntax Error Positions_. The agent checked these off, but I would have expected _LexerError_ to take a row and column in each enum. In this case the agent seems to not have followed the instructions literally. This deviates from the behavior we've seen thus far where the AI has been following instructions extremely literally, almost to a fault. That's ok though, we can move on without this even though the behavior of the AI is interesting in this case.

Looking at the implementation there are no boolean literals, so as a sort of thought expirement it's fun to think about what it would take to add them. We could add a _BooleanLiteral_ enum for _Token_, which is a single line of code and is pretty straightforward.

```rust
pub enum Token {
  (...)
  BooleanLiteral(bool)
  (...)
}
```

Then, I think that _next_token_ can start by checking if the word "true" or "false" is up next and turn it into a _BoolanLiteral_.

```rust
let word = self.peek_word();
if word == "true" {
  self.consume_chars(4);
  return Ok(Some(Token::BooleanLiteral(true)));
}
if word == "false" {
  self.consume_chars(5);
  return Ok(Some(Token::BooleanLiteral(false)));
}
```

This feels easy to extend! I really like using the AI here because this is tedious code to write by hand and can be really finnicky if anything is even slightly amiss. This is a well studied problem and I'm sure there are so many lexers in the training data. It's nice to see it handle this so effortlessly.

Moving on, the next TODO section is for _AST Structure_.

> Let's move into the 'AST structure' from the TODO and build that into the query parser.

Cool, it very quickly has the structure of the AST! Again, it took only one iteration after the first test failure to get it to green. However, there's really no logic here, this is just AST modeling. Looking at the test again you can get a feel for what is being represented.

```rust
Query {
  select: vec![
    SelectExpr {
      function: FunctionCall {
        name: "avg".to_string(),
        args: vec![FunctionArg::Identifier("value".to_string())],
      },
      alias: Some("avg_value".to_string()),
    }
  ],
  from: "metrics".to_string(),
  time_range: Some(TimeRange::Last {
    duration: 3600_000_000_000, // 1 hour in nanoseconds
  }),
  filter: Some(FilterExpr::TagFilter(TagFilter {
    key: "region".to_string(),
    op: TagFilterOp::Eq,
    value: "us-west".to_string(),
  })),
  group_by: vec!["datacenter".to_string()],
  order_by: vec![("avg_value".to_string(), true)],
  limit: Some(10),
  offset: None,
};
```

It allows for complex filtering and so forth, I do like the AST representation here. Again, I assume this type of thing appeared often in the training data.

The big missing piece for me is that there is nothing tying it all together. This does feel like a theme so far in this project. We have all of the pieces, but the project is lacking the coherent singular combination of the pieces. Maybe that's coming up, but there is no interface that takes a string as an argument and outputs an AST. We really just need to go string -> lexer -> inflated AST. The rest of the _Query Parser_ TODOs do not explicitly spell the need for the clean singular interface out. This is tough because as a vibe coder there is no obvious place to poke and prod at what is taking shape. We can see the fragments, all of the bottoms up ideas, but the fragments never form a whole piece. You can't poke at any one thing, at least not yet.

> Go ahead and complete the rest of the TODO items in 'Query Parser'.

I swear I didn't plan this, but it got so close to what I was describing. We have a top level _Parser_ in the _mod.rs_. However, the interface looks like this:

```rust
impl<'a> Parser<'a> {
  pub fn new(tokens: &'a [Token]) -> Self {
    (...)
  }

  pub fn parse(&mut self) -> Result<Query, AstError> {
    (...)
  }

  (...)
}
```

You can't go string -> lexer -> inflated AST, but you can now do the second half! As long as you have the necessary collection of _Token_ ready you can parse those into a _Query_ (which remember is the AST)! Now, how to you get the tokens? Well, use the lexer, duh! Is the lexer part of the parser? No, it's not. You parse tokens, duh! Going from a string to tokens? That's uhh... not a parser thing. I don't know.

The other thing that was added (besides the new tests) was a whole validation component. There is a now a _QueryValidator_ which can be passed to a parser. It allows for various functions names to be registered and it will validate that you're not attempting to call some unknown function in your query. The built in functions it defines are:

```
- avg
- sum
- min
- max
- count
- rate
- stddev
- percentile
```

So that looks like a preview of things to come! It also introduced the concept of a _Schema_, which I'm not so sure about. It looks to accept tag names and known values. I'm not so sure what this is going for, those won't necessary be known ahead of time. The rest of the valdation is very sensical. It can walk the AST for various parts of the query and ensure that it behaves correctly. This is cool! Just to show off one example, there is a test for validating that when you select using the avg function that you use the correct arity of the avg function.

```rust
let query = Query {
  select: vec![
    SelectExpr {
      function: FunctionCall {
        name: "avg".to_string(),
        args: vec![
          FunctionArg::Identifier("value".to_string()),
          FunctionArg::Identifier("count".to_string()),
        ],
      },
      alias: None,
    }
  ],
  from: "metrics".to_string(),
  time_range: None,
  filter: None,
  group_by: vec![],
  order_by: vec![],
  limit: None,
  offset: None,
};

assert!(matches!(
  validator.validate(&query),
  Err(ValidationError::InvalidArgumentCount(_, _, _))
));
```

Great, now all of the TODOs for _Query Parser_ are checked off. I felt like this particular component really summed up my experience thus far with vibe coding. The AI knocked the known algorithms out without a problem, and was able to build them in a really nice way. However, the AI struggled with taking those pieces and putting them together. Without some sort of opinioned design guidance it seems to slam the pieces together until they're combined by force. [Next time](https://patrickgombert.com/2025/vibe-coding-in-anger-part-6/) we'll continue with the _Execution Framework_ which will allow us to take the _Query_ and execute it! The code up to this point can be found [here](https://github.com/patrickgombert/vctsdb/tree/afc0e941bb79ff6bd4534111c62ffd701ea3ac23).
