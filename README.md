# C++ Policy Scope & Attribute Aliases Proposal

Author: Artur Bać 2024.11.27
Version: 2.0

## Core Policy Scopes & Custom Attribute Aliases

## Motivation & Impact

Modern C++ developers spend considerable time adding and reviewing [[nodiscard]] attributes on functions returning std::expected, often missing some during development which leads to silent error handling failures. This repetitive task scales poorly with codebase size, where entire APIs typically share common error handling patterns. Teams waste development cycles maintaining these attributes across related functions while a single oversight can introduce subtle runtime bugs, and by 1 year practice with using expected this happens all the time.
Making std::expected's error handling non-discardable by default would eliminate this class of bugs while reducing maintenance overhead. When explicit error dismissal is needed, developers would be able by proposed opt-out  [[discardable]] attribute. This change aligns with C++'s "safe by default" philosophy while better matching developer intent in real-world applications.

For example, current practice requires marking each function individually:

```cpp
namespace file_ops {
    [[nodiscard]]
    auto open_file(string_view path) -> expected<file_handle, error_code>;
    [[nodiscard]]
    auto write_data(file_handle& file, span<const byte> data) -> expected<size_t, error_code>;
    [[nodiscard]]
    auto flush_to_disk(file_handle& file) -> expected<void, error_code>;
    auto close_file(file_handle& file) -> expected<void, error_code>;  // Oops! Missing nodiscard
    
    struct file_writer {
        [[nodiscard]]
        auto write_header() -> expected<void, error_code>;
        [[nodiscard]]
        auto write_metadata(const metadata& md) -> expected<void, error_code>;
        auto write_footer() -> expected<void, error_code>;  // Oops! Missing nodiscard again
    };
}
```

This proposal introduces two key features to C++:
1. A new `policy` scope for applying attributes to blocks of code , where keyword name 'policy' is proposed it does not matter what new keyword name will be used for the author.
2. Mechanism to define aliases for sets of attributes, enabling consistent attribute policies

This proposal would simplify this to:

```cpp
namespace file_ops {
    [[nodiscard]]  // All expected<T,E> returns will be nodiscard
    policy {
        auto open_file(string_view path) -> expected<file_handle, error_code>;
        auto write_data(file_handle& file, span<const byte> data) -> expected<size_t, error_code>;
        auto flush_to_disk(file_handle& file) -> expected<void, error_code>;
        auto close_file(file_handle& file) -> expected<void, error_code>;
        
        struct file_writer {
            auto write_header() -> expected<void, error_code>;
            auto write_metadata(const metadata& md) -> expected<void, error_code>;
            auto write_footer() -> expected<void, error_code>;
        };
        
        [[discardable]]  // Explicitly opt-out of nodiscard
        auto try_cleanup() -> expected<void, error_code>;
    }
}
```

These additions transform how we manage code policies, making them explicit and compiler-enforced rather than relying on error-prone manual attribute application. By introducing a dedicated policy scope for blocks of code, we can ensure consistent error handling across related functions while maintaining the flexibility to override them when needed. This approach naturally aligns with how error handling code is actually organized and dramatically reduces the maintenance burden of managing attributes in large codebases.

Below is citation from P3081, which explains why I think clang-tidy is not a solution here for missing nodiscard attributes.
Policy should be applied on code level and not by external tool by checking static analisis on code and every function.
```
Note this good summary by David Chisnall in a January 2024 FreeBSD mailing list post, [Chisnall2024]:
“Between modern C++ with static analysers and Rust, there was a small safety delta.
The recommendation [to prefer Rust for new projects] was primarily based on a human-
factors decision: it’s far easier to prevent people from committing code that doesn’t compile
than it is to prevent them from committing code that raises static analysis warnings.
If a project isn’t doing pre-merge static analysis, it’s basically impossible.”
```

## 1. Core Design Principles

### 1.1 Basic Rules
- Code outside policy blocks follows standard C++ behavior
- Policy blocks create an explicit scope for attribute application
- Support for attribute aliases using vendor/project-specific sets of attributes
- Focus on essential attributes with clear use cases
- Individual declarations can override policy block attributes when needed

### 1.1.1 Policy Block Scope
The policy block creates an explicit scope for attribute application:

```cpp
namespace algorithms 
{
    [[nodiscard]]
    policy 
    {
        auto binary_search(span<int> data, int value) -> int;  // nodiscard applied
    }

    // Outside policy block
    auto debug_search(span<int> data, int value) -> int;   // standard C++ rules apply
}
```

### 1.2 Core Supported Attributes
The following attributes/keywords are proposed as initially supported within policy blocks:

#### [[nodiscard]]
```cpp
namespace algorithms 
{
    [[nodiscard]]
    policy 
    {
        auto binary_search(span<int> data, int value) -> int;  // nodiscard applied
        
        [[discardable]] // Overrides policy nodiscard
        auto debug_search(span<int> data, int value) -> int;   // can ignore return
    }
}
```

#### [[discardable]] new attribute
- Explicitly allows return value to be discarded
- Can override policy-level [[nodiscard]]

#### [[deprecated]]
- Marks entire APIs as deprecated within policy block
- Supports gradual API evolution
- Enables clear migration paths

#### Standardized profiles from P3081
Standardized profiles from P3081 naturally fit within policy blocks:

type_safety, bounds_safety, initialization_safety, lifetime_safety, arithmetic_safety

## 2. Policy Nesting and Inheritance

### 2.1 Nested Policy Blocks
- Nested policy blocks inherit attributes from parent policies
- Multiple levels of nesting accumulate attributes
- Override rules apply at each level

```cpp
namespace algorithms 
{
    [[nodiscard]]
    policy 
    {
        auto parent_func() -> int;  // nodiscard applied
        
        [[discardable]]
        policy 
        {
            auto debug_func() -> int;  // discardable applied, overrides nodiscard
            
            [[deprecated]]
            policy 
            {
                auto legacy_func() -> int;  // discardable and deprecated applied
            }
        }
    }
}
```

## 3. Attribute Application Rules

### 3.1 Non-Enforced Attributes
nodiscard attribute should be silently ignored for incompatible declarations:

```cpp
namespace algorithms {
    [[nodiscard]]
    policy {
        auto get_value() -> int;          // ✓ nodiscard applied
        auto process_data() -> void;      // ✓ nodiscard silently ignored (void return)
        
        struct result_t {
            auto query() -> expected<int>;          // ✓ nodiscard applied
            auto clear() -> expected<void>;         // ✓ nodiscard applied
            ~result_t();                  // ✓ nodiscard silently ignored
        };
    }
}
```

### 3.2 Enforced Attributes

Safety profiles follow strict enforcement rules:

```cpp
using [[required company::safety]] = [[enforce(type_safety), enforce(bounds_safety)]];

namespace algorithms {
    [[required company::safety]]
    policy {
        auto safe_calc(int x) -> int;     // ✓ enforces both safety rules
        
        auto unsafe_cast(void* ptr) -> void* {  // × Error: violates type safety
            return ptr;
        }
        
        template<typename T>
        auto array_access(T* arr, size_t i) -> T* {  // × Error: violates bounds safety
            return &arr[i];  
        }
    }
}
```

## 4. Attribute Aliases

### 4.1 Alias Declaration Syntax
```cpp
// Project-specific attribute combinations
using [[projectx::safety_critical]] = [[nodiscard, enforce(type_safety), enforce(lifetime_safety)]];
using [[company::deprecated_api]] = [[deprecated("use v2 API"), discardable]];

namespace fast_math 
{
    [[required projectx::safety_critical]]
    policy 
    {
        auto quick_sqrt(double) -> double;  // Gets all safety_critical attributes
    }
}
```

### 4.2 Alias Rules
1. Missing attribute (alias) for attribute marked with require shoudl cause compile error
2. Aliases visible after declaration in same translation unit
3. Cannot be redefined in same translation unit to different values, redeclaration with same values should be no op.
4. Unknown **required** aliases cause compile error

## 5. Conclusion

The policy-based approach provides a cleaner, more explicit mechanism for managing attributes across blocks of code. The explicit policy keyword makes the intent clearer and avoids potential confusion with regular namespace attributes. This design provides a strong foundation for future language evolution while maintaining backward compatibility and enabling gradual adoption.
