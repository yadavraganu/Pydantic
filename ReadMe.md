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
