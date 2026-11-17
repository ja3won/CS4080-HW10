I changed the local array to grow when it becomes full and used two bytes instead of one for the stack slot.

In the `Compiler` structure, I changed the fixed array to:
```c
Local* locals;
int localCount;
int localCapacity;
```

In `initCompiler()`, I gave it an initial capacity of 256:
```c
compiler->localCount = 0;
compiler->localCapacity = UINT8_COUNT;
compiler->locals = malloc(sizeof(Local) * compiler->localCapacity);
```

Then I made the array double in size when it is full:
```c
if (current->localCount == current->localCapacity) {
  current->localCapacity *= 2;
  current->locals = realloc(
      current->locals,
      sizeof(Local) * current->localCapacity);
}
```

I also removed the old error that stopped the compiler at 256 locals, as well as add a helper that writes a local-variable instruction with a two-byte slot:
```c
static void emitLocal(uint8_t instruction, int slot) {
  emitByte(instruction);
  emitByte((slot >> 8) & 0xff);
  emitByte(slot & 0xff);
}
```

Local variable access now uses this helper:
```c
emitLocal(OP_GET_LOCAL, arg);
emitLocal(OP_SET_LOCAL, arg);
```

Finally, the VM reads the slot as a short value:
```c
case OP_GET_LOCAL: {
  uint16_t slot = READ_SHORT();
  push(frame->slots[slot]);
  break;
}

case OP_SET_LOCAL: {
  uint16_t slot = READ_SHORT();
  frame->slots[slot] = peek(0);
  break;
}
```