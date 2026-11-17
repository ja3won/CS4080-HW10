I decided that every case should stop automatically after its statements finish, so I did not add `break` support.

I added these tokens to `TokenType`:
```c
TOKEN_CASE, TOKEN_COLON, TOKEN_DEFAULT, TOKEN_SWITCH,
```
The scanner returns `TOKEN_COLON` for `:` and recognizes the three new keywords.

I added `TOKEN_SWITCH` to `statement()`:
```c
} else if (match(TOKEN_SWITCH)) {
  switchStatement();
```
The switch value stays on the stack while each case is checked. 

I then added `OP_DUP` so the value can be copied before comparing it with a case value:
```c
case OP_DUP:
  push(peek(0));
  break;
```

Finally, I added this compiler function:
```c
#define MAX_CASES 256

static void switchStatement() {
  consume(TOKEN_LEFT_PAREN, "Expect '(' after 'switch'.");
  expression();
  consume(TOKEN_RIGHT_PAREN, "Expect ')' after value.");
  consume(TOKEN_LEFT_BRACE, "Expect '{' before cases.");

  int endJumps[MAX_CASES];
  int endCount = 0;
  int nextCase = -1;
  bool hasDefault = false;

  while (!check(TOKEN_RIGHT_BRACE) && !check(TOKEN_EOF)) {
    if (match(TOKEN_CASE)) {
      if (hasDefault) {
        error("A case cannot come after default.");
      }

      if (nextCase != -1) {
        endJumps[endCount++] = emitJump(OP_JUMP);
        patchJump(nextCase);
        emitByte(OP_POP);
      }

      emitByte(OP_DUP);
      expression();
      consume(TOKEN_COLON, "Expect ':' after case value.");
      emitByte(OP_EQUAL);
      nextCase = emitJump(OP_JUMP_IF_FALSE);
      emitByte(OP_POP);

      while (!check(TOKEN_CASE) &&!check(TOKEN_DEFAULT) &&!check(TOKEN_RIGHT_BRACE) &&!check(TOKEN_EOF)) {
        statement();
      }
    } else if (match(TOKEN_DEFAULT)) {
      hasDefault = true;

      if (nextCase != -1) {
        patchJump(nextCase);
        emitByte(OP_POP);
        nextCase = -1;
      }

      consume(TOKEN_COLON, "Expect ':' after default.");
      while (!check(TOKEN_RIGHT_BRACE) && !check(TOKEN_EOF)) {
        statement();
      }
    } else {
      error("Expect 'case' or 'default'.");
      advance();
    }
  }

  if (nextCase != -1) {
    patchJump(nextCase);
    emitByte(OP_POP);
  }

  consume(TOKEN_RIGHT_BRACE, "Expect '}' after switch.");
  emitByte(OP_POP); // Remove the original switch value.

  for (int i = 0; i < endCount; i++) {
    patchJump(endJumps[i]);
  }
}
```
The compiler keeps the switch value on the stack and compares a duplicate of it with each case value. If the comparison is false, it jumps to the next case. If it is true, the case statements run and an unconditional jump skips the remaining cases. At the end, the saved jumps are patched to the first instruction after the switch. The final OP_POP removes the original switch value from the stack. The stored jumps send a matching case to the end of the switch.