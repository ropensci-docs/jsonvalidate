# Validate a json file

Validate a single json against a schema. This is a convenience wrapper
around `json_validator(schema)(json)` or
`json_schema$new(schema, engine = "ajv")$validate(json)`. See
[`json_validator()`](https://docs.ropensci.org/jsonvalidate/reference/json_validator.md)
for further details.

## Usage

``` r
json_validate(
  json,
  schema,
  verbose = FALSE,
  greedy = FALSE,
  error = FALSE,
  engine = "imjv",
  reference = NULL,
  query = NULL,
  strict = FALSE
)
```

## Arguments

- json:

  Contents of a json object, or a filename containing one.

- schema:

  Contents of the json schema, or a filename containing a schema.

- verbose:

  Be verbose? If `TRUE`, then an attribute "errors" will list validation
  failures as a data.frame

- greedy:

  Continue after the first error?

- error:

  Throw an error on parse failure? If `TRUE`, then the function returns
  `NULL` on success (i.e., call only for the side-effect of an error on
  failure, like `stopifnot`).

- engine:

  Specify the validation engine to use. Options are "imjv" (the default;
  which uses "is-my-json-valid") and "ajv" (Another JSON Schema
  Validator). The latter supports more recent json schema features.

- reference:

  Reference within schema to use for validating against a sub-schema
  instead of the full schema passed in. For example if the schema has a
  'definitions' list including a definition for a 'Hello' object, one
  could pass "#/definitions/Hello" and the validator would check that
  the json is a valid "Hello" object. Only available if
  `engine = "ajv"`.

- query:

  A string indicating a component of the data to validate the schema
  against. Eventually this may support full
  [jsonpath](https://www.npmjs.com/package/jsonpath) syntax, but for now
  this must be the name of an element within `json`. See the examples
  for more details.

- strict:

  Set whether the schema should be parsed strictly or not. If in strict
  mode schemas will error to "prevent any unexpected behaviours or
  silently ignored mistakes in user schema". For example it will error
  if encounters unknown formats or unknown keywords. See
  https://ajv.js.org/strict-mode.html for details. Only available in
  `engine = "ajv"`.

## Examples

``` r
# A simple schema example:
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

# Test if some (invalid) json conforms to the schema
jsonvalidate::json_validate("{}", schema, verbose = TRUE)
#> [1] FALSE
#> attr(,"errors")
#>        field     message
#> 1    data.id is required
#> 2  data.name is required
#> 3 data.price is required

# Test if some (valid) json conforms to the schema
json <- '{
    "id": 1,
    "name": "A green door",
    "price": 12.50,
    "tags": ["home", "green"]
}'
jsonvalidate::json_validate(json, schema)
#> [1] TRUE

# Test a fraction of a data against a reference into the schema:
jsonvalidate::json_validate(json, schema,
                            query = "tags", reference = "#/properties/tags",
                            engine = "ajv", verbose = TRUE)
#> [1] TRUE
```
