### Field Aliases and Default Values

#### Concepts:
* Type hints and standard defaults are sometimes insufficient when mapping complex external structural conventions to a Pythonic convention.
* The `Field` object allows advanced metadata configuration for attributes on a `BaseModel`.
* Standard aliases specified using `Field(alias="...")` force the input configuration to strictly expect that alias variant during deserialization/validation.
* Providing a default value alongside a `Field` configurations must be done explicitly using the `default` keyword argument inside the `Field` instance.
* By default serialization happens using original names not the deserialization

#### Code Implementation:
```python
from pydantic import BaseModel, Field, ValidationError

class Model(BaseModel):
    id_: int = Field(alias="id")
    last_name: str = Field(alias="lastName")

# Deserializing via valid JSON containing expected aliases
json_data = '{"id": 100, "lastName": "Gauss"}'
m = Model.model_validate_json(json_data)
print(m)  # Output: Model(id_=100, last_name='Gauss')

```

### Alias Generator Functions

#### Concepts:
* Instead of configuring aliases one field at a time across massive models, programmatic naming conversions can be mapped dynamically.
* Pydantic provides pre-baked case converters (`to_camel`, `to_snake`, `to_pascal`) inside `pydantic.alias_generators`.
* Custom programmatic transformations can be leveraged by registering any explicit string formatting function into the `ConfigDict(alias_generator=...)` setup.


#### Code Implementation:
```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

# Example using a custom generator function
def make_upper(in_str: str) -> str:
    return in_str.upper()

class UpperModel(BaseModel):
    model_config = ConfigDict(alias_generator=make_upper)
    user_id: int
    first_name: str

# The auto-generated aliases can be viewed via model_fields
print(UpperModel.model_fields.keys()) 

```

### Deserializing by Field Name or Alias

#### Concepts:
* By default, assigning an `alias` locks down validation so that only the defined alias is valid during data loading.
* To accept both Pythonic field names and structured external aliases seamlessly during deserialization, use `ConfigDict(populate_by_name=True)`.


#### Code Implementation:
```python
from pydantic import BaseModel, ConfigDict, Field

class FlexibleModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    id_: int = Field(alias="id")
    first_name: str = Field(alias="firstName")

# Now both input methods are completely valid
m1 = FlexibleModel(id_=10, first_name="Newton")
m2 = FlexibleModel(id=10, firstName="Newton")

# Export control via by_alias parameter
print(m1.model_dump(by_alias=True))
print(m1.model_dump_json(by_alias=True))

```

### Serialization Aliases

#### Concepts:
* Used when ingestion specifications vary wildly from outward response rules (e.g., extracting from messy source payloads while responding with a clean standard schema).
* `alias` functions as a catch-all fallback configuration, whereas `serialization_alias` overrides how data is structured outward when `model_dump(by_alias=True)` or `model_dump_json(by_alias=True)` is fired.


#### Code Implementation:
```python
from pydantic import BaseModel, Field

class OutboundCleanModel(BaseModel):
    id_: int = Field(alias="ID", serialization_alias="id")
    first_name: str = Field(alias="FirstName", serialization_alias="firstName")

# Deserializes using original messy target names
m = OutboundCleanModel(ID=100, FirstName="Isaac")

# Serializes into cleaner modern formatting
print(m.model_dump(by_alias=True))  # Output: {'id': 100, 'firstName': 'Isaac'}

```

### Validation Aliases

#### Concepts:
* `validation_alias` functions strictly as an ingestion override mechanism.
* Think of generic `alias` as a dual-action fallback configuration, whereas `validation_alias` and `serialization_alias` function as specialized fine-grained controls.
* Complex rules can take lists or structures to validate against several potential structural formats (e.g., tracking a value that can appear in multiple locations across different versions of an API).


* **Code Implementation**:
```python
from pydantic import BaseModel, Field, ConfigDict

class InboundOverrideModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
    first_name: str = Field(validation_alias="FirstName", alias="firstName")

# Deserializes via validation_alias
m = InboundOverrideModel.model_validate({"FirstName": "Isaac"})

# Serialization drops back to standard field naming or plain 'alias'
print(m.model_dump(by_alias=True))  # Output uses 'firstName'

```
### Custom Serializers

#### Concepts:
* Used to explicitly manage how field variables transition into serialized targets (e.g., trimming float outputs, customizing date string patterns).
* Implemented using the `@field_serializer` method decorator.
* Execution rules can be scoped cleanly using the `when_used` argument:
* `always`: Executes during Python `dict` extraction as well as full JSON dumps.
* `unless-none`: Skips formatting pipeline completely if the value maps as `None`.
* `json`: Only processes transformation explicitly during standard string JSON dumps.
* `json-unless-none`: Only runs during string JSON transformation steps provided the target is active.


#### Code Implementation:
```python
from pydantic import BaseModel, field_serializer
from datetime import datetime

class TimedModel(BaseModel):
    dt: datetime | None = None

    @field_serializer("dt", when_used="always")
    def serialize_datetime(self, value):
        if isinstance(value, datetime):
            return value.strftime("%Y-%m-%d %H:%M:%S")
        return value

m = TimedModel(dt="2026-05-25T12:00:00")
print(m.model_dump())  # Output: {'dt': '2026-05-25 12:00:00'}

```
