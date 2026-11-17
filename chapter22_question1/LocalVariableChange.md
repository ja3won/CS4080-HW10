Right now, `resolveLocal()` checks the local array from the end to the
beginning. This works, but it is a linear search, so looking up a variable can
take longer when a function has many locals.

I would use a hash table that maps a variable name directly to its stack slot.
The average lookup time would change from `O(n)` to `O(1)`. The basic idea would look like this:
```c
typedef struct {
  ObjString* name;
  int slot;
} LocalEntry;

LocalEntry localsByName[256];
```

Then, `resolveLocal()` could use the name's hash instead of checking every
local:

```c
static int resolveLocal(Compiler* compiler, Token* name) {
  int index = hashToken(name) % 256;

  while (localsByName[index].name != NULL) {
    LocalEntry* entry = &localsByName[index];

    if (entry->name->length == name->length &&
        memcmp(entry->name->chars, name->start, name->length) == 0) {
      return entry->slot;
    }

    index = (index + 1) % 256;
  }

  return -1;
}
```

When a local variable is declared, its name and slot would be inserted into
the table. Because Lox allows shadowing, the table would also need to save the
old slot for a name and restore it when the inner scope ends. Otherwise, an
outer variable could be lost after leaving a block.

```c
typedef struct {
  Token name;
  int slot;
  int previousSlot;
} Local;
```

This would make lookup faster, but it would also add code for hashing,
collisions, inserting names, and restoring shadowed variables.

I do not think the extra complexity is worth it for clox. Local variables are
only resolved while compiling, and most functions will not have many locals.
The current reverse scan is also simple and automatically handles shadowing.
I would keep the array unless profiling showed that variable lookup was a real
performance problem. A hash table would make sense for a much larger compiler
or a language with very large functions.
