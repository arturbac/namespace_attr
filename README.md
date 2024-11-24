# C++ Namespace Attributes & Attribute Aliases Proposal
## Core Attributes & Custom Attribute Aliases for Namespaces

## Motivation & Impact

This proposal introduces two key features to C++:
1. Ability to apply attributes to namespaces, affecting all contained declarations
2. Mechanism to define aliases for sets of attributes, enabling consistent attribute policies

These additions provide powerful tools for code organization, maintenance, and evolution while maintaining complete backward compatibility.

### Primary Goals
1. Simplify code maintenance by reducing repetitive attribute declarations
2. Enable consistent code style and behavior across large codebases
3. Provide mechanism for gradual language evolution through opt-in features
4. Support team-specific coding standards through attribute aliases
5. Maintain 100% backward compatibility in single translation unit by:
   - Preserving existing behavior for unannotated namespaces
   - Requiring consistent attributes within translation units
   - Allowing mixed usage of attributed and non-attributed code
   - Supporting incremental adoption without breaking changes
   - Enabling safe refactoring of existing code

### Benefits
1. **Reduced Boilerplate**
   - Single declaration for common attributes
   - Fewer lines of repetitive code
   - Lower maintenance burden

2. **Enhanced Consistency**
   - Enforced attribute policies across namespaces
   - Reduced chance of missing attributes
   - Clearer code intent

3. **Better Tooling Support**
   - Clear attribute policies for static analysis
   - Easier refactoring
   - Better IDE support potential

4. **Future Evolution Path**
   - Framework for new language features
   - Backward compatible opt-in mechanism
   - Support for team-specific extensions

### Potential Risks & Mitigations
1. **Code Clarity**
   - Risk: Attribute inheritance might be non-obvious
   - Mitigation: Clear documentation requirements and IDE support

2. **Compilation Performance**
   - Risk: Additional attribute processing overhead
   - Mitigation: Reduced overall code size due to fewer attribute declarations and potential for compile-time optimizations

3. **Migration Challenges**
   - Risk: Inconsistent adoption across large codebases
   - Mitigation: Gradual opt-in approach

4. **Feature Abuse**
   - Risk: Overuse of custom attributes
   - Mitigation: Clear naming conventions and best practices

## 1. Core Design Principles

### 1.1 Basic Rules
- Unannotated namespaces follow standard C++ behavior
- Annotated namespaces must have consistent attributes across all declarations
- Support for attribute aliases using vendor/project-specific namespaces
- Focus on essential attributes with clear use cases
- Individual functions can override namespace-level attributes when needed

### 1.2 Core Supported Attributes
The following attributes are proposed as initial examples demonstrating the utility and scope of namespace attributes. The mechanism is extensible to support additional attributes as needed:

#### [[nodiscard]]
- Enforces return value usage across namespace
- Reduces error-prone code by preventing ignored returns
- Reduces number of common errors by forgetting to annotate with nodiscard an n-th function declaration
- Can be overridden on individual functions using [[maybe_unused]]
- Particularly valuable for error-handling and resource management

Example of [[nodiscard]] override:
```cpp
[[nodiscard]] namespace algorithms {
    int binary_search(span<int> data, int value);  // nodiscard applied
    
    [[maybe_unused]] // Overrides namespace nodiscard
    int debug_search(span<int> data, int value);   // can ignore return
}
```

#### [[maybe_unused]]
- Suppresses unused entity warnings namespace-wide
- Useful for debug, testing, and optional feature implementations
- Can override namespace-level [[nodiscard]] on individual functions
- Reduces noise in development builds

#### [[deprecated]]
- Marks entire APIs as deprecated
- Supports gradual API evolution
- Enables clear migration paths

#### [[constexpr_default]] (new)
- Makes constexpr the default for all functions in namespace
- Supports compile-time evaluation by default
- Non-inline functions or explicitly marked inline functions opt-out of constexpr behavior
```cpp
[[constexpr_default]] namespace math {
    // Implicitly constexpr due to namespace attribute
    double sqrt(double x) { 
        return /*...*/; 
    }
    
    // Opt-out through non-inline
    double expensive_calc(double x);  // Not constexpr
    
    // Opt-out through explicit inline
    inline double another_calc(double x) { // Not constexpr
        return /*...*/;
    }
}
```

## 2. Attribute Consistency Model

### 2.1 Basic Declaration Rules
```cpp
// All declarations must match
[[nodiscard]] namespace math {
    double sqrt(double);
}

[[nodiscard]] namespace math {  // OK: matches previous declaration
    double pow(double, double);
}

namespace math {  // Error: missing [[nodiscard]] from previous declaration
    double cos(double);
}
```

### 2.2 Multiple Attributes
```cpp
[[nodiscard, constexpr_default]] namespace algorithms {
    int binary_search(span<int> data, int value);
}

[[nodiscard, constexpr_default]] namespace algorithms {  // OK: exact match
    int lower_bound(span<int> data, int value);
}
```

## 3. Nested Namespace Behavior

### 3.1 Attribute Inheritance
- Nested namespaces inherit all attributes from their parent namespace
- Multiple levels of nesting accumulate attributes from all parent levels
- Inherited attributes behave as if directly applied to the nested namespace

Example of attribute inheritance:
```cpp
[[nodiscard]] namespace parent {
    int parent_func();  // nodiscard applied
    
    namespace child {   // inherits nodiscard
        int child_func();  // nodiscard applied
        
        namespace grandchild {  // inherits nodiscard
            int grandchild_func();  // nodiscard applied
        }
    }
}
```

### 3.2 Attribute Override Rules
- Nested namespaces can override parent namespace attributes
- Override applies to the entire nested namespace and its children
- Original parent attributes remain unchanged
- Multiple attributes are handled individually for override purposes

Example of attribute override:
```cpp
[[nodiscard]] namespace algorithms {
    int parent_search(span<int>);  // nodiscard applied
    
    [[maybe_unused]] namespace debug {  // overrides parent nodiscard
        int debug_search(span<int>);    // maybe_unused applied
        
        namespace detail {  // inherits debug's maybe_unused
            int helper_func();  // maybe_unused applied
        }
    }
    
    namespace optimized {  // inherits parent nodiscard
        int fast_search(span<int>);  // nodiscard applied
    }
}
```

### 3.3 Inline Namespace Behavior
- Inline namespaces maintain their own attribute rules when declared
- When declarations are inlined into parent namespace, they retain their original attributes
- This allows for fine-grained control of attributes in versioned namespaces

Example of inline namespace attributes:
```cpp
[[nodiscard]] namespace math {
    double basic_sqrt(double);  // nodiscard applied
    
    [[maybe_unused]] inline namespace v2 {
        double advanced_sqrt(double);  // maybe_unused applied
        
        namespace detail {  // inherits v2's maybe_unused
            double helper_func();  // maybe_unused applied
        }
    }
    
    // When v2 is inlined:
    // math::basic_sqrt      - nodiscard applied
    // math::advanced_sqrt   - maybe_unused applied (from v2)
    // math::detail::helper_func - maybe_unused applied (from v2)
}
```

### 3.4 Multiple Attribute Interaction
```cpp
[[nodiscard, constexpr_default]] namespace compute {
    int base_calc(int);  // both attributes applied
    
    [[maybe_unused]] namespace debug {
        int debug_calc(int);  // maybe_unused only, keeps constexpr_default
        
        [[deprecated]] namespace old {  // combines maybe_unused and deprecated
            int legacy_calc(int);  // maybe_unused and deprecated applied
        }
    }
    
    [[deprecated]] inline namespace v1 {
        // When inlined, these keep deprecated but inherit nodiscard
        int old_calc(int);  // nodiscard and deprecated applied
    }
}
```


## 4. Backward Compatibility

### 4.1 Example of Backward Compatibility
```cpp
// old_header.hpp (existing code)
namespace math {
    double sqrt(double);
}

// new_feature.hpp (new code)
[[nodiscard]] namespace new_math {
    double advanced_sqrt(double);
}

// mixed_usage.hpp (mixing old and new)
namespace math {  // OK: continues without attributes as before
    double cos(double);
}

[[nodiscard]] namespace new_math {  // OK: maintains new attributes
    double sin(double);
}

// future_upgrade.hpp (gradual adoption)
[[nodiscard]] namespace math {  // Error: can't add attributes to math
    double tan(double);         // Must be consistent throughout program
}
```

### 4.2 Cross Translation Unit Consistency

1. **Attribute Consistency**
   - Attributes must be consistent across all translation units
   - Same namespace must have same attributes throughout the program
   - Violating attribute consistency is ill-formed
   - Compile-time error if attributes mismatch between translation units

2. **Header File Impact**
   - Headers should declare consistent attributes
   - Including headers with conflicting attributes is ill-formed
   - Tools should verify attribute consistency during build

3. **Modules Interaction**
   - Module interface must declare definitive attributes
   - Module implementation units must match interface attributes
   - Importing modules with conflicting attributes is ill-formed

Example of consistency requirements:
```cpp
// header1.hpp
[[nodiscard]] namespace math {
    double sqrt(double);
}

// header2.hpp
namespace math {  // Error: must have [[nodiscard]] to be consistent
    double pow(double, double);
}

// header3.hpp
[[nodiscard]] namespace math {  // OK: consistent with other declarations
    double sin(double);
}
```

## 5. Attribute Aliases

### 5.1 Alias Declaration Syntax
```cpp
// Project-specific attribute combinations
using [[projectx::performance_critical]] = [[nodiscard, constexpr_default]];
using [[company::deprecated_api]] = [[deprecated("use v2 API"), maybe_unused]];
using [[team::core_math]] = [[nodiscard, constexpr_default]];

// Usage
[[projectx::performance_critical]] namespace fast_math {
    double quick_sqrt(double);
}

// Equivalent to:
[[nodiscard, constexpr_default]] namespace fast_math {
    double quick_sqrt(double);
}
```

### 5.2 Alias Rules
1. Namespace prefix must use format: `identifier::identifier`
2. Cannot conflict with existing vendor namespaces (clang::, gnu::, msvc::)
3. Aliases are visible after declaration in the same translation unit
4. Aliases can combine multiple attributes
5. Aliases cannot be redefined in the same translation unit

## 6. Examples

### 6.1 Complex Alias Usage
```cpp
// Define company-wide attribute policies
using [[megacorp::core_api]] = [[nodiscard, constexpr_default]];
using [[megacorp::stable_api]] = [[nodiscard]];
using [[megacorp::legacy_api]] = [[deprecated("contact team X for migration")]];

// Usage in code
[[megacorp::core_api]] namespace core {
    template<typename T>
    T min(T a, T b) { return a < b ? a : b; }
}

[[megacorp::legacy_api]] namespace old {
    void old_function();
}
```

### 6.2 Mixed Attribute Usage
```cpp
// Define performance-critical code attributes
using [[project::perf]] = [[nodiscard, constexpr_default]];

// Apply to key namespaces
[[project::perf]] namespace algorithms {
    bool binary_search(span<const int> data, int value);
}

// Direct attribute usage for special cases
[[nodiscard, deprecated("use algorithms::binary_search")]] 
namespace legacy_algorithms {
    bool old_search(const int* data, size_t size, int value);
}
```
## 7. Future Considerations

1. Support for modules
2. Additional core attributes
3. Cross-translation unit consistency checking
4. IDE support for attribute visualization
5. Refactoring tools for attribute management

## 8. Evolution Path

This proposal provides foundation for:

1. Future language feature adoption
2. Team-specific coding standards
3. Gradual modernization of C++ codebases
4. Enhanced static analysis capabilities

## 9. Conclusion

This focused proposal provides a practical solution for namespace-level attributes with the addition of powerful attribute aliases. The design maintains backward compatibility while enabling teams to define and enforce their own attribute policies. The cross-translation unit consistency requirements ensure well-defined behavior across the entire program.
