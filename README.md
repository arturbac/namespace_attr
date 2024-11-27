# C++ Namespace Attributes & Attribute Aliases Proposal
## Core Attributes & Custom Attribute Aliases for Namespaces

## Motivation & Impact

C++ developers spend significant time adding and reviewing attributes like [[nodiscard]] on every function, often missing some during development which leads to bugs where function results are silently discarded. With the upcoming standardized profiles from P3081, this burden will only increase as more attributes like enforce,apply,suspend need to be consistently applied. Since functions are typically grouped by their purpose and safety requirements, manually repeating the same sets of attributes across entire APIs wastes development time and is error-prone.

For example, current practice often relies on macro definitions:

```cpp
// Current approach with macros
#if defined(_MSC_VER)
    #if defined(MYLIB_EXPORTS)
        #define MY_LIBRARY_API __declspec(dllexport) 
    #else
        #define MY_LIBRARY_API __declspec(dllimport)
    #endif
#else
    #define MY_LIBRARY_API [[visibility("default")
#endif

namespace algorithms 
{
    MY_LIBRARY_API [[nodiscard]] int binary_search(span<int> data, int value);
    MY_LIBRARY_API [[nodiscard]] int interpolation_search(span<int> data, int value);
    MY_LIBRARY_API [[nodiscard]] int jump_search(span<int> data, int value);
    // Easy to forget the macro on new functions
    int debug_search(span<int> data, int value);  // Missing attributes!
}
```

This proposal introduces two key features to C++:
1. Ability to apply attributes to namespaces, affecting all contained declarations within given block scope of namespace declaration
2. Mechanism to define aliases for sets of attributes, enabling consistent attribute policies

This proposal would simplify this to:

```cpp
#if defined(_MSC_VER)
#if defined(MYLIB_EXPORTS)
using [[required::mylib::api]] = [[msvc::dllexport, nodiscard]];
#else
using [[required::mylib::api]] = [[msvc::dllimport, nodiscard]];
#endif
#else
using [[required::mylib::api]] = [[visibility("default"),nodiscard]];
#endif

[[required::mylib::api]]
namespace algorithms 
{
    int binary_search(span<int> data, int value);      // Automatically gets all attributes
    int interpolation_search(span<int> data, int value);
    int jump_search(span<int> data, int value);
    
    [[maybe_unused]] // Only need to mark exceptions
    int debug_search(span<int> data, int value);
}
```

These additions transform how we manage code policies, making them explicit and compiler-enforced rather than relying on error-prone manual attribute application or macro definitions. By allowing attributes to be applied at the namespace level for blocks of code, we can ensure consistent policies across related functions while maintaining the flexibility to override them when needed. This approach naturally aligns with how code is actually organized and dramatically reduces the maintenance burden of managing attributes in large codebases.

### Primary Goals
1. Simplify code maintenance by reducing repetitive attribute declarations
2. Enable consistent code style and behavior across large codebases
3. Provide mechanism for gradual language evolution through opt-in features
4. Support team-specific coding standards through attribute aliases
5. Maintain 100% backward compatibility in single translation unit by:
   - Preserving existing behavior for unannotated namespaces
   - Allowing mixed usage of attributed and non-attributed code
   - Supporting incremental adoption without breaking changes
   - Enabling safe refactoring of existing code

### Benefits
1. **Reduced Boilerplate**
   - Single declaration for common attributes
   - Fewer lines of repetitive code
   - Lower maintenance burden

2. **Enhanced Consistency**
   - Enforced attribute policies across blocks of code 
   - Reduced chance of missing attributes
   - Clearer code intent

3. **Better Tooling Support**
   - Clear attribute policies for static analysis
   - Easier refactoring
   - Better IDE support potential

4. **Future Evolution Path**
   - Framework for use with new language features
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
- Annotated namespace declaration affects only code within it declaration scope
- Support for attribute aliases using vendor/project-specific sets of attributes
- Focus on essential attributes with clear use cases
- Individual functions can override namespace code block attributes when needed

### 1.1.1 Annotated namespace declaration scope
Explanation of core concept

By "annotated namespace declaration scope" document means a namespace code block surrounded by {} see BEGIN and END comments
```cpp
[[attribute_x]]
namespace algorithms 
// BEGIN
{
    int binary_search(span<int> data, int value);  // attribute_x applied

}
// END

// Other code block of the same namespace
namespace algorithms 
{
    int debug_search(span<int> data, int value);   // attribute_x not applied

}
```


### 1.2 Core Supported Attributes
The following attributes/keywords are proposed as initialy supported that can be used as namespace attributes.

#### [[nodiscard]]
- Enforces return value usage across scope of code block within given namespace declaration
- Reduces error-prone code by preventing ignored returns
- Reduces number of common errors by forgetting to annotate with nodiscard an n-th function declaration
- Can be overridden on individual functions using [[maybe_unused]]
- Particularly valuable for error-handling and resource management

Example of [[nodiscard]] override:
```cpp
[[nodiscard]]
namespace algorithms 
{
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

#### constexpr
- Makes constexpr the default for all functions in namespace
- Non-inline functions or explicitly marked inline functions opt-out of constexpr behavior
```cpp
constexpr namespace math {
    // Implicitly constexpr due to namespace attribute
    double sqrt(double x);
    
    double sqrt(double x) { 
        return /*...*/; 
    }
    
    // Opt-out through non-inline, implementation not available
    double expensive_calc(double x);  // Not constexpr
    
    // Opt-out through explicit inline
    inline double another_calc(double x) { // Not constexpr
        return /*...*/;
    }
}
```
#### Standarized profiles from P3081 

Standarized profiles from P3081 are natural candidates for use with namespace attributes 

type_safety, bounds_safety, initialization_safety, lifetime_safety, arithmetic_safety

## 2. Attribute Consistency Model

### 2.1 Basic Declaration Rules
attributes or constexpr applied to namespace declaration affects only current scope of namespace code block
```cpp

[[nodiscard]] 
constexpr namespace math 
{
    double sqrt(double); // constexpr and nodiscard from namespace code block
}

[[deprecated]]
namespace math 
{ 
    double pow(double, double); // deprecated from namespace code block
}

namespace math 
{
    double cos(double); // default c++ rules
}
```

## 3. Nested Namespace Behavior

### 3.1 Attribute Inheritance
- Nested namespaces inherit all attributes from their parent namespace
- Multiple levels of nesting accumulate attributes from all parent levels
- Inherited attributes behave as if directly applied to the nested namespace

Example of attribute inheritance:
```cpp
[[nodiscard]] 
namespace parent 
{
    int parent_func();  // nodiscard applied
    
    namespace child // inherits nodiscard
    {   
        int child_func();  // nodiscard applied
        
        namespace grandchild // inherits nodiscard
        {  
            int grandchild_func();  // nodiscard applied
        }
    }
}
```

### 3.2 Attribute Override Rules
- Nested namespaces can override parent namespace code block attributes
- Override applies to the entire nested namespace code block and its children
- Original parent namespace code block attributes remain unchanged
- Multiple attributes are handled individually for override purposes

Example of attribute override:
```cpp
[[nodiscard]] 
namespace algorithms 
{
    int parent_search(span<int>);  // nodiscard applied
    
    [[maybe_unused]] 
    namespace debug // overrides parent nodiscard
    {  
        int debug_search(span<int>);    // maybe_unused applied
        
        namespace detail // inherits debug's maybe_unused
        {  
            int helper_func();  // maybe_unused applied
        }
    }
    
    constexpr namespace optimized // inherits parent nodiscard
    {  
        int fast_search(span<int>) {/* .. */}  // constexpr and nodiscard applied
    }
}
```

### 3.3 Inline Namespace Behavior
- Inline namespaces maintain their own attribute rules when declared
- When declarations are inlined into parent namespace, they retain their original attributes
- This allows for fine-grained control of attributes in versioned namespaces

Example of inline namespace attributes:
```cpp

[[maybe_unused]] 
namespace math 
{
    double basic_sqrt(double);  // maybe_unused applied
    
    [[nodiscard]] 
    inline namespace v2 
    {
        double advanced_sqrt(double);  // nodiscard applied
        
        namespace detail // inherits v2's nodiscard
        {  
            double helper_func();  // nodiscard applied
        }
    }
    
    // When v2 is inlined:
    // math::basic_sqrt      - maybe_unused applied
    // math::advanced_sqrt   - nodiscard applied (from v2)
    // math::detail::helper_func - nodiscard applied (from v2)
}
```

### 3.4 Multiple Attribute Interaction
```cpp
[[nodiscard]] 
constexpr namespace compute 
{
    int base_calc(int);  // constexpr and nodiscard applied
    
    [[maybe_unused]]
    namespace debug 
    {
        int debug_calc(int);  // maybe_unused only, keeps constexpr
        
        [[deprecated]]
        namespace old // combines maybe_unused and deprecated
        {  
            int legacy_calc(int);  // maybe_unused and deprecated applied
        }
    }
    
    [[deprecated]]
    inline namespace v1 
    {
        // When inlined, these keep deprecated but inherit nodiscard
        int old_calc(int);  // nodiscard and deprecated applied
    }
}
```

## 3.5 Attribute Application Rules

The behavior of namespace-level attributes is determined by the attribute type itself, not by the declarations they are applied to. This section defines how different categories of attributes behave when they encounter incompatible declarations.

### 3.5.1 Non-Enforced Attributes

These attributes can be silently ignored for incompatible declarations without generating compiler errors:

#### [[nodiscard]]
- Silently ignored for void functions/methods
- Applied to all other function/method declarations that return a value
- Applied to class/struct declarations affecting their non-void member functions

```cpp
[[nodiscard]]
namespace algorithms {
    int get_value();          // ✓ nodiscard applied
    void process_data();      // ✓ nodiscard silently ignored (void return)
    
    struct result_t {
        int query();          // ✓ nodiscard applied
        void clear();         // ✓ nodiscard silently ignored (void return)
        ~result_t();         // ✓ nodiscard silently ignored (destructor)
    };
}
```

#### [[deprecated]]
- Should be applied to any names or entities it is allowed to be applied to.

#### [[maybe_unused]]
- Should be applied to any declarations it is allowed to be applied to.

### 3.5.2 Enforced Attributes

Upcoming safety profiles should follow their use rules example enforce should enforce and cause compile error when it enforcement is violated:

#### enforce() Family
- enforce(type_safety)
- enforce(bounds_safety)
- enforce(lifetime_safety)
- enforce(arithmetic_safety)
- enforce(initialization_safety)

```cpp
using [[required::company::safety]] = [[enforce(type_safety), enforce(bounds_safety)]];

[[required::company::safety]]
namespace algorithms {
    int safe_calc(int x);     // ✓ enforces both safety rules
    
    void* unsafe_cast(void* ptr) {  // × Error: violates type safety
        return ptr;
    }
    
    template<typename T>
    T* array_access(T* arr, size_t i) {  // × Error: violates bounds safety
        return &arr[i];  
    }
}
```

### 3.5.3 Combined Attribute Behavior

When mixing enforced and non-enforced attributes, each follows its own rules:

```cpp
using [[required::company::critical]] = [[nodiscard, enforce(type_safety), enforce(lifetime_safety)]];

[[required::company::critical]]
namespace control_system {
    int get_sensor();         // ✓ all attributes applied
    void stop_motor();        // ✓ nodiscard ignored, safety rules enforced
    
    void* get_raw_ptr() {     // × Error: violates type safety
        return nullptr;       //   (enforcement failure takes precedence)
    }
    
    struct device_t {
        int query();          // ✓ all attributes applied
        void reset();         // ✓ nodiscard ignored, safety rules enforced
        ~device_t();         // ✓ nodiscard ignored, safety rules enforced
    };
}
```

### 3.5.4 Key Rules Summary

1. Attribute behavior is determined by the attribute type, not the declaration
2. Non-enforced attributes can be silently ignored when incompatible
3. Enforced attributes must cause compilation errors when violated
4. When mixing attribute types, enforcement failures take precedence
5. Inheritance of attributes in nested namespaces follows the same rules

This design ensures that:
- Safety-critical attributes cannot be accidentally bypassed
- Natural incompatibilities (like nodiscard with void) are handled gracefully
- Complex attribute combinations behave predictably
- Code maintainability is preserved through consistent rules

## 4. Backward Compatibility

### 4.1 Example of Backward Compatibility
Because all unannotated namespace code blocks retain original C++ rules it does not affect in any way existing code included into the same translation unit
```cpp


// old_header.hpp (existing code)
namespace math {
    double sqrt(double);
}

// new_feature.hpp (new code)
[[nodiscard]] 
namespace math {
    double advanced_sqrt(double);
}

// translation unit
#include <old_header.hpp>
#include <new_feature.hpp>

// advanced_sqrt will have nodiscard attribute while sqrt would retain standard c++ rules

```

## 5. Attribute Aliases

### 5.1 Alias Declaration Syntax
```cpp
// Project-specific attribute combinations
using [[required::projectx::safty_critical]] = [[nodiscard,enforce(type_safety),enforce(lifetime_safety)]];
using [[required::company::deprecated_api]] = [[deprecated("use v2 API"), maybe_unused, suppress(type_safety,"backward compatibility unsafe")]];
using [[required::team::core_math]] = [[nodiscard]];

// Usage
[[required::projectx::safty_critical]]
namespace fast_math 
{
    double quick_sqrt(double);
}

// Equivalent to:
[[nodiscard,enforce(type_safety),enforce(lifetime_safety)]]
namespace fast_math 
{
    double quick_sqrt(double);
}
```
### 5.2 Alias Rules
1. Namespace prefix must use format: `required::identifier::identifier`
3. Aliases are visible after declaration in the same translation unit
4. Aliases can combine multiple attributes
5. Aliases cannot be redefined with different attribute sets in the same translation unit

### 5.2.1 "required" prefix
- "required" prefix in alias name is proposed to be obligatory to solve problem of missing attribute alias declaration
- default rules for unknown attributes must be retained to maintain backward compatibility
- any attribute used that starts with "required" prefix and for which alias declaration is not visible should cause compile error

```cpp
// Project-specific attribute combinations
using [[required::projectx::safty_critical]] = [[nodiscard]];

// Usage
[[required::projectx::deprecated_api]] // compile error declaration not visible prior to use
namespace fast_math 
{
    double quick_sqrt(double);
}
using [[required::company::deprecated_api]] = [[deprecated("use v2 API"), maybe_unused]];

[[gnu::const]] // retain standard c++ rules for compilers not knowing it, ie ignored
namespace fast_math 
{
    double quick_sqrt(double);
}
```

## 6. Future Considerations

1. Support for modules
2. Additional core attributes
3. Cross-translation unit consistency checking
4. IDE support for attribute visualization
5. Refactoring tools for attribute management

## 7. Evolution Path

This proposal provides foundation for:

1. Future language feature adoption
2. Team-specific coding standards
3. Gradual modernization of C++ codebases
4. Enhanced static analysis capabilities

## 8. Conclusion

This focused proposal provides a practical solution for namespace-level attributes with the addition of powerful attribute aliases. The design maintains backward compatibility while enabling teams to define and enforce their own attribute policies. The cross-translation unit consistency requirements ensure well-defined behavior across the entire program.
