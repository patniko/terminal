# Security Best Practices for Windows Terminal Development

This document outlines security best practices for contributors to the Windows Terminal project.

## Table of Contents
- [Memory Safety](#memory-safety)
- [Input Validation](#input-validation)
- [Integer Arithmetic](#integer-arithmetic)
- [String Handling](#string-handling)
- [Process Creation](#process-creation)
- [File Operations](#file-operations)
- [Testing](#testing)
- [Code Review](#code-review)

---

## Memory Safety

### Use RAII and Smart Pointers
Always use RAII (Resource Acquisition Is Initialization) and smart pointers to manage resources.

**✅ Good:**
```cpp
auto buffer = std::make_unique<byte[]>(size);
wil::unique_hfont font{ CreateFontIndirect(&logFont) };
```

**❌ Bad:**
```cpp
byte* buffer = new byte[size];  // Manual memory management
HFONT font = CreateFontIndirect(&logFont);  // No automatic cleanup
```

### Avoid Raw Pointers for Ownership
Use raw pointers only for non-owning references.

**✅ Good:**
```cpp
void ProcessData(const std::span<byte> data) { }  // Non-owning view
std::unique_ptr<Data> data = std::make_unique<Data>();  // Ownership
```

### Use std::span for Array Views
Replace pointer + size pairs with `std::span`.

**✅ Good:**
```cpp
void ProcessBuffer(std::span<const byte> buffer);
```

**❌ Bad:**
```cpp
void ProcessBuffer(const byte* buffer, size_t size);
```

---

## Input Validation

### Validate All External Input
Always validate data from:
- Settings files (JSON)
- Command line arguments
- Environment variables
- VT escape sequences
- Clipboard data
- Network connections

**✅ Good:**
```cpp
if (width > MAX_FONT_WIDTH || width < MIN_FONT_WIDTH) {
    LOG_HR(E_INVALIDARG);
    return E_INVALIDARG;
}
```

### Use Allowlists Over Denylists
Prefer allowlists (what is permitted) over denylists (what is blocked).

**✅ Good:**
```cpp
if (fileExtension == L".json" || fileExtension == L".txt") {
    // Process file
}
```

**❌ Bad:**
```cpp
if (fileExtension != L".exe" && fileExtension != L".dll") {
    // Process file - misses many dangerous extensions
}
```

---

## Integer Arithmetic

### Use Checked Arithmetic for Size Calculations
Always use `base::CheckedNumeric` for calculations that determine buffer sizes.

**✅ Good:**
```cpp
base::CheckedNumeric<DWORD> totalSize = width;
totalSize *= height;
THROW_HR_IF(E_ARITHMETIC_OVERFLOW, !totalSize.IsValid());
auto buffer = std::vector<byte>(totalSize.ValueOrDie());
```

**❌ Bad:**
```cpp
DWORD totalSize = width * height;  // Can overflow
auto buffer = std::vector<byte>(totalSize);
```

### Use gsl::narrow for Type Conversions
Use `gsl::narrow` or `gsl::narrow_cast` for safe type conversions.

**✅ Good:**
```cpp
const auto width = gsl::narrow<WORD>(value);  // Throws on overflow
const auto height = gsl::narrow_cast<int>(size);  // Explicit cast
```

**❌ Bad:**
```cpp
const auto width = (WORD)value;  // Silent truncation
```

### Check for Division by Zero
Always validate denominators before division.

**✅ Good:**
```cpp
THROW_HR_IF(E_INVALIDARG, denominator == 0);
const auto result = numerator / denominator;
```

---

## String Handling

### Use Safe String Functions
Always use `_s` variants of C string functions.

**✅ Good:**
```cpp
sprintf_s(buffer, sizeof(buffer), "Value: %d", value);
wcscpy_s(dest, destSize, source);
```

**❌ Bad:**
```cpp
sprintf(buffer, "Value: %d", value);  // Buffer overflow risk
wcscpy(dest, source);  // No bounds checking
```

### Use std::string and std::wstring
Prefer C++ strings over C-style character arrays.

**✅ Good:**
```cpp
std::wstring path = GetPath();
path += L"\\file.txt";
```

**❌ Bad:**
```cpp
WCHAR path[MAX_PATH];
wcscat(path, L"\\file.txt");  // Potential overflow
```

### Validate String Lengths
Check string lengths before operations.

**✅ Good:**
```cpp
if (input.length() > MAX_COMMAND_LENGTH) {
    return E_INVALIDARG;
}
```

---

## Process Creation

### Validate Command Lines
Always validate command line strings before process creation.

**✅ Good:**
```cpp
// Command line comes from user profile settings
auto cmdline = wil::ExpandEnvironmentStringsW<std::wstring>(profileCommandLine);
// Additional validation here if needed
CreateProcessW(nullptr, cmdline.data(), ...);
```

### Use Proper Security Attributes
Specify security attributes when creating processes.

**✅ Good:**
```cpp
SECURITY_ATTRIBUTES sa{};
sa.nLength = sizeof(SECURITY_ATTRIBUTES);
sa.bInheritHandle = TRUE;
CreateProcess(..., &sa, ...);
```

### Document ShellExecute Usage
Always document why `ShellExecute` is used and what inputs it receives.

**✅ Good:**
```cpp
// Open user settings file with default editor
// filePath comes from known secure location
ShellExecute(nullptr, nullptr, filePath.c_str(), nullptr, nullptr, SW_SHOW);
```

---

## File Operations

### Validate File Paths
Check for path traversal attempts.

**✅ Good:**
```cpp
if (filePath.find(L"..") != std::wstring::npos) {
    LOG_HR(E_ACCESSDENIED);
    return E_ACCESSDENIED;
}
```

### Use Canonical Paths
Resolve paths to canonical form before security checks.

**✅ Good:**
```cpp
auto canonicalPath = std::filesystem::canonical(userPath);
if (!IsPathInAllowedDirectory(canonicalPath)) {
    return E_ACCESSDENIED;
}
```

### Set Proper File Permissions
Use security descriptors for files that need protection.

**✅ Good:**
```cpp
SECURITY_ATTRIBUTES sa{};
ConvertStringSecurityDescriptorToSecurityDescriptor(
    L"D:(A;;GA;;;AU)",  // Allow general access to authenticated users
    SDDL_REVISION_1,
    &sa.lpSecurityDescriptor,
    nullptr);
CreateFile(..., &sa, ...);
```

---

## Testing

### Write Security-Focused Tests
Create tests that specifically target security boundaries.

**✅ Good:**
```cpp
TEST_METHOD(FontResource_RejectsOversizedDimensions)
{
    // Attempt to create font with dimensions that would overflow
    const auto width = std::numeric_limits<WORD>::max();
    const auto height = std::numeric_limits<WORD>::max();
    
    VERIFY_THROWS_HR(
        CreateFontResource(width, height),
        E_ARITHMETIC_OVERFLOW);
}
```

### Use Fuzzing
Enable and maintain fuzzing for:
- VT parser
- Settings JSON parser
- Input handling
- Buffer operations

### Test Boundary Conditions
Always test:
- Maximum values
- Minimum values
- Zero values
- Negative values (for signed types)
- Off-by-one conditions

---

## Code Review

### Security Review Checklist
When reviewing code, check for:

#### Memory Safety
- [ ] No manual `new`/`delete` without RAII wrapper
- [ ] No pointer arithmetic without bounds checking
- [ ] Buffer allocations use checked arithmetic
- [ ] All resources have clear ownership

#### Input Validation
- [ ] All external input is validated
- [ ] Validation uses allowlists where possible
- [ ] Error handling is present and correct

#### Integer Safety
- [ ] Size calculations use `base::CheckedNumeric`
- [ ] Type conversions use `gsl::narrow`
- [ ] Division checks for zero

#### String Safety
- [ ] Uses `_s` variants of C string functions
- [ ] Prefers C++ strings over C arrays
- [ ] String lengths are validated

#### Process/File Operations
- [ ] ShellExecute/CreateProcess usage is documented
- [ ] File paths are validated
- [ ] Security attributes are specified

---

## Common Patterns

### Safe Buffer Allocation
```cpp
// 1. Calculate size with overflow checking
base::CheckedNumeric<size_t> requiredSize = elementCount;
requiredSize *= elementSize;
THROW_HR_IF(E_ARITHMETIC_OVERFLOW, !requiredSize.IsValid());

// 2. Allocate with safe size
auto buffer = std::vector<byte>(requiredSize.ValueOrDie());

// 3. Use std::span for safe access
auto bufferSpan = std::span<byte>(buffer);
```

### Safe String Formatting
```cpp
// Use fmt library or std::format (C++20)
auto formatted = fmt::format("Value: {}", value);

// Or sprintf_s with proper buffer size
CHAR buffer[256];
sprintf_s(buffer, sizeof(buffer), "Value: %d", value);
```

### Safe Type Narrowing
```cpp
// From larger to smaller type
const auto narrowed = gsl::narrow<uint16_t>(largeValue);  // Throws on overflow

// When overflow is acceptable (explicit intent)
const auto truncated = gsl::narrow_cast<uint16_t>(largeValue);
```

---

## Tools and Libraries

### Required Libraries
- **WIL** (Windows Implementation Libraries): RAII wrappers, error handling
- **GSL** (Guidelines Support Library): `narrow`, `span`, `not_null`
- **Chromium base::CheckedNumeric**: Overflow-safe arithmetic

### Static Analysis
- Enable `/analyze` (MSVC static analysis)
- Use CppCoreCheck rules
- Address all warnings in AuditMode builds

### Runtime Checks
- Enable Address Sanitizer (ASan) in test builds
- Use Debug CRT (checks for buffer overruns, heap corruption)
- Enable /GS (buffer security check)

---

## References

- [Windows Terminal Security Model](../SECURITY.md)
- [Microsoft Security Development Lifecycle](https://www.microsoft.com/en-us/securityengineering/sdl)
- [CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
- [OWASP Secure Coding Practices](https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/)
- [SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)

---

## Questions?

If you have security concerns or questions, please:
1. Review existing documentation in the `/doc` directory
2. Check the [Security Policy](../SECURITY.md)
3. Contact the security team for guidance
4. Report vulnerabilities to MSRC (not via GitHub issues)
