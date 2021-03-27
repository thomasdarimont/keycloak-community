# Validation SPI

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-2045](https://issues.redhat.com/browse/KEYCLOAK-2045)

## Motivation

Validation is currently scattered around several places and implemented in different ways 
which makes it very hard to reuse validation logic in a consistent way. 
Besides some static helper functions that are used in different places, there are also many 
component-specific validations that are directly wired to the surrounding business logic.
A unified and consistent validation mechanism is currently missing.

There have been several approaches to establish a validation infrastructure, but they have 
not been followed consistently.

The goal of this proposal is to provide an extensible and unified validation SPI 
that can be used across the Keycloak code-base. The need for a new validation API was highlighted 
by the design discussions for the [user-profile](https://github.com/keycloak/keycloak-community/blob/master/design/user-profile.md) support.

The goals of this proposal can be summarized as:

* Provide a uniform and extensible validation SPI
* Consolidate the scattered validation logic with the new Validation SPI

## Design Principles

The design of the new validation SPI should follow the following design principles:

- Simplicity
The API should have a small API surface and be easy to use.

- API Granularity
The API should support low-level validations.

- Efficiency
Validations should be fast and have little overhead.

- Extensibility
There should be support for built-in as well as user provided validations.
The new SPI should be easy to extend. Developers should be able to provide reusable 
custom validation logic components.

- Versatility
Validations should be flexible and offer support for parameterization. 
There should be a way to check if the parameterization for a validation is valid.
Validations should be composable. It should be possible to access other validations
from with a validation. It should be possible to validate simple values according as well as complex objects.

## Considered Options

### Bean Validation API

The JSR 380 Bean Validation 2.0 is a mature and rich validation framework that is extensively used in Java applications.

- Optimized for validation of complex Java bean objects based on declarative configuration rather than single values.

Allthough, value validations are possible via `javax.validation.Validator#validateValue`, they require a bean class 
with a property with the annotated constraints.
```java
	<T> Set<ConstraintViolation<T>> Validator#validateValue(Class<T> beanType,
												  String propertyName,
												  Object value,
												  Class<?>... groups);
```
[Example for validating a value via Validator#validateValue](https://gist.github.com/thomasdarimont/e6e375819b3b1adff86b7669e5b033b4).

- hibernate-validator is currently not used in Keycloak, thus we'd need to add a dependency for:
```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.12.Final</version>
</dependency>
```
- No direct support for validation (Constraint) lookups by name. The user-profile support needs a way to lookup validations by name.
- No easy way to dynamically configure constraints programatically. The user-profile support needs a way to configure validation dynamically.
- Definition of [custom validations is quite involved](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-customconstraints) - you need 2 classes (annotation + constraint) + error message + META-INF service manifest.

All this was considered and evaluated in a small prototype, but then it was concluded that it makes more sense to go 
with a custom implementation that provides a narrow API. Additionally custom extensions should be easier and provide 
a better integration with the existing Keycloak infrastructure. Further more dynamic validation logic lookups 
and configuration should be possible and easy to do. 

Note that it is still possible to provide a validation/validator implementation that leverages bean validation. 
Also, bean-validation still works quite well for declarative validation in JAX-RS endpoints.

### Fully fledged Validation SPI

An earlier attempt at a unified validation SPI can be found here [KEYCLOAK-2045 Add support for flexible Validation](https://github.com/keycloak/keycloak/pull/7324) and was discussed on the [keycloak-dev mailing list](https://groups.google.com/g/keycloak-dev/c/-XjQu7rn56s).


## Proposed Validation SPI

To provide an simple and easy to use API the following types are defined:

### Core API
The proposal focus around the following types:
1. `Validator`: `Provider` interface for validation mechanics.
1. `ValidationContext`: Holds state of a sequence of validations.
1. `ValidationResult` Denotes the outcome of a validation.
1. `ValidationError` Represents an error found during validation.
1. `ValidatorLookup`: Helper class to provide lookups for built-in and user provided validations.

### Support API
The following types provide the integration with the Keycloak infrastructure:
1. `ValidatorFactory`: `ProviderFactory` interface for custom validation contributions.
1. `ValidatorSPI`: Provides the `validator` SPI.
1. `CompactValidator`: Convenience class for built-in validations.
1. `BuiltinValidators`: Denotes a registry for internal built-in validations.

The proposed package name is `org.keycloak.validation`. Note the current PR [KEYCLOAK-2045 Simple Validation API](https://github.com/keycloak/keycloak/pull/7887) uses the package name `org.keycloak.validate` to provide a transition path for the current API.

### Module location
Initially, the package `org.keycloak.validation` is treated as an internal API, so the component 
is provided by the module `server-spi-private`. Once the API is declared as a public API, the 
package can be moved to the `server-private` module.

### Validator interface

The proposed Validator interface.

```java
public interface Validator extends Provider {
    /**
     * Validates the given {@code input} with an additional {@code inputHint} and {@code config}.
     *
     * @param input     the value to validate
     * @param inputHint an optional input hint to guide the validation
     * @param context   the validation context
     * @param config    parameterization for the current validation
     * @return the validation context with the outcome of the validation
     */
    ValidationContext validate(Object input, String inputHint, ValidationContext context, Map<String, Object> config);

    // convenience methods, which all delegate to the validate(...) method above
    default ValidationContext validate(Object input) {...}
    default ValidationContext validate(Object input, ValidationContext context) {...}
    default ValidationContext validate(Object input, String inputHint, ValidationContext context) { ... }

    /**
     * Validates the given validation config.
     *
     * @param config the config to be validated
     * @return the validation result
     */
    default ValidationResult validateConfig(Map<String, Object> config) {
        return ValidationResult.OK;
    }
}
```

### Adding new Validator

New built-in `Validator`'s can implement the provided `CompactValidator` interface.
A `CompactValidator` needs to provide the following two methods:
1. `String getId()` 
1. `ValidationContext validate(Object input, String inputHint, ValidationContext context, Map<String, Object> config)`

Depending on the requirements the method `ValidationResult validateConfig(Map<String, Object> config)` can be implemented
to support validation of config values that are used to parameterize the validation.

Users can provide their own validations via the `provider` SPI by implementing the `CompactValidator` interface as a singleton 
class or by creating two separate classes by implementing `Validator` and `ValidatorFactory` respectively. 
In either case, users need to register the new validator via Keycloaks SPI mechanism.

### PoC for new Validator SPI

The Proof of concept implementation can be found here in the PR [KEYCLOAK-2045 Simple Validation API #7887](https://github.com/keycloak/keycloak/pull/7887).

Some usage examples can be found here in the [ValidatorTest](https://github.com/thomasdarimont/keycloak/blob/issue/KEYCLOAK-2045-Simple-Validation-SPI/server-spi-private/src/test/java/org/keycloak/validate/ValidatorTest.java).