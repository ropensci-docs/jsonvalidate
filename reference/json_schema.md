# Interact with JSON schemas

Interact with JSON schemas, using them to validate json strings or
serialise objects to JSON safely.

This interface supersedes json_schema and changes some default
arguments. While the old interface is not going away any time soon,
users are encouraged to switch to this interface, which is what we will
develop in the future.

## Public fields

- `schema`:

  The parsed schema, cannot be rebound

- `engine`:

  The name of the schema validation engine

## Methods

### Public methods

- [`json_schema$new()`](#method-json_schema-new)

- [`json_schema$validate()`](#method-json_schema-validate)

- [`json_schema$serialise()`](#method-json_schema-serialise)

------------------------------------------------------------------------

### Method `new()`

Create a new `json_schema` object.

#### Usage

    json_schema$new(schema, engine = "ajv", reference = NULL, strict = FALSE)

#### Arguments

- `schema`:

  Contents of the json schema, or a filename containing a schema.

- `engine`:

  Specify the validation engine to use. Options are "ajv" (the default;
  "Another JSON Schema Validator") or "imjv" ("is-my-json-valid", the
  default everywhere in versions prior to 1.4.0, and the default for
  [json_validator](https://docs.ropensci.org/jsonvalidate/reference/json_validator.md).
  *Use of `ajv` is strongly recommended for all new code*.

- `reference`:

  Reference within schema to use for validating against a sub-schema
  instead of the full schema passed in. For example if the schema has a
  'definitions' list including a definition for a 'Hello' object, one
  could pass "#/definitions/Hello" and the validator would check that
  the json is a valid "Hello" object. Only available if
  `engine = "ajv"`.

- `strict`:

  Set whether the schema should be parsed strictly or not. If in strict
  mode schemas will error to "prevent any unexpected behaviours or
  silently ignored mistakes in user schema". For example it will error
  if encounters unknown formats or unknown keywords. See
  https://ajv.js.org/strict-mode.html for details. Only available in
  `engine = "ajv"` and silently ignored for "imjv". Validate a json
  string against a schema.

------------------------------------------------------------------------

### Method `validate()`

#### Usage

    json_schema$validate(
      json,
      verbose = FALSE,
      greedy = FALSE,
      error = FALSE,
      query = NULL
    )

#### Arguments

- `json`:

  Contents of a json object, or a filename containing one.

- `verbose`:

  Be verbose? If `TRUE`, then an attribute "errors" will list validation
  failures as a data.frame

- `greedy`:

  Continue after the first error?

- `error`:

  Throw an error on parse failure? If `TRUE`, then the function returns
  `NULL` on success (i.e., call only for the side-effect of an error on
  failure, like `stopifnot`).

- `query`:

  A string indicating a component of the data to validate the schema
  against. Eventually this may support full
  [jsonpath](https://www.npmjs.com/package/jsonpath) syntax, but for now
  this must be the name of an element within `json`. See the examples
  for more details. Serialise an R object to JSON with unboxing guided
  by the schema. See
  [json_serialise](https://docs.ropensci.org/jsonvalidate/reference/json_serialise.md)
  for details on the problem and the algorithm.

------------------------------------------------------------------------

### Method `serialise()`

#### Usage

    json_schema$serialise(object)

#### Arguments

- `object`:

  An R object to serialise

## Examples

``` r
# This is the schema from ?json_validator
schema <- '{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "Product",
    "description": "A product from Acme\'s catalog",
    "type": "object",
    "properties": {
        "id": {
            "description": "The unique identifier for a product",
            "type": "integer"
        },
        "name": {
            "description": "Name of the product",
            "type": "string"
        },
        "price": {
            "type": "number",
            "minimum": 0,
            "exclusiveMinimum": true
        },
        "tags": {
            "type": "array",
            "items": {
                "type": "string"
            },
            "minItems": 1,
            "uniqueItems": true
        }
    },
    "required": ["id", "name", "price"]
}'

# We're going to use a validator object below
v <- jsonvalidate::json_validator(schema, "ajv")

# And this is some data that we might generate in R that we want to
# serialise using that schema
x <- list(id = 1, name = "apple", price = 0.50, tags = "fruit")

# If we serialise to json, then 'id', 'name' and "price' end up a
# length 1-arrays
jsonlite::toJSON(x)
#> {"id":[1],"name":["apple"],"price":[0.5],"tags":["fruit"]} 

# ...and that fails validation
v(jsonlite::toJSON(x))
#> [1] FALSE

# If we auto-unbox then 'fruit' ends up as a string and not an array,
# also failing validation:
jsonlite::toJSON(x, auto_unbox = TRUE)
#> {"id":1,"name":"apple","price":0.5,"tags":"fruit"} 
v(jsonlite::toJSON(x, auto_unbox = TRUE))
#> [1] FALSE

# Using json_serialise we can guide the serialisation process using
# the schema:
jsonvalidate::json_serialise(x, schema)
#> {"id":1,"name":"apple","price":0.5,"tags":["fruit"]} 

# ...and this way we do pass validation:
v(jsonvalidate::json_serialise(x, schema))
#> [1] TRUE

# It is typically much more efficient to construct a json_schema
# object first and do both operations with it:
obj <- jsonvalidate::json_schema$new(schema)
json <- obj$serialise(x)
obj$validate(json)
#> [1] TRUE
```
