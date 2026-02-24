---
name: mapstruct-antipattern-mitigation-with-unit-test
description: Describe how to make Unit Tests for insegure mappers that uses antipattern Setter-Based Mapping
license: Apache-2.0
metadata:
  author: anibalanto
  version: "1.0"
---

# mapstruct-antipattern-mitigation-with-unit-test

This is a technical standard for testing many existing mappers. The objective is to ensure that following any migration or changes to `Source` or `Target` classes, data transfer remains consistent and no unexpected null values are introduced.

## 🧪 Implementation Standard

To ensure the test is pure and decoupled from POJO state, you must **mock the Source object behavior**. This forces the developer to explicitly define which fields the mapper is expected to read.

### Code Template (Java + JUnit 5 + Mockito)

```java
@ExtendWith(MockitoExtension.class)
class UserMapperTest {

    private final UserMapper mapper = Mappers.getMapper(UserMapper.class);

    @Test
    void shouldMapSourceToTargetExhaustively() {
        // 1. Arrange: Source Mocking
        // We create a mock to define exactly what the "source" provides
        UserSource sourceMock = mock(UserSource.class);
        
        when(sourceMock.getFullName()).thenReturn("John Doe");
        when(sourceMock.getEmail()).thenReturn("john@example.com");
        when(sourceMock.getAge()).thenReturn(30);

        // 2. Act: Execute mapping logic
        UserTarget target = mapper.toTarget(sourceMock);

        // 3. Assert: Exhaustive field-by-field verification
        // Validating the whole object is forbidden; each attribute must be verified individually
        assertAll("Mapping Integrity Verification",
            // Validate @Mapping(source = "fullName", target = "name")
            () -> assertEquals(sourceMock.getFullName(), target.getName(), "Mapping error: name field"),
            
            // Validate implicit/identical name fields
            () -> assertEquals(sourceMock.getEmail(), target.getEmail(), "Mapping error: email field"),
            
            // Validate default values or constants defined in the mapper
            () -> assertEquals("ACTIVE", target.getStatus(), "Status must be ACTIVE per mapper config"),
            
            // Ensure no unexpected nulls exist in the target after migration
            () -> assertNotNull(target.getCreatedAt(), "Timestamp field must not be null")
        );
    }
}

```

---

## 🚨 Golden Rules (Checklist)

* **Mandatory Mocking:** The `Source` object **must** be a mock. This ensures the test validates the mapper's interaction with the source API.
* **Individual Assertions:** Using `assertEquals(expectedObj, actualObj)` is prohibited. Use individual `assertEquals` for every field in the `Target`.
* **Constant Validation:** If a mapper defines `@Mapping(target = "x", constant = "y")`, the test must verify that "y" reaches the target regardless of the source.
* **Zero-Null Policy:** If a `Target` field is mandatory post-migration, the test must fail if the mapper is not explicitly populating it.

---

## When to use

This skill should be used when exists previously a lot of mappers using antipattern Setter-Based Mapping and is not possible migrate each mapper consistently.

## Instructions

1. **Field Reflection:** List all fields in the `Target` class.
2. **Origin Identification:**
* **Direct Match:** Fields with identical names in `Source` and `Target`.
* **@Mapping:** Fields requiring explicit transformation or name changes.
* **Default/Constant:** Fields populated by the mapper's internal configuration (not dependent on the `Source`).
