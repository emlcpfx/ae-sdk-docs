# Q&A: error-handling

**10 entries** tagged with `error-handling`.

---

## What is the best practice when a plugin runs out of GPU memory during rendering?

Returning PF_Err_OUT_OF_MEMORY may cause AE to show a black frame without warning the user. Options: (1) Use a C++ text library to render 'Out of GPU memory' text directly onto the frame. (2) Set a static global boolean (protected by mutex) in the render thread, then display the warning in another thread like PF_UpdateParamUI. Avoid using out_data warning messages during renders as they would fail render queue operations and prevent MFR from retrying with fewer threads.

*Contributors: [**Jonah (Baskl.ai/Haligonian)**](../contributors/jonah-baskl-ai-haligonian/), [**tlafo**](../contributors/tlafo/) · Source: adobe-plugin-devs · 2023-11-30 · Tags: `gpu`, `out-of-memory`, `error-handling`, `mfr`, `render-queue`*

---

## What are best practices for error handling and memory safety in AE SDK plugins?

Recommendations: (1) Wrap AE handles in C++ RAII classes with proper destructors, use smart pointers. (2) Use scope guards (like the pattern from Dr. Dobb's) for cleanup on any exit path. (3) Use std::expected (C++23) or Result types to chain operations safely. (4) Implement an allocation manager that clears allocations when exiting entry functions, since AE handle pointers are not valid between calls. (5) In debug builds, add a cache validator to detect extra memory still allocated at function exit points. (6) Use custom breakpoint assertions that trigger breakpoints when debugger is attached instead of crashing.

*Contributors: [**Alex Bizeau (maxon)**](../contributors/alex-bizeau-maxon/), [**fad**](../contributors/fad/), [**James Whiffin**](../contributors/james-whiffin/) · Source: adobe-plugin-devs · 2025-07-27 · Tags: `error-handling`, `memory-safety`, `raii`, `scope-guard`, `smart-pointers`, `best-practice`*

---

## What are best practices for AE SDK string conversions between A_char, A_UTF16Char, and std::string?

Use MultiByteToWideChar with NULL as the last parameter to predict the required buffer size. For error handling, only use PF_Err_OUT_OF_MEMORY as the error return, because PF_Err_INTERNAL_STRUCT_DAMAGED tells AE the effect instance is bad and AE won't talk to it again. PF_Err_OUT_OF_MEMORY is the polite way of telling AE a certain operation has failed and its results should be ignored.

*Contributors: [**shachar carmi**](../contributors/shachar-carmi/) · Source: adobe-forum-sdk · 2024-03-01 · Tags: `string-conversion`, `utf16`, `utf8`, `error-handling`, `buffer-size`*

---

## Should error handling be done when checking out layers, and should check-in occur immediately after use?

Yes, error handling should be implemented for checkout operations. Check-in should be done immediately after getting the value. In a loop structure, you should checkout a frame, check for errors, perform the job if no error occurred, and then check-in the frame before moving to the next iteration. Each checked-out resource must be checked in, whether it was used or not.

```cpp
For X
     Checkout frame x
         If (no err) dojob
     Err2(check-in frame x)
End of loop
```

*Tags: `layer-checkout`, `aegp`, `error-handling`, `render-loop`, `threading`*

---

## How should error handling be structured when developing After Effects plugins?

Use std::expected for function returns instead of traditional error codes, allowing for proper .then chaining. Wrap all AE handles in classes with proper construction and destruction via smart pointers. Implement an allocation class manager that clears allocations when exiting entry functions or idle hooks in AEGP, since pointers are not valid between AE calls. In debug mode, use a cache validator to detect extra memory still allocated at endpoints.

*Tags: `aegp`, `memory`, `debugging`, `error-handling`*

---

## What pattern should be used to convert traditional After Effects C API functions to Rust?

Convert functions that return PF_Err with output parameters to Rust Result types. For example, a C function like `PF_Err GetThing(Thing* output);` becomes `fn get_thing() -> Result<Thing, Error>;` in the Rust crate.

```cpp
// C API
PF_Err GetThing(Thing* output);

// Rust
fn get_thing() -> Result<Thing, Error>;
```

*Tags: `scripting`, `rust`, `error-handling`*

---

## Does PF_Interrupt return a value that triggers catch blocks or just an error code?

PF_Interrupt returns an error code (PF_err), not an exception. A non-NULL PF_err does not trigger a catch block in the traditional sense—the error handling depends on how the calling code checks the error return value rather than exception-based control flow.

*Tags: `aegp`, `debugging`, `error-handling`*

---

## How can error handling be improved when developing After Effects plugins?

Use std::expected for all function returns instead of traditional error codes. This allows for proper error handling chains using .then() chaining, making error handling much easier and more maintainable. Wrap this in a result struct containing both error codes and results.

*Tags: `aegp`, `error-handling`, `cpp`, `best-practices`*

---

## How should I handle string conversions between std::string and After Effects A_char and A_UTF16Char types?

Create a utility class with static methods for bidirectional string conversion. Use strcpy_s for A_char conversion and std::wstring_convert with std::codecvt_utf8_utf16 for UTF-16 conversions. Always ensure null-termination and check buffer bounds before copying. Use try-catch blocks for UTF-16 conversion to handle range_error exceptions.

```cpp
#include <locale>
#include <codecvt>
#include <string>
class AEStringConverter {
public:
  static PF_Err StringToAChar(const std::string& inString, A_char* outAChar, size_t bufferSize) {
    PF_Err err = PF_Err_NONE;
    errno_t copyResult = strcpy_s(outAChar, bufferSize, inString.c_str());
    if (copyResult != 0) err = PF_Err_OUT_OF_MEMORY;
    return err;
  }
  static PF_Err StringToAUTF16Char(const std::string& inString, A_UTF16Char* outAChar, size_t maxOutChars) {
    PF_Err err = PF_Err_NONE;
    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    std::wstring utf16String = converter.from_bytes(inString);
    if (utf16String.size() + 1 > maxOutChars) return PF_Err_OUT_OF_MEMORY;
    std::copy(utf16String.begin(), utf16String.end(), outAChar);
    outAChar[utf16String.size()] = L'\0';
    return err;
  }
  static PF_Err AUTF16CharToString(const A_UTF16Char* inAUTF16Char, std::string* outString) {
    PF_Err err = PF_Err_NONE;
    try {
      std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
      *outString = converter.to_bytes((wchar_t*)inAUTF16Char);
    } catch (const std::range_error&) {
      err = PF_Err_OUT_OF_MEMORY;
    } catch (...) {
      err = PF_Err_OUT_OF_MEMORY;
    }
    return err;
  }
};
```

*Tags: `sdk`, `string-handling`, `character-encoding`, `error-handling`, `cross-platform`, `build`*

---

## What is the correct error code to return from an After Effects plugin when a string conversion or operation fails?

Use PF_Err_OUT_OF_MEMORY as the standard error code for failed operations. Avoid PF_Err_INTERNAL_STRUCT_DAMAGED because it tells After Effects that the effect instance is corrupted and After Effects will stop communicating with that instance. PF_Err_OUT_OF_MEMORY is the appropriate way to signal that an operation failed and its results should be ignored.

*Tags: `sdk`, `error-handling`, `aegp`*

---
