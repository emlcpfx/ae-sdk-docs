# Q&A: aeio

**6 entries** tagged with `aeio`.

---

## How do I port an AEIO plugin with a user options dialog from Windows to macOS?

To port an AEIO_UserOptionsDialog from Windows to macOS, you need to use native macOS UI frameworks instead of Windows dialogs. For Carbon implementation, refer to the Adobe forums thread at https://forums.adobe.com/thread/559946?tstart=0 which contains code examples and links for Cocoa. Alternatively, you can use AEGP_ExecuteScript() to open a dialog via JavaScript, which works cross-platform on both Windows and macOS without requiring separate implementations.

*Tags: `aeio`, `ui`, `cross-platform`, `macos`, `windows`*

---

## What is a good approach for creating user option dialogs in AEIO plugins that work on both Windows and macOS?

You can use AEGP_ExecuteScript() to implement user option dialogs in JavaScript, which provides a cross-platform solution that works identically on Windows and macOS. This avoids the need to maintain separate platform-specific dialog code. For implementation details, see the example at https://forums.adobe.com/message/3625857#3625857 and search the Adobe After Effects scripting community forum at https://forums.adobe.com/community/aftereffects_general_discussion/ae_scripting for additional script examples with checkboxes and dropdown lists.

*Tags: `aeio`, `scripting`, `ui`, `cross-platform`, `aegp`*

---

## Can I set a higher priority for my custom AEIO to override Adobe's built-in handler for a format like FLV?

No, the After Effects SDK does not support overriding the priority of built-in AEIOs. After Effects will ignore custom AEIOs that associate to formats it already handles natively. The recommended workaround is to ask users to change the file extension to a different one (e.g., .flv to .custom_flv), and optionally create an import menu entry to automate this extension change for better user experience.

*Tags: `aeio`, `sdk`, `aegp`, `deployment`*

---

## How can I access and modify pixels in DrawSparseFrame of an AEIO without using Iterate8Suite?

You can create a PF_InData struct and fill it with whatever data you have available, which should allow you to use the iterate suites. Alternatively, you can iterate through the buffer's pixels directly using nested loops to access individual pixels via sampleIntegral32. However, be careful with coordinate ordering—ensure you're passing coordinates in the correct order (x,y) rather than swapped (y,x), and account for thumbnail resolution differences which can cause crashes or unexpected behavior.

```cpp
for (A_long i = 0; i < 1000; i++)
for (A_long j = 0; j < 1000; j++){
  PF_Pixel *pixel = sampleIntegral32(wP, i, j);
  pixel->alpha = PF_MAX_CHAN8;
  pixel->red = PF_MAX_CHAN8;
  pixel->green = PF_MAX_CHAN8;
  pixel->blue = PF_MAX_CHAN8;
}
```

*Tags: `aeio`, `aegp`, `memory`, `output-rect`, `debugging`*

---

## How do you open and read files in an AEIO plugin using the SDK API?

The After Effects SDK does not provide built-in file I/O functions. Instead, you use standard C library functions like fopen, fread, and fclose to open and read files. When you receive a file path as A_UTF16Char in functions like VerifyFileImportable(), you can use it directly with these standard functions.

*Tags: `aeio`, `import`, `sdk`, `aegp`*

---

## How do you convert A_UTF16Char strings to 8-bit char strings in After Effects plugins?

You can convert A_UTF16Char (UTF-16) to char (8-bit) by iterating through the UTF-16 string and copying each character to a buffer. Create a char buffer large enough to hold the converted string, then use a while loop to copy characters from the UTF-16 string pointer to the char buffer pointer, incrementing both pointers. Ensure the buffer is null-terminated and large enough to avoid overflow.

```cpp
char buffer8[1024] = {'\0'};
char *char8 = buffer8;
A_UTF16Char *char16 = thatUTF16String;
while(*char16) {
  *char8++ = *char16++;
}
*char8 = NULL;
```

*Tags: `aeio`, `sdk`, `cross-platform`*

---
