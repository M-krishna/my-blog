+++
title = "Writing a simple Lexer"
date = 2024-04-08
template = "post.html"
+++

In this blog post, I’ll be going over how to write a simple *Lexer* that parses and generates *Tokens* for few characters. We’ll also add few rules to the parsing logic on how to parse certain characters.

The code can be found on [Github](https://github.com/M-krishna/lexer).

Let’s first define what we are trying to *parse*. We’ll parse the following characters.

- `{` → open curly bracket
- `}` → closed curly bracket
- `(` → open parenthesis
- `)` → closed parenthesis
- `*` → star
- `+` → plus
- `,` → comma
- `.` → dot
- `-` → minus

Our goal is to parse these characters and generate token out of these. Generally we call these characters *lexemes*. For example, `{` is represented as `LEFT_BRACE` and you can name it whatever you want as long it represents the correct token. Let’s breakdown our problem. Currently, our goal is to parse only one character. Let’s choose `{` for our initial implementation. 

We should parse the `{` and generate a token like `LEFT_BRACE {` in a string format.

## Initial implementation

---

Let’s start by defining our token type because eventually we’ll be generating token once we parse a character.

```python
from enum import Enum

class TokenType(Enum):
	LEFT_BRACE = 1
```

In the above code snippet, I have created a class named `TokenType` which is an *enum*, which holds all the required *tokens* for our parser(lexer). Currently we have only one token which is the left curly brace (`{`), we’ll add more once we setup our parser.

Now let’s create a class for our Lexer which takes in a *source*, which are the characters that we want to parse.

```python
class Lexer:
	def __init__(self, source):
		self.source = source
		
if __name__ == "__main__":
	characters: str = "{" # for now only one
	lexer = Lexer(characters)
```

In the above code snippet, we have defined a class named `Lexer` which will hold everything that we need to achieve our goal. Our *Lexer* takes in a *source* which is nothing but a *string of characters,* that contains the character that we need to parse.

Under the *if* condition, we have a variable named `characters` which will hold the things we need to parse. Currently it contains `{` , we’ll add more later. Right after that, we have initialized our `Lexer` class by passing in the *characters* as *source*.

The next step is to “consume” the character and for now, let’s print it.

## Consume characters

---

Let’s add a method that consume characters one by one. For now, we implement this method that’ll print the character it consumes.

```python
class Lexer:
	def __init__(self, source) -> None:
		self.source = source
		self.start_position = 0
		self.current_position = 0
	
	def consume_characters(self) -> None:
		while (not self.isAtEnd()):
			self.start_position = self.current_position
			self._consume_character()
			
	def _consume_character(self) -> None:
		current_character: str = self.eat_and_move_on()
		print(current_character)
		
	def eat_and_move_on(self):
		self.current_position = self.current_position + 1
		return self.source[self.current_position - 1]
		
	def isAtEnd(self) -> bool:
		return self.current_position >= len(self.source)
```

There are a few things to unpack from the above code snippet. Let’s look at one by one. We have two similar methods `consume_characters` and `_consume_character`. And we have a couple of helper methods `isAtEnd` and `eat_and_move_on` which we are using to make a clear abstraction.

`consume_characters` is a top level method. The goal of this method is to consume all the characters that we pass one by one until it reaches the EOF(End of file). This method uses the `isAtEnd` helper method to constantly check whether it reached the end of the file or not.

If it doesn’t reach the end of the file, we are doing two things

- Setting the value of `start_position` to the current value of `current_position` . We will see why we are doing this.
- Second we are calling the `_consume_character` method. The goal of the method is to consume the one character at a time and process it. For now we are just printing it.

The `isAtEnd` method is pretty self explanatory, it checks whether the *current_position* reaches the end of the file. It does that by comparing the value of the *current_position* and the length of the *source* (which is the string of characters that we want to parse).

As the name suggests, the `eat_and_move_on` method “eats” the current character that the `current_position` points to and then increment the value of the `current_position` so that on the next iteration, it can “eat” the next character. Here “eat” is just a metaphor for *consume*.

We’ll update our initialization code and test our current functionality.

```python
if __name__ == "__main__":
	characters: str = "{" # for now only one
	lexer = Lexer(characters)
	lexer.consume_characters()
```

Go ahead and run this code and see what’s happening. You should see `{` on the console. 

We can add more characters to our `characters` variable and our program will still consume and eat all the characters. Try to run the program by modifying the value of the `characters` variable to this: `{}{}{}{}` , you should see all the characters print in a new line. Now that we’ve written our logic to consume character one by one, we can start generating token from it. But first of all what are *tokens*?

## Tokens

---

Tokens are the smallest unit of meaningful text in programming languages, such as keywords, identifiers, operators, literals and punctuation marks. Tokens serve as the basic building blocks of the language’s syntax and used to represent individual elements within code.

As you have seen from the code snippet at the very beginning of this blog post, we have a *token type* called `LEFT_BRACE` which is a meaningful representation of `{`. These are what we call as *tokens*. Let’s look at a few more single character token types.

```python
from enum import Enum

class TokenType(Enum):
  LEFT_BRACE        = 1
  RIGHT_BRACE       = 2
  LEFT_PAREN        = 3
  RIGHT_PAREN       = 4
  STAR              = 5
  PLUS              = 6
  MINUS             = 7
  COMMA             = 8
  DOT               = 9
```

In the above code snippet, we’ve added few more meaningful representation of the characters. You’ll see how tokens come together once we start generating it buy parsing the characters. Let’s generate tokens now, shall we?

## Generating Tokens

---

We’ll start from where we left off. In our `Lexer` class, specifically in the `consume_character` method, we were just printing the characters. Now we’ll use the characters to generate tokens. Let’s first generate a token for the `{` (LEFT_BRACE) character.

But how do we represent *tokens*? Currently we have represented them as text like `LEFT_BRACE` , `RIGHT_BRACE` and so on. But we have to represent them in some way. We’ll create a `Token` class which for now only holds the *character* and the *textual representation* of that character.

```python
class Token:
	def __init__(self, tt: TokenType, value: chr) -> None:
		self.token_type = tt
		self.value      = value
	
	def __repr__(self) -> str:
    return f"{self.token_type} {self.value}"
```

In the above code snippet, we’ve created a `Token` class, which holds the *token type* and the *value*(character) of the token. Now that we can represent *tokens* in some form, let’s start generating them.

```python
class Lexer:
	def __init__(self, ...) -> None:
		...
		...
		self.tokens: list = [] # we store all the tokens in a list
	
	def _consume_character(self) -> None:
		current_character: str = self.eat_and_move_on()
		if current_character == '{':
			self.generate_token(TokenType.LEFT_BRACE.name)
		
	def generate_token(self, tt: TokenType) -> None
		text: str = self.source[self.start_position:self.current_position]
		token: Token = Token(tt, text)
		self.tokens.append(token)
```

Let’s unpack the above code. 

I’ve added a new method named `generate_token` which takes in the `TokenType` based on the character which we are currently parsing and it gets the *text*(character) from the *source* using the *start position* and the *current position*. Finally it creates a new *Token* object and pushes it into our *tokens* list.

In the `consume_character` , I’ve added an *if condition* which checks for the `{` character and if it’s *true* then, we use the `generate_token` method by passing in the *token type* and generate the token.

Go ahead and execute this code and try to print the `tokens` list and see what is in that list for yourself.

Currently, we are generating token only for the `{` character. We will now add all the characters that our *lexer* needs to parse and generate *token.* If you look at the above `TokenType` class we’ve added all the lexemes that we are trying to parse. Now the only thing that we have to do is generate tokens based on the characters. Let’s do that.

```python
def _consume_character(self) -> None:
		current_character: str = self.eat_and_move_on()
		if current_character == '{':
			self.generate_token(TokenType.LEFT_BRACE.name)
		if current_character == '}':
			self.generate_token(TokenType.RIGHT_BRACE.name)
		if current_character == '(':
			self.generate_token(TokenType.LEFT_PAREN.name)
		if current_character == ')':
			self.generate_token(TokenType.RIGHT_PAREN.name)
		if current_character == '*':
			self.generate_token(TokenType.STAR.name)
		if current_character == '+':
			self.generate_token(TokenType.PLUS.name)
		if current_character == '-':
			self.generate_token(TokenType.MINUS.name)
		if current_character == ',':
			self.generate_token(TokenType.COMMA.name)
		if current_character == '.':
			self.generate_token(TokenType.DOT.name)
```

We’ve added few more *if conditions* to check for other types of characters and generate tokens for them. Don’t worry about the beauty of the code, since Python doesn’t support switch cases we have used *if conditions*, we can also avoid using *if statements* and do this much more cleaner. But to keep things simple we are gonna stay with this for now.

## Ignoring whitespaces

---

Let’s add the ability to ignore (discard) whitespaces to our Lexer. We will take three things into consideration.

1. Simple space (’ ‘)
2. Carriage return (’\r’)
3. New line (’\n’)

If we come across these characters during our source language parsing we should ignore them. We’ll add this functionality to our `_consume_character` function. But first we’ll define a function to check whitespace characters.

```python
def isWhitespace(self, character: str) -> bool:
	SPACE = ' '
	NEWLINE = '\n'
	CARRIAGE_RETURN = '\r'
	return character in [SPACE, NEWLINE, CARRIAGE_RETURN]
	
def _consume_character(self) -> None:
	current_character: str = self.eat_and_move_on()
	...
	
	if self.isWhitespace(current_character):
		return
```

We’ve added a method named `isWhitespace` which takes in a character we are currently parsing and checks whether it is a *white space* or not. In the `_consume_character` method we are using that function by passing in the character. If it returns *true* we are returning from the function, which means we are doing nothing (discarding). That’s it, we’ve successfully added our functionality to ignore whitespaces. Go ahead and play around with the code and see what happens.

## Two character lexemes

---

So far we’ve seen only one character lexemes like {, }, (, and so on. We’ll look at how to handle ***relational operators***. Things like:

- `!=`
- `==`
- `<=`
- `>=`

As you can see, there will be two cases that we have to deal with when parsing these characters and generating tokens. Let’s look at an example for `!=`

### Parsing `!=`

---

The bang operator (`!`) can be used as a standalone operator, which stands for *negation* or *not.* So when parsing `!` we have to check whether the next character is `=` or something else. If it is `=` we have to consider both of the character as a single token, which represents `!=` (*not equal to*).

If it is only `!` we have to consider it as *not* (negation) operator. Let’s add the functionality to achieve this.

```python
class TokenType(Enum):
	...
	...
	BANG = 10
	BANG_EQUAL = 11
	
class Lexer:
	...
	def match_next_character(self, expected_character: str) -> bool:
		if self.isAtEnd(): return False
		if (self.source[self.current_position] != expected_character): return False
		
		self.current_position = self.current_position + 1
		return True
		
	def _consume_character(self) -> None:
		current_character: str = self.eat_and_move_on()
		...
		...
		if current_character == '!':
			if self.match_next_character('='):
				self.generate_and_add_token(TokenType.BANG_EQUAL.name)
			self.generate_and_add_token(TokenType.BANG.name)
			
characters: str = "!=="
lexer.consume_characters()
```

Let’s unpack. We’ve added a new method named `match_next_character` which takes in the *expected_character* which we are looking for and returns *true* if it matches else *false*.

In the `_consume_character` method we’ve added a condition to check if the current character that we are parsing is `!` , if its *true*, we are checking whether the *next character* is equal to `=`. If its *true*, we are generating a token with the `BANG_EQUAL` type, else `BANG` type. We are using the keyword `BANG` because, this operator (`!`) is called as *bang operator*.

If you notice, in the `match_next_character` method, we are *incrementing the value of `current_position`*. Why is that? It’s because, we’ve confirmed that the next character matches the `=` character, so we have to treat `!=` as a single operator. For this particular reason, we are incrementing the `current_position` value.

If we didn’t do that, our Lexer will parse the `=` again even after we consider the `!=` as a single operator. Go ahead and comment out that line and run the code to see the difference.

Voila 🎉 we’ve successfully parsed the `!=` operator. In a similar way, we can able to parse the remaining three *relational operators*.

```python
if current_character == '=':
    self.generate_and_add_token(
        TokenType.EQUAL_EQUAL.name if self.match_next_character('=') else TokenType.EQUAL.name
    )
if current_character == '<':
    self.generate_and_add_token(
        TokenType.LESS_THAN_EQUAL.name if self.match_next_character('=') else TokenType.LESS_THAN.name
    )
if current_character == '>':
    self.generate_and_add_token(
        TokenType.GREATER_THAN_EQUAL.name if self.match_next_character('=') else TokenType.GREATER_THAN.name
    )
```

## Keyword Parsing

---

Let’s examine some keywords commonly found in most programming languages. To keep things simpler, let’s take some keywords from Python.

- `and`
- `or`
- `not`
- `if`
- `elif`
- `else`
- `while`
- `for`
- `break`
- `continue`
- `def`
- `return`
- `class`
- `print`

Let’s breakdown the approach for parsing the keywords.

Whenever we come across a character, we will first check whether it is an alphabet or not, why? because all the *keywords* above only contain alphabets. Once we confirm that it’s an alphabet, we’ll *peek* through the following characters and check each and every character is an alphabet or not. Once we reach a *non-alphabet* character, we will stop there and take the *text* we parsed. Then we will check the parsed word with our keywords list, if we find a match, we will be generating the appropriate token.

```python
class TokenType(Enum):
	...
	...
  # Keywords
  AND = 18
  OR = 19
  NOT = 20
  IF = 21
  ELIF = 22
  ELSE = 23
  WHILE = 24
  FOR = 25
  BREAK = 26
  CONTINUE = 27
  DEF = 28
  RETURN = 29
  CLASS = 30
  PRINT = 31
```

```python

class Lexer:
   # Keywords object which we will use to match against the parsed text
    KEYWORDS = {
    "and": TokenType.AND.name,
    "or": TokenType.OR.name,
    "not": TokenType.NOT.name,
    "if": TokenType.IF.name,
    "elif": TokenType.ELIF.name,
    "else": TokenType.ELSE.name,
    "while": TokenType.WHILE.name,
    "for": TokenType.FOR.name,
    "break": TokenType.BREAK.name,
    "continue": TokenType.CONTINUE.name,
    "def": TokenType.DEF.name,
    "return": TokenType.RETURN.name,
    "class": TokenType.CLASS.name,
    "print": TokenType.PRINT.name
	}
	...
	...
	if current_character.isalpha():
	  while (self.peek().isalpha()):
	      self.eat_and_move_on()
	  keyword: str = self.source[self.start_position:self.current_position]
	  self.generate_and_add_token(
	      Lexer.KEYWORDS.get(keyword, None)
	  )
	  
    def peek(self) -> chr:
        if self.isAtEnd(): return '\0'
        return self.source[self.current_position]
```

There is a lot to unpack. Let’s look at it one by one.

We have added all the keywords that we needed under `TokenType` and we have also created a `KEYWORDS` object, which holds all the keywords with their names as *keys* and their respective token types as their *values*.

We have also added a new method named `peek` . The job of this method is to return the character in the *current_position*. We will use this method to read (parse) keywords.

Finally we’ve added an *if condition* which checks whether the character we are parsing is an *alphabet* or not. If it is an alphabet we will continue reading the following characters using the `peek` method until we reach any *non-alphabetic* character. Once we have reached the end of the non-alphabetic character, we will use the value of `start_position` and `current_position` to get the parsed text or character. Since our `KEYWORDS` object holds all the necessary keyword, we’ll match it against that object. If we find a match then generate the appropriate token, else do nothing.

We have reached the end of this blog post. We have created a Lexer that parses quite a lot of content. On that note, try adding support for parsing *strings*, *numbers* and *identifiers.* To take it one step further modify the code to parse a subset of Javascript.