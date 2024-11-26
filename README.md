# C++ Namespace Attributes & Attribute Aliases Proposal
## Core Attributes & Custom Attribute Aliases for Namespaces

## Motivation & Impact

This proposal introduces two key features to C++:
1. Ability to apply attributes to namespaces, affecting all contained declarations whitin given block of namespace declaration
2. Mechanism to define aliases for sets of attributes, enabling consistent attribute policies

These additions provide powerful tools for code organization, maintenance, and evolution while maintaining complete backward compatibility.

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

[[type_safety]], [[bounds_safety]],[[initialization_safety]],[[lifetime_safety]],[[arithmetic_safety]]
with all [[suppress(X)]]  variants

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
    int base_calc(int);  // both attributes applied
    
    [[maybe_unused]]
    namespace debug 
    {
        int debug_calc(int);  // maybe_unused only, keeps constexpr_default
        
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
using [[required::projectx::safty_critical]] = [[nodiscard,type_safety,lifetime_safety]];
using [[required::company::deprecated_api]] = [[deprecated("use v2 API"), maybe_unused, suppress(type_safety)]];
using [[required::team::core_math]] = [[nodiscard]];

// Usage
[[required::projectx::safty_critical]]
namespace fast_math 
{
    double quick_sqrt(double);
}

// Equivalent to:
[[nodiscard]]
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

// Equivalent to:
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
