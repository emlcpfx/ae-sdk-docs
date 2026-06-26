# Character Encoding

> 1 Q&A · source: AE plugin dev community Discord

### How should I handle string conversions between std::string and After Effects A_char and A_UTF16Char types?

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

*Tags: `build`, `character-encoding`, `cross-platform`, `error-handling`, `sdk`, `string-handling`*

---
