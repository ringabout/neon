## magic functions

A function which is marked with the magic pragma is implemented by the compiler. It doesn't need an implementation. The name of the corresponding enum 
in the compiler is the result adding a 'm' before the magic name. For instance, the `ConStrStr` magic is corresponding to the `mConStrStr` enum. 

```nim
proc `&`*(x: string, y: string): string {.magic: "ConStrStr".}
```
