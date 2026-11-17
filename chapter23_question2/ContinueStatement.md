I added `TOKEN_CONTINUE` and made the scanner recognize `continue`. In,`statement()`, add:
```c
} else if (match(TOKEN_CONTINUE)) {
  continueStatement();
```

I used two compiler variables to keep track of the loop that is currently being compiled:
```c
int continueLocation = -1;
int loopDepth = 0;
```
`continueLocation` stores where the jump should go, while `loopDepth` helps clean up variables created inside the loop.

When compiling a loop, I saved the old values first. This was needed so an inner loop does not change where an outer loop continues:
```c
int previousLocation = continueLocation;
int previousDepth = loopDepth;

continueLocation = currentChunk()->count;
loopDepth = current->scopeDepth;
```

Then, for a `for` loop with an increment expression, I changed the location after compiling that expression:
```c
int incrementLocation = currentChunk()->count;
expression();
emitByte(OP_POP);
consume(TOKEN_RIGHT_PAREN, "Expect ')' after for clauses.");

emitLoop(continueLocation);
continueLocation = incrementLocation;
patchJump(bodyJump);
```

After compiling the loop body, I jump back and restore the outer loop's values:
```c
statement();
emitLoop(continueLocation);

continueLocation = previousLocation;
loopDepth = previousDepth;
```
The `while` loop uses the same variables, but its continue location is the beginning of its condition.

The `continue` compiler function checks that a loop is active and removes any locals declared inside the loop before jumping. This prevents those locals from being left on the stack:
```c
static void continueStatement() {
  if (continueLocation == -1) {
    error("Can't use 'continue' outside of a loop.");
  }

  consume(TOKEN_SEMICOLON,
          "Expect ';' after 'continue'.");

  for (int i = current->localCount - 1;
       i >= 0 && current->locals[i].depth > loopDepth;
       i--) {
    emitByte(OP_POP);
  }

  emitLoop(continueLocation);
}
```