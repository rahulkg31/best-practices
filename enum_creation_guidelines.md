# Enum Creation Guidelines

## Purpose

Use these guidelines to keep Java enums consistent, readable, and easy to use across the project.

## 1. When to Use an Enum

Use an enum when a value represents a fixed, well-defined set of options.

Examples:

- Database types: `mongo`, `postgres`, `h2`
- Service categories: `infrastructure`, `security_and_identity`, `application`
- Status values: `active`, `inactive`
- Supported modes or strategies

For a simple enum that does not need an external string representation, keep it simple:

```java
public enum Status {
    ACTIVE,
    INACTIVE
}
```

Do not add fields, lookup maps, or methods that are not required.

## 2. Enums With External Values

When an enum value is represented externally as a string—for example in configuration, API requests, environment variables, or database values—use a dedicated `value` field.

Example:

```java
public enum DbType {
    MONGO("mongo"),
    POSTGRES("postgres"),
    H2("h2");

    private final String value;
}
```

This keeps the Java enum name separate from its external representation:

- `POSTGRES` → enum constant name
- `postgres` → external value

## 3. Naming Conventions

### Enum class

Use singular PascalCase:

```java
DbType
Category
Field
ServiceType
```

### Enum constants

Use uppercase `SNAKE_CASE`:

```java
POSTGRES
SECURITY_AND_IDENTITY
DATA_PROCESSING
PRIVATE_IP
```

### External values

Use the format required by the external contract:

```java
POSTGRES("postgres")
SECURITY_AND_IDENTITY("security_and_identity")
PRIVATE_IP("private_ip")
```

## 4. Recommended Standard Pattern

For enums with external string values, the standard pattern should normally contain:

1. Enum constants and their external values
2. Static lookup map
3. Instance `value` field
4. Static initialization block
5. Constructor
6. `getName()`
7. `getValue()`
8. `fromValue()`
9. `isValid()`

### Standard Example

```java
import java.util.HashMap;
import java.util.Map;

public enum SomeType {

    FIRST("first"),
    SECOND("second"),
    THIRD("third");

    private static final Map<String, SomeType> VALUE_TO_TYPE = new HashMap<>();

    private final String value;

    static {
        for (SomeType type : SomeType.values()) {
            VALUE_TO_TYPE.put(type.getValue(), type);
        }
    }

    SomeType(String value) {
        this.value = value;
    }

    public String getName() {
        return name();
    }

    public String getValue() {
        return value;
    }

    public static SomeType fromValue(String value) {
        if (value == null) {
            return null;
        }

        return VALUE_TO_TYPE.get(value.toLowerCase());
    }

    public static boolean isValid(String value) {
        return fromValue(value) != null;
    }
}
```

## 5. Method Guidelines

### `getName()`

Returns the Java enum constant name.

```java
SomeType.FIRST.getName();
```

Result:

```text
FIRST
```

This is a convenient wrapper around Java's built-in `Enum.name()` method.

### `getValue()`

Returns the external string representation.

```java
SomeType.FIRST.getValue();
```

Result:

```text
first
```

Use this when interacting with configuration, APIs, persistence, or other external representations.

### `fromValue()`

Converts an external string into the corresponding enum constant.

```java
SomeType type = SomeType.fromValue("first");
```

Result:

```text
SomeType.FIRST
```

It should normally return `null` when the supplied value is `null` or unsupported.

### `isValid()`

Checks whether a value represents a supported enum value.

```java
if (SomeType.isValid(input)) {
    // Valid value
}
```

## 6. Do Not Force the Full Pattern

The standard pattern should not be applied to every enum automatically.

For a simple internal enum:

```java
public enum Status {
    ACTIVE,
    INACTIVE
}
```

there is no need for `value`, `VALUE_TO_TYPE`, `getValue()`, `fromValue()`, or `isValid()` unless they provide an actual requirement.

