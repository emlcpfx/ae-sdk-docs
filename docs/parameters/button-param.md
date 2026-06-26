# Button Param

> 3 Q&As · source: AE plugin dev community Discord

### How do you implement a text editor in the Effect Control Window of an AE plugin?

Instead of trying to handle PF_Event_KEYDOWNs directly (which has limitations like missing Tab key and no param index), use a button that runs a script via AEGP_ExecuteScript(). The script string can display a text box dialog. Pass data to it by find+replacing in the script string before execution. Store the text in an arb data parameter. The script can return data back to AE.

*Tags: `aegp-execute-script`, `arb-data`, `button-param`, `custom-ui`, `text-input`*

---

### How can an AE plugin save a file (like CSV) when a button is clicked?

The AE SDK only deals with importing files into a project or rendering to custom file types. If you want to save arbitrary data to a file at the click of a button, just use standard C/C++ file I/O directly: fopen/fwrite/fclose. There's no need for any special SDK mechanism.

*Tags: `button-param`, `csv`, `export`, `file-io`, `fopen`*

---

### What is the best practice to spawn a window with text input from an effect's parameters?

Two separate concerns: (1) Getting user input: Use AEGP_ExecuteScript to launch a JavaScript prompt - it takes text input and returns whether the user hit OK or Cancel. For fancier UI, use OS-level windows (no third-party library needed). (2) Storing the data: Use an ARB parameter which allows undo/redo. You can have a button param launch the prompt and store the string in a hidden ARB, or combine both into one ARB with a custom UI that shows the current string AND is clickable.

*Tags: `aegp-execute-script`, `arb-param`, `button-param`, `custom-ui`, `dialog`, `text-input`*

---
