---
CIP: 57
Title: Plutus Contract Blueprint
Authors: KtorZ <matthias.benkort@cardanofoundation.org>, scarmuega <santiago@carmuega.me>
Discussions-To: https://github.com/cardano-foundation/CIPs/pull/258
Status: Draft 
Type: Process
Created: 2022-05-15
License: CC-BY-4.0
---

## Abstract

This document specifies a language for documenting Plutus contracts in a machine-readable manner. This is akin to what [OpenAPI](https://swagger.io/specification) or [AsyncAPI](https://www.asyncapi.com/docs/specifications/v2.4.0) are for, documenting HTTP services and asynchronous services respectively. In a similar fashion, A Plutus contract has a binary interface which is mostly defined by its datum and redeemer. Since all interactions with contracts are done through transactions, we can also model _endpoints_ of a contract as transitions from a known state to another purely declaratively. 

This document is therefore a meta-specification defining the vocabulary and validation rules with which one can specify a Plutus contract interface, a.k.a **Plutus contract blueprint**.

## Motivation

While publicly accessible, on-chain contracts are currently inscrutable. Ideally, one would want to get an understanding of transactions revolving around script executions. This is both useful to visualize and to control the evolution of a contract life-cycle; but also, as a user interacting with a contract, to ensure that one is authorizing a transaction to do what it's intended to. Having a machine-readable specification in the form of a JSON-schema makes it easier (or even, possible) to enable a wide variety of use cases from a single concise document, such as:

- Code generators for serialization/deserialization of Contract's elements
- Contract API Reference / Documentation, also automatically generated
- Extra automated transaction validation layers 
- Better wallet UI / integration with DApps
- Automated Plutus-code scaffolding 

Moreover, by making the effort to write a clear specification of their contracts, DApps developers make their contracts easier to audit (as they're able to specify the expected behavior). 

## Specification

### Overview 

This specification introduces the notion of a _Plutus contract blueprint_, as a JSON document which it itself a _JSON-schema_ as per the definition of given in [JSON Schema: A Media Type for Describing JSON Documents: Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core.html).

Said differently, Plutus blueprints are first and foremost, valid JSON schemas (according to the specification linked above). This specification defines a core vocabulary and additional keywords which are tailored to the specification of Plutus contracts. Tools supporting this specification must implement the semantic and validation rules specified in this document. 

Meta-schemas for Plutus blueprints (i.e. schemas used for validating Plutus blueprints themselves) are given [in annexe](./schemas/README.md).

### Document Structure 

A _Plutus contract blueprint_ is made of a single document describing one or more on-chain validators. The document itself is a JSON object with the following fields:

| Fields                       | Description                                               |
| ---                          | ---                                                       |
| [preamble](#preamble)        | An object with meta-information about the contract        |
| [validators](#validators)    | An object of named validators                             |
| ?[endpoints](#endpoints)     | A description of the contract binary interface            |
| ?[definitions](#definitions) | A registry of definition re-used across the specification |

Note that examples of specifications are given later in the document to keep the specification succinct enough and not bloated with examples.

#### preamble

The `preamble` fields stores meta-information about the contract such as version numbers or a short description. This field is mainly meant for humans as a mean to contextualize a specification. 

| Fields       | Description                                                              |
| ---          | ---                                                                      |
| title        | A short and descriptive title of the application                         |
| ?description | A more elaborate description                                             |
| version      | A version number                                                         |
| license      | A license under which the specification and contract code is distributed |

#### validators

Where the essence of the specification lies. This section describes each validator involved in the contract (simple applications will likely have only a single validator). Each validator can be named, as a UTF-8 identifier. A validator is mainly defined by three things: a redeemer, a datum and some compiled code. Note that the datum is optional in the case of a minting policy. 

| Fields        | Description                                                     |
| ---           | ---                                                             |
| ?description  | An informative description of the validator                     |
| ?datum        | A description of the datum format expected by this validator    |
| redeemer      | A description of the redeemer format expected by this validator |
| ?compiledCode | The full serialized / compiled script template                  |

#### endpoints

A list of endpoints for the contract which may trigger one of more validators. Each endpoint describes a transition from zero or more states (i.e. validators' input datums) and to zero or more states (i.e.  validators' output datums) via some redeemers. 

| Fields       | Description                                                        |
| ---          | ---                                                                |
| ?description | An informative description of the transition                       |
| from         | A set of validators' datums from which the transition can start    |
| to           | A set of validators' datums as a result of applying the transition |
| via          | Description of the necessary redeemers to perform the transition   |

#### definitions

A set of extra schemas to be re-used as references across the specification.

### Core vocabulary

Plutus blueprints ultimately describes on-chain data value. The smallest unit we consider is called a _Plutus Data_. We therefore replace the JSON-schema vocabulary with a new set of primitives to represent any _Plutus Data_:

| Primitive     | Description
| ---           | ---
| `integer`     | A signed integer at an arbitrary precision
| `bytes`       | A bytes string of an arbitrary length
| `list`        | An ordered list of Plutus data
| `map`         | An associative list of Plutus data keys and values
| `constructor` | A generic constructor primitive to define Plutus Data as sums of products

Using these primitives, it becomes possible to represent the entire domain (i.e. possible values) which can be manipulated by Plutus contracts. 

We do however provide some convenient core definitions which can be referenced in blueprints. These definitions cover the following instances: `Address`, `AssetId`, `AssetName`, `Bool`, `Certificate`, `Credential`, `Data`, `Datum`, `DatumHash`, `DelegationCredential`, `Ed25519Signature`, `Ed25519SigningKey`, `Ed25519VerificationKeyHash`, `Ed25519VerificationKey`, `PolicyId`, `Rational`, `ScriptHash`, `ScriptPurpose`, `StakePoolId`, `TimeRange`, `TransactionId`, `TransactionInput`, `TransactionOutputReference`, `TransactionOutput`, `Transaction`, `Unit`, `ValidatorHash`, `Value`, `VrfKeyHash`.

These definitions are [provided in the meta-schema](./schemas/v1/plutus-types.json) in annexe, more may be added later to catch-up with evolution of the core protocol. 

### Additional keywords

Similarly to JSON schemas, we provide extra validation keywords and keywords for applying subschemas with logic to further refine the definition of core primitives. Keywords allow to combine core data-types into bigger types and we'll later give some pre-defined definitions which we assume to be part of the core vocabulary and therefore, recognized by any tool supporting this standard. 

When presented with a validation keyword with a malformed value (e.g. `"maxLength": "foo"`), programs are expected to return an appropriate error.

Beside, we define a _Plutus Data Schema_ as a JSON object with a set of fields depending on its corresponding data-type. When we refer to a _Plutus Data Schema_, we refer to the entire schema definition, with its validations and with the semantic of each keywords applied. 

Here below are detailed all the accepted keywords for each data-type.

#### For any data-type

> 💡 Keywords in this section applies to any instance data-type described above. 

##### dataType 

The value of this keyword must be a string, with one of the following value: `integer`, `bytes`, `list`, `map` or `constructor`. This keyword is mandatory for any instance and defines the realm of other applicable keywords for that instance.

##### title 

This keyword's value must be a string. This keyword can be used to decorate a user interface and qualify an instance with some short title.

##### description

This keyword's value must be a string. This keyword can be used to decorate a user interface and provide explanation about the purpose of the instance described by this schema.

##### $comment

This keyword's value must be a string. It is meant mainly for programmers and humans reading the specification. This keyword should be ignored by programs.


##### allOf

This keyword's value must be a non-empty array.  Each item of the array MUST be a valid _Plutus Data Schema_. An instance validates successfully against this keyword if it validates successfully against all schemas defined by this keyword's value.

##### anyOf

This keyword's value must be a non-empty array. Each item of the array must be a valid _Plutus Data Schema_. An instance validates successfully against this keyword if it validates successfully against at least one schema defined by this keyword's value. 

##### oneOf

This keyword's value must be a non-empty array. Each item of the array must be a valid _Plutus Data Schema_. An instance validates successfully against this keyword if it validates successfully against exactly one schema defined by this keyword's value.

##### maybe 

This keyword's value must be a valid _Plutus Data Schema_. An instance validates successfully against this keyword if it validates successfully against the equivalent schema by the transformation here under:

<table>
<tr><td>
  <pre>
  {
    "maybe": __SUBSCHEMA__
  } 
  </pre>
</td><td>
⇒
</td><td>
<pre>
{
  "oneOf": [
    { 
      "dataType": "constructor"
      "index": 0 
      "fields": [ __SUBSCHEMA__ ]
    },
    {
      "dataType": "constructor"
      "index": 1
      "fields": []
    }
  ]
}
</pre>
</td>
</tr>
</table>

##### not

This keyword's value must be a valid _Plutus Data Schema_. An instance is valid against this keyword if it fails to validate successfully against the schema defined by this keyword.

#### For `{ "dataType": "bytes" }`

> 💡 Keywords in this section only applies to `bytes`. Using them in conjunction with an invalid data-type should result in an error.

##### enum 

The value of this keyword must be an array of hex-encoded string literals. An instance validates successfully against this keyword if once hex-encoded, its value matches one of the elements of the keyword's values. 

##### maxLength 

The value of this keyword must be a non-negative integer. A bytes instance is valid against this keyword if its length is less than, or equal to, the value of this keyword. 

##### minLength 

The value of this keyword must be a non-negative integer. A bytes instance is valid against this keyword if its length is greater than, or equal to, the value of this keyword. 

#### For `{ "dataType": "integer" }`

> 💡 Keywords in this section only applies to `integer`. Using them in conjunction with an invalid type should result in an error.

##### multipleOf

The value of "multipleOf" must be a integer, strictly greater than 0. The instance is valid if division by this keyword's value results in an integer.

##### maximum  

The value of "maximum" must be a integer, representing an inclusive upper limit. This keyword validates only if the instance is less than or exactly equal to "maximum".

##### exclusiveMaximum

The value of "exclusiveMaximum" must be an integer, representing an exclusive upper limit. The instance is valid only if it has a value strictly less than (not equal to) "exclusiveMaximum".

##### minimum 

The value of "minimum" must be an integer, representing an inclusive lower limit. This keyword validates only if the instance is greater than or exactly equal to "minimum".

##### exclusiveMinimum 


The value of "exclusiveMinimum" must be a integer, representing an exclusive lower limit. The instance is valid only if it has a value strictly greater than (not equal to) "exclusiveMinimum".

#### For `{ "dataType": "list" }`

> 💡 Keywords in this section only applies to `list`. Using them in conjunction with an invalid data-type should result in an error.

##### items 

The value of this keyword must be either another _Plutus Data Schema_, or an array of _Plutus Data Schema_. This keywords applies its subschema to all child instances of the list. When it is defined as a single subschema, then it applies to all items of the list. When it is defined as a array, then 

##### maxItems 

The value of this keyword must be a non-negative integer. An array instance is valid against "maxItems" if its size is less than, or equal to, the value of this keyword.

##### minItems 

The value of this keyword must be a non-negative integer. A list instance is valid against "minItems" if its size is greater than, or equal to, the value of this keyword. Omitting this keyword has the same behavior as a value of 0.

##### uniqueItems

The value of this keyword must be a boolean. If this keyword has boolean value false, the instance validates successfully. If it has boolean value true, the instance validates successfully if all of its elements are unique.

#### For `{ "dataType": "map" }`

> 💡 Keywords in this section only applies to `map`. Using them in conjunction with an invalid data-type should result in an error.

##### properties

The value of "properties" must be an object. Each value of this object must be a valid _Plutus Data Schema_. 

Validation succeeds if, for each name that appears in both the instance and as a name within this keyword's value, the child instance for that name successfully validates against the corresponding schema. Omitting this keyword has the same assertion behavior as an empty object.

##### additionalProperties

The value of "additionalProperties" must be a valid JSON Schema.

The behavior of this keyword depends on the presence and annotation results of "properties" within the same schema object. Validation with "additionalProperties" applies only to the child values of instance names that do not appear in the annotation results of either "properties".

For all such properties, validation succeeds if the child instance validates against the "additionalProperties" schema. Omitting this keyword has the same assertion behavior as an empty schema.

##### propertyNames

The value of "propertyNames" must be a valid JSON Schema.

If the instance is an object, this keyword validates if every property name in the instance validates against the provided schema. Note the property name that the schema is testing will always be a string. Omitting this keyword has the same behavior as an empty schema.

##### maxProperties

The value of this keyword must be a non-negative integer. An object instance is valid against "maxProperties" if its number of properties is less than, or equal to, the value of this keyword.

##### minProperties

The value of this keyword must be a non-negative integer. An object instance is valid against "maxProperties" if its number of properties is greater than, or equal to, the value of this keyword.

#### For `{ "dataType": "constructor" }`

> 💡 Keywords in this section only applies to `constructor`. Using them in conjunction with an invalid data-type should result in an error.

##### index

This keyword's value must be a non-negative integer. An instance is valid against this keyword if it represents a Plutus constructor whose index is the same as this keyword's value. 

##### fields

This keyword's value must be an array of valid _Plutus Data Schema_; possibly empty. It must also always be specified with the `index` keyword. An instance is valid against this keyword if it represents a Plutus constructor for which each field is valid under each subschema given by this keyword's value. Fields are compare positionally. 

## Example(s)

<details>
  <summary>Lobster Challenge Contract</summary>

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",

  "$id": "https://cips.cardano.org/cips/cip57/schemas/v1/lobster-challenge.json",

  "$vocabulary": {
    "https://json-schema.org/draft/2020-12/vocab/core": true,
    "https://json-schema.org/draft/2020-12/vocab/applicator": true,
    "https://json-schema.org/draft/2020-12/vocab/validation": true,
    "https://cips.cardano.org/cips/cip57": true
  },

  "preamble": {
    "title": "Lobster Challenge",
    "description": "A specification of the lobster challenge contract: <https://github.com/input-output-hk/lobster-challenge/>",
    "version": "1.0.0",
    "license": "Apache-2.0"
  },
  "validators": {
    "main": {
      "description": "The lobster contract validator, verifying vote counts and driving token evolution.",
      "datum": {
        { "$ref": "plutus-types.json#/$defs/Unit" }
      },
      "redeemer": {
        { "$ref": "plutus-types.json#/$defs/Unit" }
      },
      "compiledCode": {
        "template": "59094c01000{{ lobsterParams }}006002003",
        "arguments": {
          "lobsterParams": {
            "$ref": "#/definitions/LobsterParams" 
          }
        }
      }
    },
    "contractIdPolicy": {
      "description": "A minting policy to create a unique thread token for the contract.",
      "compiledCode": {
        "template": "59094c01000{{ tokenName }}{{ txOutRef }}00402",
        "arguments": {
          "tokenName": { "$ref": "plutus-types.json#/$defs/AssetName" },
          "txOutRef": { "$ref": "plutus-types.json#/$defs/TransactionOutputReference" }
        }
      }
    },
    "alwaysTruePolicy": {
      "description": "A minting policy that is always true.",
      "compiledCode": {
        "template": "59094c01000006002003"
      }
    }
  },
  "endpoints": {
    "init": {
      "description": "Initialize the contract instance. The state is carried not by datums, but by token values minting with the 'alwaysTruePolicy'", 
      "from": null,
      "to": { "$refs": "#/validators/main/datum" },
      "via": { "$refs": "#/validators/main/redeemer" },
    },
    "vote": {
      "description": "A vote that is increasing the voting count and the name counter by minting more tokens.", 
      "from": { "$refs": "#/validators/main/datum" },
      "to": { "$refs": "#/validators/main/datum" },
      "via": { "$refs": "#/validators/main/redeemer" },
    }
  },
  "definitions": {
    "LobsterParams": {
      "dataType": "sum",
      "cases": {
        "LobsterParams": [
          { "dataType": "integer", "title": "seed" },
          { "$ref": "core#/$defs/PolicyId", "title": "nft" },
          { "$ref": "core#/$defs/PolicyId", "title": "counter" },
          { "$ref": "core#/$defs/PolicyId", "title": "vote" },
          { "dataType": "integer", "minimum": 0, "title": "nameCount" },
          { "dataType": "integer", "minimum": 0, "title": "voteCount" }
        ]
      }
    }
  } 
}
```
</details>

## Rationale

### Choice of JSON-Schemas as a foundation

JSON schemas are pervasively used in the industry for describing all sort of data models. Over the years, they have matured enough to be well understood by and familiar to a large portion of developers. Plus, tooling now exists in pretty much any major language to parse and process JSON schemas. Thus, using it as a foundation for the blueprint only makes sense. 

### Divergence from JSON-Schemas primitives

This specification defines a new set of primitives types `integer`, `bytes`, `list`, `map` and `sum` instead of the classic `integer`, `number`, `string`, `bool`, `array`, `object`, `null`. This is not only to reflect better the underlying structure of Plutus data which differs from JSON by many aspects, but also to allow defining or re-defining logic and validation keywords for each of those primitives. 

Note however that apart from the keyword `type`, the terminology (and semantic) used for JSON schemas has been preserved to not "reinvent the wheel" and makes it easier to build tools on top by leveraging what already exists. Plutus schemas do not use `type` but use `dataType` instead to avoid possible confusion with JSON-schemas. A Plutus data schema is almost a JSON-schemas, but only supports a subset of the available keywords and has subtle differences for some of them (e.g. keywords for the `bytes` data-type operate mostly on hex-encoded strings).

### Additional Resources

- https://json-schema.org/draft/2020-12/json-schema-core.html
- https://json-schema.org/draft/2020-12/json-schema-validation.html

## Path To Active 

- [ ] A command-line tool for validating Plutus blueprint specifications
- [ ] A tool for rendering Plutus blueprint specifications as documentation
- [ ] Write specifications for a few real-world contracts, identify and fix gaps
- [ ] Settle on an approach (possibly, none) to capture other contract constraints such as:
  - State tokens needed for driving certain transitions 
  - Specific structure requirements like number of inputs / outputs or their ordering

## Copyright

CC-BY-4.0
