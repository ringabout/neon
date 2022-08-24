## magic functions

A function which is marked with the magic pragma is implemented by the compiler. It doesn't need an implementation. The name of the corresponding enum 
in the compiler is the result adding a 'm' before the magic name. For instance, the `ConStrStr` magic is corresponding to the `mConStrStr` enum. 

```nim
proc `&`*(x: string, y: string): string {.magic: "ConStrStr".}
```

## Is it efficent to use `$` function to concatenate multiple strings

It is a magic proc used to concatenate strings and chars. The Nim compiler does some optimizations to make it perform as good as the in-place version.

There are four overloads for `$` function in the system module.

```nim
proc `&`*(x: string, y: char): string {.magic: "ConStrStr", noSideEffect.}
proc `&`*(x, y: char): string {.magic: "ConStrStr", noSideEffect.}
proc `&`*(x, y: string): string {.magic: "ConStrStr", noSideEffect.}
proc `&`*(x: char, y: string): string {.magic: "ConStrStr", noSideEffect.}
```

All of them have the same magic `mConStrStr`, which is needed in the optimization phases. 

```nim
s = "Hello " & name & ", how do you feel" & 'z'
```

Here is the ast of the right-side expression:

```nim
StmtList
  Infix
    Ident "&"
    Infix
      Ident "&"
      Infix
        Ident "&"
        StrLit "Hello "
        Ident "name"
      StrLit ", how do you feel"
    CharLit 122
```

There are lots of `&` call in the ast, which causes unnecessary overheads. They can be eliminated by lifting the leaf nodes like strings and chars to the same level as the first `&` ident. 

Here is the pseudo code:
```nim
proc traverseConcatTree(result: PNode, x: PNode) =
  if x.hasMagic(mConStrStr):
    for i in 1 ..< x.len: # x[0] is the ident "&"
      result.add traverseConcatTree(x[i])
  else:
    result.add copyTree(x)
```

After the transforming, the ast tree becomes now flatten:

```nim
StmtList
  Infix
    Ident "&"
    StrLit "Hello "
    Ident "name"
    StrLit ", how do you feel"
    CharLit 122
```

There are some adjacent constant expressions which can be merged into a single expression

```nim
StmtList
  Infix
    Ident "&"
    StrLit "Hello "
    Ident "name"
    StrLit ", how do you feelz"
```

So the expression becomes `&`("Hello ", name, ", how do you feelz"). If all of the parameters are constant expressions, the Infix call can be eliminated.

Above should concludes all the transforms in the optimization phases. Now comes the codegen phase. First, the compiler needs to preallocate the memory used by strings. Then `appendString` adds the value of string literals and identifiers to `tmp0`. Finally, it assigns the value of `tmp0`
to `s`.


```c
string tmp0;
string s;
string name;
...
tmp0 = rawNewString(6 + 18 + name.len + 1);
// we cannot generate s = rawNewString(...) here, because
// ``s`` may be used on the right side of the expression
appendString(tmp0, strlit_1);
appendString(tmp0, name);
appendString(tmp0, strlit_2);
asgn(s, tmp0);
```
