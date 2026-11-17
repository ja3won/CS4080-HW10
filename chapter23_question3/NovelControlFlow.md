I would add a `between` clause to `for` loops. It would run only between two iterations of the loop. This would be useful for adding separators when building a string or printing a list.

The syntax would look like this:
```text
for (var item in items) {
  result = result + item;
} between {
  result = result + ", ";
}
```
The loop body would run first. If another item remains, the `between` block would run before the next iteration. It would not run after the final item. 

Without this feature, the same code usually needs an extra boolean variable or an index check:
```text
var first = true;
for (var item in items) {
  if (!first) result = result + ", ";
  result = result + item;
  first = false;
}
```
The `between` form is easier to read because the main loop only describes what happens to each item. The separator is placed in its own block, so there is no need for an off-by-one check or for knowing the position of the current item.

As for whether or not this feature would be useful, I think it would, but I would keep it limited to loops that can tell whether another iteration exists. For a normal condition-based `while` loop, I could use an ordinary statement at the end of the loop instead. This keeps the new syntax small while making common list and string-building code cleaner.