---
title: Implementation Details
layout: default
---

Implementation Details
======================

I found while implementing the parser and lexer along side all of the ISPC source
code was a little distracting. There's always the tempation to change things in order
to make them fit more nicely in a certain paradigm or preferences. I decided to develop
most of the code outside of the source tree and make it so that I can easily write a
method to convert from the new AST to the old AST.

### Lexer

The new lexer is generic to the character type, so that it will work with `char`, `char16_t`, `char32_t` and `wchar_t`.
When C++20 comes along, I also expect it will work with `char8_t` with little to no effort. I also parameterized the
size type used to describe lines, columns and indices because they are generally less than 16-bits each and can take
up a lot of memory in large files. To give an example, here's what's usually stored in each token.

```cpp
struct SourcePos final {
	std::size_t index;
	std::size_t line;
	std::size_t column;
};
struct SourceRange final {
	SourcePos start;
	SourcePos finish;
};
```

Which is probably going to be 48-bytes per token (not including other token fields.)
If the size type is parameterized, then the size is usually either going to be 12-bytes (`uint16_t`)
or 24-bytes (`uint32_t`) per token, which is much better.

The lexer ended up looking like this:

```cpp
template <typename Char, typename SizeType>
class Lexer final {
  public:
  	using TokenType = Token<Char, SizeType>;

  	static auto Lex(std::basic_string_view<Char> &sourceCode) -> std::vector<TokenType>;
};
```

I opted to make it a stateless lexer because none of the tokens are context sensitive.
This may not have been the case if I decided to either:

 - Use the lexer hack, where type specifiers are given to the lexer when handling `typedef`
 - Write a lexer for preprocessing (the `#include` directive requires special tokens for the header path)

### Parser

The parser also required parameterization of the token types.
I chose to do this for two reasons:

 - Keeping it de-coupled from the lexer
 - Making it easier to mock the input (if needed)
 - If I chose to implement a preprocessor, the tokens emitted
   from it would be heterogenous, since each token may or may
   not be part of a macro expansion. If the token types are parameterized,
   then this can easily be accommodated without having to re-write
   the parser.

These leads to the consequence that each component of the AST
is also parameterized by the token type. In order to keep the
code readable, I opted for putting everything into a `Parser`
class that is parameterized by the token type, so that I don't
have template declarations all over the place. Here's what is looks like:

```cpp
template <typename TokenIterator>
class Parser final {

    /* ... */

    class FunctionDecl final {
    	Type returnType;
    	FunctionDeclarator declarator;
	CompoundStmt body;
    };

    using std::variant<FunctionDecl, VariableDecl, StructDecl> TopLevelNode;

    class AST final {
        std::vector<TopLevelNode> nodes;
      public:
    };
  public:
    auto Parse(TokenIterator begin, TokenIterator end) -> ASTContext<TokenIterator>;
};
```

The interface to the AST is kept separate, so that
client code does not have to have templates all over
the place. Each node in the AST has a base class using
static polymorphism (it's not shown above). That
way, nobody has to visit a type like `ispc::Parser<const Token<const char *, std::size_t>>::FunctionDecl`
and instead they use an interface and heirarachy like this:

```cpp
template <typename Derived, typename TokenIt>
class FunctionDecl {
  public:

    template <typename DerivedStmtVisitor>
    auto VisitBody(StmtVisitor<DerivedStmtVisitor> &) const;

    template <typename DerivedTypeVisitor>
    auto VisitReturnType(TypeVisitor<DerivedTypeVisitor> &) const;

    auto GetName() const -> TokenIt;

    /* ... */
};

template <typename DerivedVisitor>
class TopLevelASTVisitor {
  public:
    template <typename DerivedFunc, typename TokenIt>
    auto Visit(const FunctionDecl<DerivedFunc, TokenIt> &func);

    /* ... */
};

template <typename TokenIt>
class ASTContext final {
  public:
    template <typename DerivedVisitor>
    auto VisitAST(TopLevelASTVisitor<DerivedVisitor> &) const;
};
```

An example usage:

```cpp
class CustomVisitor final : public TopLevelASTVisitor<CustomVisitor> {
  public:
    template <typename DerivedFunc, typename TokenIt>
    void VisitImpl(const FunctionDecl<DerivedFunc, TokenIt> &func) {
    	std::cout << "Function name: " << func.GetName() << std::endl;
    }
};
```

Yes there's still templates, but they're much shorter and the expectation
of what's required is shown clearly in `TopLevelASTVisitor`. There's also
the benefit of having return types in visitation methods. It would not be
as easy to have a non-void return type using virtual visitor classes. With
this method, there's a clean picture of what's an input and what's an output
in a visitor call:

```cpp
auto errorList = astContext.VisitAST(typeChecker);
```

Instead of something like:

```cpp
astContext.VisitAST(typeChecker);
auto errorList = typeChecker.GetErrorList();
```

*Which would also have several less-readable components to it in the implementation of
a derived visitor.*

### Downsides

As with most template-heavy code, the downside is going to be build times.
Also, anyone not familiar with CRTP is probably going to be confused when
looking at the AST heirarachy. And lastly, template errors are a horror.

I also wonder whether the type parameterization of the lexer is actually
worth it, since most compilers only support UTF-8 and the benefits of smaller
token types might be negligible.

### Conclusion

I'm happy with the design and I think it'll hold up for the rest of my
time working on this project. It's extremely nice having very loose coupling
between the lexer and parser and reassuring to know that supporting a preprocessing
phase won't invoke a rewrite. I'm really looking forward to seeing the implementation
to completion.
