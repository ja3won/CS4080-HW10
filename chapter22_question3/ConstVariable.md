I chose the keyword `const` because it is short and is already used in languages like JavaScript and C++. A `const` variable can be assigned once when it is declared, but it cannot be changed afterward. I think this makes the purpose of the variable clear and helps catch accidental assignments.

First, I added `TOKEN_CONST` to `scanner.h`. In `scanner.c`, I added it with the other keywords:
```c
case 'c':
  if (scanner.current - scanner.start == 5) {
    if (memcmp(scanner.start, "class", 5) == 0) {
      return TOKEN_CLASS;
    }
    if (memcmp(scanner.start, "const", 5) == 0) {
      return TOKEN_CONST;
    }
  }
  break;
```

Then I added an `isConst` field to the `Local` structure:
```c
bool isConst;
```
When a `const` variable is declared, this field is set to `true`. I also keep the names of global constants so they can be checked during compilation.

The assignment check in `namedVariable()` is:
```c
if (canAssign && match(TOKEN_EQUAL)) {
  if (isConstLocal(current, &name) ||
      isConstGlobal(current, &name)) {
    error("Can't assign to a const variable.");
  }

  expression();
  emitBytes(setOp, (uint8_t)arg);
}
```

Since declaration required an initializer, I implemented:
```c
static void constDeclaration() {
  uint8_t global = parseVariable("Expect const name.", true);

  if (current->scopeDepth == 0) {
    current->constGlobals[current->constGlobalCount++] = parser.previous;
  }

  consume(TOKEN_EQUAL, "Expect '=' after const name.");
  expression();
  consume(TOKEN_SEMICOLON,
          "Expect ';' after const declaration.");
  defineVariable(global);
}
```
My code above connects `constDeclaration()` to `declaration()` and adds `TOKEN_CONST`.

A valid example how this works then is:
```text
const score = 100;
print score;
```

This produces a compile error:
```text
const score = 100;
score = 200;
```

The error message is:
```text
Can't assign to a const variable.
```