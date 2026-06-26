# Dialog

> 1 Q&A · source: AE plugin dev community Discord

### What is the best practice to spawn a window with text input from an effect's parameters?

Two separate concerns: (1) Getting user input: Use AEGP_ExecuteScript to launch a JavaScript prompt - it takes text input and returns whether the user hit OK or Cancel. For fancier UI, use OS-level windows (no third-party library needed). (2) Storing the data: Use an ARB parameter which allows undo/redo. You can have a button param launch the prompt and store the string in a hidden ARB, or combine both into one ARB with a custom UI that shows the current string AND is clickable.

*Tags: `aegp-execute-script`, `arb-param`, `button-param`, `custom-ui`, `dialog`, `text-input`*

---
