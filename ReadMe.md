### Creating a Pydantic Model

* **Concept**: Inherit from `BaseModel` and use type hints to define the schema.
* **Key Detail**: Pydantic models are standard classes; you can add custom methods.
* **Post-Instantiation**: By default, Pydantic does not re-validate if you manually overwrite an attribute after the object is created.

```python
from pydantic import BaseModel

class Person(BaseModel):
    first_name: str
    last_name: str
    age: int

    def greet(self):
        return f"Hello, {self.first_name}!"

p = Person(first_name="Isaac", last_name="Newton", age=84)
p.age = "eighty-four"  # Standard Pydantic won't stop this manual override yet

```

### Deserialization & Serialization

* **Deserialization**: Turning data (dict/JSON) into an object.
* **Serialization**: Turning an object into data.

```python
# Deserialization
data_dict = {"first_name": "Isaac", "last_name": "Newton", "age": 84}
p = Person.model_validate(data_dict)
p_json = Person.model_validate_json('{"first_name": "Isaac", "last_name": "Newton", "age": 84}')

# Serialization
p.model_dump() # Returns dict
p.model_dump_json() # Returns string

# Filtering during export
p.model_dump(include={"last_name"}) # {'last_name': 'Newton'}
p.model_dump(exclude={"age"})       # {'first_name': 'Isaac', 'last_name': 'Newton'}

```

### Type Coercion

* **Logic**: Pydantic attempts to convert types to match the schema.
* **Safe Conversions**: String `"123"` to `int`, or float `1.0` to `int`.
* **Strictness**: It won't coerce complex objects (like a dict) into a string just because it's possible; it enforces data integrity.

```python
class Model(BaseModel):
    age: int

# Pydantic coerces these to 10
m1 = Model(age="10")
m2 = Model(age=10.0)

```

### Required vs. Optional Fields

* **Required**: No default value.
* **Optional**: Has a default value.
* **Mutable Defaults**: Pydantic handles lists/dicts safely by creating a **deep copy** for every new instance (unlike standard Python functions).

```python
class Survey(BaseModel):
    results: list[int] = [] # SAFE: each instance gets a fresh list

```

### Nullable Fields & Combinations

The distinction between a "missing key" (Optional) and a "null value" (Nullable) is essential.

```python
# 1. Required, Not Nullable (Must provide a non-null value)
field: int 

# 2. Required, Nullable (Must provide a value, but it can be None)
field: int | None 

# 3. Optional, Not Nullable (Missing key defaults to 10; if provided, must be int)
field: int = 10 

# 4. Optional, Nullable (Missing key defaults to None)
field: int | None = None 

```

### Inspecting Fields

* **`model_fields`**: Returns metadata (type, default, etc.) for all fields.
* **`model_fields_set`**: Returns a set of fields that were **actually passed** by the user.

```python
class Model(BaseModel):
    a: int = 1
    b: int = 2

m = Model(a=5)
print(m.model_fields_set) # {'a'}
# Dump only what the user provided:
m.model_dump(include=m.model_fields_set) # {'a': 5}

```

### JSON Schema Generation

* **Purpose**: Used for generating API documentation (OpenAPI).

```python
from pprint import pprint

class Model(BaseModel):
    name: str
    age: int | None = None

pprint(Model.model_json_schema())

```
### Handling Extra Fields

* **Core Concept:** Dictates how Pydantic handles fields present in the input data that are not defined on the model.
* **Default Behavior:** Extra fields are ignored by default. They will not throw an error but won't be accessible on the instance.
* **Configuration Options (`extra`):** `'ignore'`, `'allow'`, or `'forbid'`.

```python
from pydantic import BaseModel, ConfigDict, ValidationError

# Case 1: Forbid Extra Fields
class StrictModel(BaseModel):
    model_config = ConfigDict(extra='forbid')
    name: str

try:
    StrictModel(name="Alice", age=30)  # 'age' is extra
except ValidationError as e:
    print(e)  # Raises Extra attributes not allowed

# Case 2: Allow Extra Fields
class FlexibleModel(BaseModel):
    model_config = ConfigDict(extra='allow')
    name: str

m = FlexibleModel(name="Bob", age=30)
print(m.model_extra)  # {'age': 30}

```
### Strict and Lax Type Coercion

* **Core Concept:** Controls whether Pydantic attempts to cast incompatible data types into the target type.
* **Lax Mode (Default):** Coerces compatible values (e.g., string `"123"` to an integer `123`).
* **Strict Mode (`strict=True`):** Disables type casting. Input types must precisely match annotations.

```python
# Lax Mode (Default)
class LaxModel(BaseModel):
    age: int

print(LaxModel(age="42"))  # Output: age=42 (coerced)

# Strict Mode
class StrictTypeModel(BaseModel):
    model_config = ConfigDict(strict=True)
    age: int

try:
    StrictTypeModel(age="42")
except ValidationError as e:
    print(e)  # Input should be a valid integer

```
### Validating Default Values

* **Core Concept:** Determines if default values are verified against validation checks upon model instantiation.
* **Default Behavior:** By default, Pydantic assumes code defaults are valid and does not check them at instantiation.

```python
# Default Behavior (No validation on default)
class BadDefaultModel(BaseModel):
    score: int = "not an int"  # Pydantic lets this slide on instantiation!

m = BadDefaultModel()
print(type(m.score))  # <class 'str'> -> Breaks your type hint!

# Enforcing Default Validation
class ValidatedDefaultModel(BaseModel):
    model_config = ConfigDict(validate_default=True)
    score: int = "not an int"

try:
    ValidatedDefaultModel()
except ValidationError as e:
    print(e)  # Input should be a valid integer

```

### Validating Assignments

* **Core Concept:** Validates data when attributes are updated *after* the model instance has already been built.
* **Default Behavior:** Mutating attributes post-creation bypasses validation by default.

```python
class AssignmentModel(BaseModel):
    model_config = ConfigDict(validate_assignment=True)
    price: float

m = AssignmentModel(price=19.99)
try:
    m.price = "Free"  # Triggers real-time validation
except ValidationError as e:
    print(e)  # Input should be a valid number

```
### Mutability

* **Core Concept:** Controls whether the model can be changed after construction, allowing objects to become hashable.
* **Default Behavior:** Models are completely mutable.

```python
class FrozenModel(BaseModel):
    model_config = ConfigDict(frozen=True)
    user_id: int

m = FrozenModel(user_id=101)

try:
    m.user_id = 102  # Raises ValidationError: Instance is frozen
except ValidationError as e:
    print(e)

# Frozen models can be used in sets or dict keys!
my_set = {m}
print(my_set)  # Workable because it generates a __hash__ method

```
### Coercing Numbers to Strings

* **Core Concept:** Specifically overrides how numerical types match when a string field annotation is expected.

```python
class NumberToStrModel(BaseModel):
    model_config = ConfigDict(coerce_numbers_to_str=True)
    username: str

m = NumberToStrModel(username=12345)
print(m.username)  # Output: '12345' (Cast directly to string)

```

### Standardizing Strings

* **Core Concept:** Automates typical string sanitization logic (stripping whitespace, modifying case) right inside the model configuration.

```python
class CleanStringModel(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        str_to_upper=True
    )
    code: str

m = CleanStringModel(code="   amzn \t ")
print(f"'{m.code}'")  # Output: 'AMZN'

```

### Handling Python Enums

* **Core Concept:** Manages how Python native `Enum` objects are handled during serialization operations like `.model_dump()`.

```python
from enum import Enum

class Status(Enum):
    ACTIVE = "active_user"
    PENDING = "pending_user"

class UserModel(BaseModel):
    model_config = ConfigDict(use_enum_values=True)
    status: Status

u = UserModel(status=Status.ACTIVE)
print(u.status)  # Returns the Enum Object: Status.ACTIVE

# Notice the difference during dictionary export:
print(u.model_dump())  # Output: {'status': 'active_user'} (Raw primitive value)

```
