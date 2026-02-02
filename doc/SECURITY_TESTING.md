# Security Testing Guide for Windows Terminal

This document provides guidance on security testing practices for the Windows Terminal project.

## Overview

Security testing is an integral part of the development process. This guide covers:
- Automated security testing (fuzzing, static analysis)
- Manual security testing
- Test case development
- Security-focused code review

---

## Automated Security Testing

### Fuzzing

#### OneFuzz Integration
The project uses Microsoft's OneFuzz for continuous fuzzing:

**Active Fuzzing Targets:**
- VT Parser (`src/terminal/parser/ft_fuzzer/`)
- Host API (`src/host/ft_fuzzer/`)
- Input handling

**Running Fuzzing Locally:**
```bash
# Build fuzzing targets
.\build\scripts\build-fuzzing.ps1

# Run VT parser fuzzer
.\bin\x64\Debug\VtCommandFuzzer.exe corpus\

# Run with ASan for better crash detection
.\bin\x64\Debug\VtCommandFuzzer_asan.exe corpus\
```

**Corpus Management:**
- Seed corpus in `src/*/ft_fuzzer/seed_corpus/`
- Add interesting test cases to seed corpus
- Regularly update corpus from CI fuzzing runs

#### AFL/libFuzzer Best Practices
- Run for at least 24 hours for new code
- Monitor coverage metrics
- Investigate all crashes and hangs
- Add crash-inducing inputs as regression tests

### Static Analysis

#### CppCoreCheck
Enable maximum static analysis in your builds:

```xml
<!-- In vcxproj or props file -->
<RunCodeAnalysis>true</RunCodeAnalysis>
<CodeAnalysisRuleSet>AllRules.ruleset</CodeAnalysisRuleSet>
<EnableCppCoreCheck>true</EnableCppCoreCheck>
```

**Key Rules to Monitor:**
- C26400-C26499: Lifetime and ownership
- C26800-C26899: Bounds and arithmetic
- C6001-C6400: Buffer overruns and memory issues

#### Manual Static Analysis Commands
```bash
# Run code analysis on specific project
msbuild TerminalCore.vcxproj /p:RunCodeAnalysis=true

# Check for specific warnings
msbuild /p:WarningLevel=4 /p:TreatWarningsAsErrors=true
```

### Dynamic Analysis

#### Address Sanitizer (ASan)
ASan detects:
- Buffer overflows (heap and stack)
- Use-after-free
- Double-free
- Memory leaks

**Enable ASan:**
```bash
# Build with ASan
msbuild /p:Configuration=Debug /p:Platform=x64 /p:EnableASan=true

# Run tests with ASan
.\bin\x64\Debug\TerminalCore.Unit.Tests.exe
```

**ASan Output:**
ASan will print detailed reports of any memory safety violations.

#### Debug CRT
The Debug C Runtime provides additional checks:
- Buffer overruns
- Heap corruption
- Uninitialized memory

**Always test in Debug builds before submitting PRs.**

---

## Manual Security Testing

### Input Validation Testing

#### VT Sequence Testing
Test the VT parser with malformed sequences:

```cpp
// Test cases for VT sequence validation
TEST_METHOD(VTParser_RejectsOversizedParameters)
{
    // CSI with extremely large parameter
    std::string input = "\x1b[9999999999999999999A";
    VERIFY_NO_CRASHES(parser.ProcessString(input));
}

TEST_METHOD(VTParser_HandlesInvalidUTF8)
{
    // Invalid UTF-8 sequences
    std::string input = "\xF0\x82\x82\xAC";  // Overlong encoding
    VERIFY_NO_CRASHES(parser.ProcessString(input));
}
```

#### Settings File Testing
Test JSON parsing with malicious inputs:

```json
{
    "profiles": {
        "list": [
            {
                "commandline": "../../../etc/passwd",
                "startingDirectory": "../../.."
            }
        ]
    }
}
```

**Expected behavior:** Paths should be validated and sanitized.

#### Clipboard Testing
Test clipboard operations with:
- Very large strings (> 1MB)
- Binary data
- Special characters and control codes
- Malformed Unicode

### Boundary Testing

#### Buffer Size Limits
Test with extreme values:

```cpp
TEST_METHOD(FontResource_HandlesMaxDimensions)
{
    const WORD maxWidth = std::numeric_limits<WORD>::max();
    const WORD maxHeight = std::numeric_limits<WORD>::max();
    
    // Should either succeed or fail gracefully with E_ARITHMETIC_OVERFLOW
    auto result = CreateFontResource(maxWidth, maxHeight);
    VERIFY_IS_TRUE(SUCCEEDED(result) || result == E_ARITHMETIC_OVERFLOW);
}

TEST_METHOD(Buffer_HandlesLargeAllocations)
{
    // Test with buffer size at reasonable maximum
    const size_t hugeSize = 1024 * 1024 * 100; // 100MB
    
    VERIFY_NO_CRASHES({
        try {
            std::vector<byte> buffer(hugeSize);
        } catch (std::bad_alloc&) {
            // Expected on low memory
        }
    });
}
```

#### Integer Overflow Testing
Test arithmetic with edge cases:

```cpp
TEST_METHOD(Math_DetectsOverflow)
{
    base::CheckedNumeric<uint32_t> large = std::numeric_limits<uint32_t>::max();
    large += 1;
    VERIFY_IS_FALSE(large.IsValid());  // Should detect overflow
}
```

### Process Creation Testing

Test command line handling:

```cpp
TEST_METHOD(ConPty_ValidatesCommandLine)
{
    // Test with various potentially problematic command lines
    std::vector<std::wstring> testCases = {
        L"",  // Empty
        L"cmd.exe & malicious.exe",  // Command injection attempt
        L"cmd.exe | more",  // Pipe
        std::wstring(10000, L'A'),  // Very long
    };
    
    for (const auto& cmdline : testCases) {
        // Should handle gracefully without crashes
        VERIFY_NO_CRASHES(CreateConPtyConnection(cmdline));
    }
}
```

---

## Security Test Case Development

### Test Case Template

```cpp
class SecurityTests
{
    TEST_CLASS(SecurityTests);

    TEST_METHOD(ComponentName_SecurityProperty)
    {
        // Arrange: Set up test conditions
        auto component = CreateComponent();
        auto maliciousInput = CreateMaliciousInput();
        
        // Act: Execute the operation
        auto result = component.ProcessInput(maliciousInput);
        
        // Assert: Verify secure behavior
        VERIFY_IS_TRUE(FAILED(result)); // Should reject bad input
        // OR
        VERIFY_IS_TRUE(SUCCEEDED(result)); // Should handle safely
        VERIFY_NO_MEMORY_LEAKS();
        VERIFY_NO_CRASHES();
    }
};
```

### Example Security Tests

#### 1. Path Traversal Prevention
```cpp
TEST_METHOD(Settings_PreventstPathTraversal)
{
    const std::wstring maliciousPath = L"..\\..\\..\\windows\\system32\\cmd.exe";
    
    VERIFY_THROWS(
        ValidateSettingsPath(maliciousPath),
        std::invalid_argument);
}
```

#### 2. Integer Overflow Prevention
```cpp
TEST_METHOD(Renderer_PreventsFontOverflow)
{
    const WORD width = 0xFFFF;
    const WORD height = 0xFFFF;
    
    VERIFY_THROWS_HR(
        fontResource.SetSize(width, height),
        E_ARITHMETIC_OVERFLOW);
}
```

#### 3. Buffer Overrun Prevention
```cpp
TEST_METHOD(Buffer_PreventsCopyOverrun)
{
    std::array<byte, 10> dest;
    std::array<byte, 20> source;
    
    // Should fail or truncate safely
    VERIFY_NO_CRASHES(SafeCopy(dest, source));
    
    // Verify no buffer overrun occurred
    VERIFY_MEMORY_INTACT(dest);
}
```

#### 4. Injection Prevention
```cpp
TEST_METHOD(CommandLine_PreventInjection)
{
    const std::wstring injection = L"cmd.exe & malicious.exe";
    
    // Should escape or reject injection attempts
    auto sanitized = SanitizeCommandLine(injection);
    VERIFY_IS_FALSE(sanitized.contains(L'&'));
}
```

---

## Security-Focused Code Review

### Review Checklist

When reviewing code for security issues, check:

#### Memory Safety
- [ ] No raw `new`/`delete` without RAII
- [ ] Buffer allocations use checked arithmetic
- [ ] Array accesses are bounds-checked
- [ ] No pointer arithmetic without validation

#### Input Validation
- [ ] All external input is validated
- [ ] Validation happens before use
- [ ] Error cases are handled
- [ ] Validation uses allowlists

#### Integer Safety
- [ ] Size calculations use `CheckedNumeric`
- [ ] Type conversions use `gsl::narrow`
- [ ] No unchecked multiplication/addition

#### String Safety
- [ ] Uses safe string functions (`_s` variants)
- [ ] String lengths are validated
- [ ] No C-style string operations without bounds

#### Resource Management
- [ ] All resources use RAII
- [ ] No leaked handles or memory
- [ ] Exception-safe resource handling

### Security-Critical Areas

Pay extra attention when reviewing:

1. **VT Parser** (`src/terminal/parser/`)
   - Complex state machine
   - Handles untrusted input
   - Many edge cases

2. **Buffer Management** (`src/buffer/out/`)
   - Direct memory manipulation
   - Size calculations
   - Unicode handling

3. **Settings Deserialization** (`src/cascadia/TerminalSettingsModel/`)
   - JSON parsing
   - User-provided data
   - Profile generation

4. **Process Creation** (`src/cascadia/TerminalConnection/`)
   - Command line construction
   - Environment manipulation
   - Privilege handling

5. **Font Rendering** (`src/renderer/`)
   - Complex size calculations
   - External font data
   - Binary data processing

---

## Regression Testing

### Adding Security Regression Tests

When a security issue is fixed:

1. Create a test that reproduces the issue
2. Verify the test fails on the vulnerable code
3. Verify the test passes with the fix
4. Add the test to the appropriate test suite

Example:
```cpp
// Regression test for GH#12345 - Integer overflow in font calculation
TEST_METHOD(FontResource_GH12345_PreventIntegerOverflow)
{
    const WORD width = 32767;  // Max safe value
    const WORD height = 32767;
    
    // This should not overflow and crash
    VERIFY_NO_CRASHES({
        auto resource = CreateFontResource(width, height);
    });
}
```

---

## Continuous Security Testing

### CI/CD Integration

Security tests run automatically on:
- Every PR (unit tests, static analysis)
- Nightly builds (fuzzing, extended tests)
- Pre-release (full security suite)

### Metrics to Track

Monitor these security metrics:
- Static analysis warnings (should be 0)
- Fuzzing crashes (should be 0)
- Code coverage in security-critical areas (> 80%)
- Time to fix security issues (< 7 days for critical)

---

## Security Test Maintenance

### Regular Activities

**Weekly:**
- Review fuzzing results
- Triage static analysis warnings
- Update test cases for new features

**Monthly:**
- Review code coverage
- Update fuzzing corpus
- Review security test suite

**Quarterly:**
- Security-focused test sprint
- Threat model review
- Penetration testing (if applicable)

---

## Tools and Resources

### Recommended Tools

1. **Fuzzing:**
   - OneFuzz (Microsoft)
   - AFL++
   - libFuzzer

2. **Static Analysis:**
   - Visual Studio Code Analysis
   - CppCoreCheck
   - Clang-Tidy

3. **Dynamic Analysis:**
   - Address Sanitizer (ASan)
   - Memory Sanitizer (MSan)
   - Thread Sanitizer (TSan)
   - Debug CRT

4. **Code Review:**
   - GitHub Security Advisories
   - CodeQL (GitHub)
   - SonarQube

### Learning Resources

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Microsoft SDL Testing](https://www.microsoft.com/en-us/securityengineering/sdl/practices)
- [Fuzzing Book](https://www.fuzzingbook.org/)
- [CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines/)

---

## Reporting Security Issues

**Never report security vulnerabilities via GitHub issues.**

Instead, report to:
- Microsoft Security Response Center (MSRC): https://msrc.microsoft.com/create-report
- Email: secure@microsoft.com
- See [SECURITY.md](../SECURITY.md) for full details

---

## Questions?

For questions about security testing:
1. Review this guide and [SECURITY_PRACTICES.md](SECURITY_PRACTICES.md)
2. Check existing security tests for examples
3. Consult with the security team
4. Refer to Microsoft SDL documentation
