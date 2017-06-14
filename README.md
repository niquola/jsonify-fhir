## FHIR  JSON+ format

Design of FHIR formats, to be honest, was essentially driven by XML and Object
Oriented implementations. Not so much attention was given to design idiomatic
JSON format. JSON and other JSON like formats like yaml, edn, avro are very
popular in avantе-garde of programmers, some modern databases now could understad
JSON as first-class data-structures, that’s why i think, good design for it is
strictly required for FHIR adoption. I would like list some problems with
current JSON format and propose solutions.

## Problem 1: variable type elements

Some resource’s elements could have variable type, in specification such
elements have [x] postfix (for example Observation.value[x]). In JSON
representation such elements encoded by substitution of postfix with specific
title-cased type name:

```
{ resourceType: "Observation", valueString: "....", ...}
{ resourceType: "Observation", valueNumber: 42, .... }
```

a) This approach to representation force unnececiary constraint: Elements that
have a choice of data type cannot repeat. I.e. They must have a maximum
cardinality of 1. There is no absolute reason to force this besides format
representation.

b) On the other side, most of Object Oriented implementations of FHIR usually
provide convenient accessors to “polymorphic elements” like
observation.getValue(), but when you work with data, without any object wrapper,
you have to manually handle this. For example check which specific
Observation.value your are dealing with by iterating through object.

c) JSON schema is very popular way to specify shape of JSON objects. But it’s
features is not enough to describe, that only one of postifexed keys are allowed
in object (i.e. valueString or valueQuantity) or if element is required say at
least one of keys should be present.

d) Implementation of FHIR search for missing elements, like
Observation?value:missing=true, in databases
native support of JSON is tricky with this representation.

This problems probably is consequence of contradiction, which was put into
representation — we represent one entity (variable type attribute) with multiple
entities (postfixed keys).

Here is variant of encoding, which solves listed concerns:

```
{ value: { $type: "String", string: "..."}}
{ value: { $type: "Quantity", Quantity: {....}}}
```

Variable type attribute could be encoded with object with meta attribute $type
(or any other meta name) and value with key named after type.
So fix inconsistency with keys and explicity embed
type information into representation. Also we relax *arity one* constrant 
and could have collections of *variable type* elements.

a. collections of [x]

```
{ value: [{type: "String", string: "..."}, {type: "Number", number: 42}], ...}
```

b. logical access to element

```
object.value
```

c. JSON schema required and mutial exclusion

```
{ 
  properties: { 
    value: { 
       properties: {
          $type: {enum: ["String", "Number"]}
          string: ....
          number: ....
        }
    }
  },
  required: [value]

}
```

d. In databases

```
SELECT * FROM resource WHERE json_path(res, 'value') is NULL;
```

## Problem 1: primitive types extensions

In FHIR you could extend primitive types with additional attributes. 
This encoded in JSON with "_" prefix and has similar to *variable type* elements problems
with collections, schema, databases etc:

```
{ 
  value: 42'
  _value { extension: [....]}
}

```

We could apply same approach and encode primitives with objects:

```
{ 
  value: {
    $type: "Number",
    number: 42,
    extension: [...]
  }
}
```

Additionally we fix *weak typed* JSON by embeding type labels into object

We pay by deeper paths and some size of Resource (because of additional $type labels):

``` 
obj.value vs obj.value.number 

{
  resourceType: "Patient",
  id: "example",
  identifier: [ ... ],
  active: { 
     $type: "boolean"
     boolean: true
  },
  gender: { 
     $type: "code"
     code: "male"
  },
  birthDate: {
    $type: "date",
    date: "1974-12-25",
    extension": [
      {
        "url": "http://hl7.org/fhir/StructureDefinition/patient-birthTime",
        "valueDateTime": "1974-12-25T14:35:45-05:00"
      }
    ]
  }
  deceased: {
    $type: 'boolean',
    boolean: false
  },
  managingOrganization: {
    "reference": "Organization/1"
  }
}
```

But we simplify typed parsing of JSON because of type information is in document 
and does not required lookup in meta-data. 
We also allow collections of extended primitives.

We could go father and add $type annotation attribute to other elements
(i.e. complex elements or datatypes):

```
  managingOrganization: {
    "reference": "Organization/1"
  }

  managingOrganization: {
    $type: "Reference",
    "reference": "Organization/1"
  }

```


## Problem 3: extensions representation

Extensions representation is essentialy driven by statically typed implementations
and XML. Both of have trubles with custom key-value objects and force to represent
extensions like collection. Such representation again force constraint on collections 
, unnececiary order,  complexity with incidental accessing such attributes and expression in JSON schema.
But for JSON-like data-structures more natural would be use
object representation - just compare:


```
{
  resourceType: "Patient",
  extension: [{
      url: "ombCategory",
      valueCoding: {
        "system": "http://hl7.org/fhir/v3/Race",
        "code": "2106-3",
        "display": "White"
      }
    },....
  ]
}

pt.extension.where(system=...).valueCoding.code

{
  resourceType: "Patient",
  extension: {
    "omb/race": {
       $type: "Codding",
       coding: {
         system: "http://hl7.org/fhir/v3/Race",
         code: "2106-3",
         display: "White"
       }
    }
  }
}

pt.extension["omb/race"].coding.code

```

## Problem 4: references

All implementers parse strings to work with references. 
There are no clear distinction between local & exteranal refs.
It would be nice to have refs more structured:

```json

// for local refs
{
   "subject": {"resourceType": "Patient", "id": "pt-1"}
}
// for external

{
   "subject": {"uri": "http://otherserver/Patient/pt-1"}
}
```

This apprach also play well with inlining resolved references:


```
{ "subject": {"resourceType": "Patient", "id": "pt-1"} }
// after inline

{ "subject": {"resourceType": "Patient", "id": "pt-1", "name": [{....}], ....} }
```


## Proposal


We propose to introduce next JSON format into FHIR,
for compatability reason it could be introduced as addition
not replacement of existing JSON format with some new MIME type.

Formal specification:

All elements encoded with annotation $type (attribute resourceType could be also generalized to $type):

Resource:

```
{ $type: "ResourceType" }
```

Element:

```
{
  $type: "Patient"
  contact: [{
    $type: "Patient.contact",
    ...
  }]
}
```

Complex datatype:

```
{
  $type: "Patient"
  contact: [{
    $type: "Patient.contact"
    telecom: [
     { 
       $type: "ContactPoint",
       ...
     }
    ]
  }]
}
```

Primitive datatypes:


```
{
  $type: "Patient"
  ...
  deceased: {
    $type: 'boolean',
    boolean: false
  },
  ...
}
```

Extensions are encoded with object not collection:

```
{
  resourceType: "Patient",
  extension: {
    $schema: {"omb/race": "http://extension-url"} // optional
    "omb/race": {
       $type: "Coding",
       coding: {
         system: "http://hl7.org/fhir/v3/Race",
         code: "2106-3",
         display: "White"
       }
    },
    "local/ext" {
       $type: "Number",
       number: 42
    }
  }
}
```

There are a lot of moder formats which easly could be deduced from
"ideomatic" JSON:

* yaml
* graphql
* edn
* Avro

## Problem 4: naming conventions for collections

Idiomatic JSON naming convention is to use plural names for collections, e.g:

```json

// for local refs
{
   "name": [....]
}
```

### Proposal

We propose using plural names to hint that the value is a collection type:

```json

// for local refs
{
   "names": [....]
}
```


## Notes

Problem with polymorphic elements and extensions is solved very nice in json-ld by context.
So, may be we could use json-ld with some constraints/conventions as primary format for FHIR JSON.


Also Grahame published `manifest` proposal, which is similar solution - http://www.healthintersections.com.au/?p=2626
