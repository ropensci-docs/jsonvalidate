# Safe JSON serialisation

Safe serialisation of json with unboxing guided by the schema.

## Usage

``` r
json_serialise(
  object,
  schema,
  engine = "ajv",
  reference = NULL,
  strict = FALSE
)
```

## Arguments

- object:

  An object to be serialised

- schema:

  A schema (string or path to a string, suitable to be passed through to
  [json_validator](https://docs.ropensci.org/jsonvalidate/reference/json_validator.md)
  or a validator object itself.

- engine:

  The engine to use. Only ajv is supported, and trying to use `imjv`
  will throw an error.

- reference:

  Reference within schema to use for validating against a sub-schema
  instead of the full schema passed in. For example if the schema has a
  'definitions' list including a definition for a 'Hello' object, one
  could pass "#/definitions/Hello" and the validator would check that
  the json is a valid "Hello" object. Only available if
  `engine = "ajv"`.

- strict:

  Set whether the schema should be parsed strictly or not. If in strict
  mode schemas will error to "prevent any unexpected behaviours or
  silently ignored mistakes in user schema". For example it will error
  if encounters unknown formats or unknown keywords. See
  https://ajv.js.org/strict-mode.html for details. Only available in
  `engine = "ajv"`.

## Value

A string, representing `object` in JSON format. As for
[`jsonlite::toJSON`](https://jeroen.r-universe.dev/jsonlite/reference/fromJSON.html)
we set the class attribute to be `json` to mark it as serialised json.

## Details

When using
[jsonlite::toJSON](https://jeroen.r-universe.dev/jsonlite/reference/fromJSON.html)
we are forced to deal with the differences between R's types and those
available in JSON. In particular:

- R has no scalar types so it is not clear if `1` should be serialised
  as a number or a vector of length 1; `jsonlite` provides support for
  "automatically unboxing" such values (assuming that length-1 vectors
  are scalars) or never unboxing them unless asked to using
  [jsonlite::unbox](https://jeroen.r-universe.dev/jsonlite/reference/unbox.html)

- JSON has no date/time values and there are many possible string
  representations.

- JSON has no [data.frame](https://rdrr.io/r/base/data.frame.html) or
  [matrix](https://rdrr.io/r/base/matrix.html) type and there are
  several ways of representing these in JSON, all equally valid (e.g.,
  row-wise, column-wise or as an array of objects).

- The handling of `NULL` and missing values (`NA`, `NaN`) are different

- We need to chose the number of digits to write numbers out at,
  balancing precision and storage.

These issues are somewhat lessened when we have a schema because we know
what our target type looks like. This function attempts to use the
schema to guide serialisation of json safely. Currently it only supports
detecting the appropriate treatment of length-1 vectors, but we will
expand functionality over time.

For a user, this function provides an argument-free replacement for
[`jsonlite::toJSON`](https://jeroen.r-universe.dev/jsonlite/reference/fromJSON.html),
accepting an R object and returning a string with the JSON
representation of the object. Internally the algorithm is:

1.  serialise the object with
    [jsonlite::toJSON](https://jeroen.r-universe.dev/jsonlite/reference/fromJSON.html),
    with `auto_unbox = FALSE` so that length-1 vectors are serialised as
    a length-1 arrays.

2.  operating entirely within JavaScript, deserialise the object with
    `JSON.parse`, traverse the object and its schema simultaneously
    looking for length-1 arrays where the schema says there should be
    scalar value and unboxing these, and re-serialise with
    `JSON.stringify`

There are several limitations to our current approach, and not all
unboxable values will be found - at the moment we know that schemas
contained within a `oneOf` block (or similar) will not be recursed into.

## Warning

Direct use of this function will be slow! If you are going to serialise
more than one or two objects with a single schema, you should use the
`serialise` method of a
[json_schema](https://docs.ropensci.org/jsonvalidate/reference/json_schema.md)
object which you create once and pass around.

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
