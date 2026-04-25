# Function specification for the Dino programming language

A function is part of a translation unit and can have an access modifier - if it is absent, the default access modifier (private) will be used
Function declaration occurs just as easily as in C, C++ or Java: ```<return type> <name>([function arguments]) { [function body] }```
Example:
```dino
int32 main() { return 0; } // Function will be private
```
By default, all functions declared in Dino are subject to mangling, but you can disable it – to do this, use the #[no_mangle] attribute before the declaration – this will prevent the symbol from being mangled in the symbol table, but it will also exclude language features such as method overloading and templates.

To declare a function/variable with linker-time resolution, use #[extern]
Example:
```dino
#[extern] int32 printf(const char* format, ...);

public void print_some_stuff() { printf(...); } // It works if the printf symbol will be available for the linker
```
