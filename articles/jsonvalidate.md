# Introduction to jsonvalidate

This package wraps
[is-my-json-valid](https://github.com/mafintosh/is-my-json-valid) using
[V8](https://cran.r-project.org/package=V8) to do JSON schema validation
in R.

You need a JSON schema file; see
[json-schema.org](https://json-schema.org/) for details on writing
these. Often someone else has done the hard work of writing one for you,
and you can just check that the JSON you are producing or consuming
conforms to the schema.

The examples below come from the [JSON schema
website](https://json-schema.org//learn/getting-started-step-by-step.html)

They describe a JSON based product catalogue, where each product has an
id, a name, a price, and an optional set of tags. A JSON representation
of a product is:

``` json
{
    "id": 1,
    "name": "A green door",
    "price": 12.50,
    "tags": ["home", "green"]
}
```

The schema that they derive looks like this:

``` json
{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "Product",
    "description": "A product from Acme's catalog",
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
}
```

This ensures the types of all fields, enforces presence of `id`, `name`
and `price`, checks that the price is not negative and checks that if
present `tags` is a unique list of strings.

There are two ways of passing the schema in to R; as a string or as a
filename. If you have a large schema loading as a file will generally be
easiest! Here’s a string representing the schema (watch out for escaping
quotes):

``` r

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
```

Create a schema object, which can be used to validate a schema:

``` r

obj <- jsonvalidate::json_schema$new(schema)
```

If we’d saved the json to a file, this would work too:

``` r

path <- tempfile()
writeLines(schema, path)
obj <- jsonvalidate::json_schema$new(path)
```

The returned object is a function that takes as its first argument a
json string, or a filename of a json file. The empty list will fail
validation because it does not contain any of the required fields:

``` r

obj$validate("{}")
```

    ## [1] FALSE

To get more information on why the validation fails, add
`verbose = TRUE`:

``` r

obj$validate("{}", verbose = TRUE)
```

    ## [1] FALSE
    ## attr(,"errors")
    ##   instancePath schemaPath  keyword missingProperty
    ## 1              #/required required              id
    ## 2              #/required required            name
    ## 3              #/required required           price
    ##                               message          schema
    ## 1    must have required property 'id' id, name, price
    ## 2  must have required property 'name' id, name, price
    ## 3 must have required property 'price' id, name, price
    ##                      parentSchema.$schema parentSchema.title
    ## 1 http://json-schema.org/draft-04/schema#            Product
    ## 2 http://json-schema.org/draft-04/schema#            Product
    ## 3 http://json-schema.org/draft-04/schema#            Product
    ##        parentSchema.description parentSchema.type
    ## 1 A product from Acme's catalog            object
    ## 2 A product from Acme's catalog            object
    ## 3 A product from Acme's catalog            object
    ##   parentSchema.properties.id.description parentSchema.properties.id.type
    ## 1    The unique identifier for a product                         integer
    ## 2    The unique identifier for a product                         integer
    ## 3    The unique identifier for a product                         integer
    ##   parentSchema.properties.name.description parentSchema.properties.name.type
    ## 1                      Name of the product                            string
    ## 2                      Name of the product                            string
    ## 3                      Name of the product                            string
    ##   parentSchema.properties.price.type parentSchema.properties.price.minimum
    ## 1                             number                                     0
    ## 2                             number                                     0
    ## 3                             number                                     0
    ##   parentSchema.properties.price.exclusiveMinimum
    ## 1                                           TRUE
    ## 2                                           TRUE
    ## 3                                           TRUE
    ##   parentSchema.properties.tags.type parentSchema.properties.tags.type
    ## 1                             array                            string
    ## 2                             array                            string
    ## 3                             array                            string
    ##   parentSchema.properties.tags.minItems
    ## 1                                     1
    ## 2                                     1
    ## 3                                     1
    ##   parentSchema.properties.tags.uniqueItems parentSchema.required dataPath
    ## 1                                     TRUE       id, name, price         
    ## 2                                     TRUE       id, name, price         
    ## 3                                     TRUE       id, name, price

The attribute “errors” is a data.frame and is present only when the json
fails validation. The error messages come straight from `ajv` and they
may not always be that informative.

Alternatively, to throw an error if the json does not validate, add
`error = TRUE` to the call:

``` r

obj$validate("{}", error = TRUE)
```

    ## Error:
    ## ! 3 errors validating json:
    ##  -  (#/required): must have required property 'id'
    ##  -  (#/required): must have required property 'name'
    ##  -  (#/required): must have required property 'price'

The JSON from the opening example works:

``` r

obj$validate('{
    "id": 1,
    "name": "A green door",
    "price": 12.50,
    "tags": ["home", "green"]
}')
```

    ## [1] TRUE

But if we tried to enter a negative price it would fail:

``` r

obj$validate('{
    "id": 1,
    "name": "A green door",
    "price": -1,
    "tags": ["home", "green"]
}', verbose = TRUE)
```

    ## [1] FALSE
    ## attr(,"errors")
    ##   instancePath                 schemaPath keyword params.comparison
    ## 1       /price #/properties/price/minimum minimum                 >
    ##   params.limit     message schema parentSchema.type parentSchema.minimum
    ## 1            0 must be > 0      0            number                    0
    ##   parentSchema.exclusiveMinimum data dataPath
    ## 1                          TRUE   -1   /price

…or duplicate tags:

``` r

obj$validate('{
    "id": 1,
    "name": "A green door",
    "price": 12.50,
    "tags": ["home", "home"]
}', verbose = TRUE)
```

    ## [1] FALSE
    ## attr(,"errors")
    ##   instancePath                    schemaPath     keyword params.i params.j
    ## 1        /tags #/properties/tags/uniqueItems uniqueItems        0        1
    ##                                                          message schema
    ## 1 must NOT have duplicate items (items ## 1 and 0 are identical)   TRUE
    ##   parentSchema.type parentSchema.type parentSchema.minItems
    ## 1             array            string                     1
    ##   parentSchema.uniqueItems       data dataPath
    ## 1                     TRUE home, home    /tags

or just basically everything wrong:

``` r

obj$validate('{
    "id": "identifier",
    "name": 1,
    "price": -1,
    "tags": ["home", "home", 1]
}', verbose = TRUE)
```

    ## [1] FALSE
    ## attr(,"errors")
    ##   instancePath                    schemaPath     keyword params.type
    ## 1          /id          #/properties/id/type        type     integer
    ## 2        /name        #/properties/name/type        type      string
    ## 3       /price    #/properties/price/minimum     minimum        <NA>
    ## 4      /tags/2  #/properties/tags/items/type        type      string
    ## 5        /tags #/properties/tags/uniqueItems uniqueItems        <NA>
    ##   params.comparison params.limit params.i params.j
    ## 1              <NA>           NA       NA       NA
    ## 2              <NA>           NA       NA       NA
    ## 3                 >            0       NA       NA
    ## 4              <NA>           NA       NA       NA
    ## 5              <NA>           NA        0        1
    ##                                                          message  schema
    ## 1                                                must be integer integer
    ## 2                                                 must be string  string
    ## 3                                                    must be > 0       0
    ## 4                                                 must be string  string
    ## 5 must NOT have duplicate items (items ## 1 and 0 are identical)    TRUE
    ##              parentSchema.description parentSchema.type parentSchema.minimum
    ## 1 The unique identifier for a product           integer                   NA
    ## 2                 Name of the product            string                   NA
    ## 3                                <NA>            number                    0
    ## 4                                <NA>            string                   NA
    ## 5                                <NA>             array                   NA
    ##   parentSchema.exclusiveMinimum parentSchema.type parentSchema.minItems
    ## 1                            NA              <NA>                    NA
    ## 2                            NA              <NA>                    NA
    ## 3                          TRUE              <NA>                    NA
    ## 4                            NA              <NA>                    NA
    ## 5                            NA            string                     1
    ##   parentSchema.uniqueItems          data dataPath
    ## 1                       NA    identifier      /id
    ## 2                       NA             1    /name
    ## 3                       NA            -1   /price
    ## 4                       NA             1  /tags/2
    ## 5                     TRUE home, home, 1    /tags

The names comes from within the `ajv` source, and may be annoying to
work with programmatically.

There is also a simple interface where you take the schema and the json
at the same time:

``` r

json <- '{
    "id": 1,
    "name": "A green door",
    "price": 12.50,
    "tags": ["home", "green"]
}'
jsonvalidate::json_validate(json, schema, engine = "ajv")
```

    ## [1] TRUE

However, this will be much slower than building the schema object once
and using it repeatedly.

Prior to 1.4.0, the recommended way of building a reusable validator
object was to use
[`jsonvalidate::json_validator`](https://docs.ropensci.org/jsonvalidate/reference/json_validator.md);
this is still supported but note that it has different defaults to
[`jsonvalidate::json_schema`](https://docs.ropensci.org/jsonvalidate/reference/json_schema.md)
(using imjv for backward compatibility).

``` r

v <- jsonvalidate::json_validator(schema, engine = "ajv")
v(json)
```

    ## [1] TRUE

While we do not intend on removing this old interface, new code should
prefer both
[`jsonvalidate::json_schema`](https://docs.ropensci.org/jsonvalidate/reference/json_schema.md)
and the `ajv` engine.

## Combining schemas

You can combine schemas with `ajv` engine. You can reference definitions
within one schema

``` r

schema <- '{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "definitions": {
    "city": { "type": "string" }
  },
  "type": "object",
  "properties": {
    "city": { "$ref": "#/definitions/city" }
  }
}'
json <- '{
    "city": "Firenze"
}'
jsonvalidate::json_validate(json, schema, engine = "ajv")
```

    ## [1] TRUE

You can reference schema from other files

``` r

city_schema <- '{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type": "string",
  "enum": ["Firenze"]
}'
address_schema <- '{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type":"object",
  "properties": {
    "city": { "$ref": "city.json" }
  }
}'

path <- tempfile()
dir.create(path)
address_path <- file.path(path, "address.json")
city_path <- file.path(path, "city.json")
writeLines(address_schema, address_path)
writeLines(city_schema, city_path)
jsonvalidate::json_validate(json, address_path, engine = "ajv")
```

    ## [1] TRUE

You can combine schemas in subdirectories. Note that the `$ref` path
needs to be relative to the schema path. You cannot use absolute paths
in `$ref` and jsonvalidate will throw an error if you try to do so.

``` r

user_schema = '{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type": "object",
  "required": ["address"],
  "properties": {
    "address": {
      "$ref": "sub/address.json"
    }
  }
}'

json <- '{
  "address": {
    "city": "Firenze"
  }
}'

path <- tempfile()
subdir <- file.path(path, "sub")
dir.create(subdir, showWarnings = FALSE, recursive = TRUE)
city_path <- file.path(subdir, "city.json")
address_path <- file.path(subdir, "address.json")
user_path <- file.path(path, "schema.json")
writeLines(city_schema, city_path)
writeLines(address_schema, address_path)
writeLines(user_schema, user_path)
jsonvalidate::json_validate(json, user_path, engine = "ajv")
```

    ## [1] TRUE
