## Examples of using TerminusDB

### Prerequisistes
Python 3.8, and
```
pip install terminusdb-client
```
Each example assumes that a terminusdb server process is already running on `127.0.0.1:6363`
Refer to [this guide on installing and running TerminusDB using Docker](https://terminusdb.com/docs/index/terminusdb/install/install-as-docker-container#install-steps)
Each example creates its own database and tears it down.

### [referenced_may_not_exist.py](referenced_may_not_exist.py)
Demonstrates that it is possible to add a document that refers to another document
that does not exist.

### [query_using_regex.py](query_using_regex.py)
Simple example of using a regular expression in a query

### [calling_people_names.py](calling_people_names.py)
Demonstrates:
* triples can be used to add properties to a document
* those triples are lost if the document is updated

### [one_way.py](one_way.py)
Demonstrates:
* it is possible to query 'upstream'
* using WOQL with variables

### [listquery.py](listquery.py)
Demonstrates:
* Programmatically using `rdf:first` and `rdf:rest` to traverse a list
* Querying a list using `WOQLQuery.path()`

### [label_problem.py](label_problem.py)
Demonstrates:
* It is best to avoid using `label` as a schema property name

### [curl_examples.sh](curl_examples.sh)
Note that this bash script also requires [person_schema.json](person_schema.json)
Demonstrates
* using the [TerminusDB document API](https://terminusdb.com/docs/index/terminusx-db/reference-guides/document-interface) and the [HTTP API](https://terminusdb.com/docs/index/terminusx-db/reference-guides/http-api)

### [define_a_schema_and_add_a_doc.py](define_a_schema_and_add_a_doc.py)
Defines a Manuscript with a creator that is a Person and a `sdPublisher` that is 
an Organisation.
Definition of these classes follows their definition in schema.org

### [composite.py](composite.py)
A Composite is identified by its `ark` property and contains zero or more Composites. 
```
    {
        "@id"           : "Composite",
        "@type"         : "Class",
        "name"          : "xsd:string",
        "ark"           : "xsd:string",
        "@key"          :
        {
            "@type"     : "Lexical",
            "@fields"   : [ "ark" ]
        },
        "contains"      :
        {
            "@type"     : "Set",
            "@class"    : "Composite"
        }
    }
```
In composite.py, A contains B which contains C.
```
python composite.py 
{'@id': 'Composite/A', '@type': 'Composite', 'ark': 'A', 'contains': ['Composite/B'], 'name': 'A'}
{'@id': 'Composite/B', '@type': 'Composite', 'ark': 'B', 'contains': ['Composite/C'], 'name': 'B'}
{'@id': 'Composite/C', '@type': 'Composite', 'ark': 'C', 'name': 'C'}
```

### When a subdocument with the same ID occurs multiple times
Some aspects of the behaviour of terminusdb are illustrated by [when_the_subdoc_already_exists.py](when_the_subdoc_already_exists.py)
`B` is added twice, but is only listed as occurring once:
```
{'@id': 'Composite/A', '@type': 'Composite', 'ark': 'A', 'contains': ['Composite/B'], 'name': 'A'}
{'@id': 'Composite/B', '@type': 'Composite', 'ark': 'B', 'contains': ['Composite/C'], 'name': 'B'}
{'@id': 'Composite/C', '@type': 'Composite', 'ark': 'C', 'name': 'C'}
{'@id': 'Composite/D', '@type': 'Composite', 'ark': 'D', 'contains': ['Composite/B'], 'name': 'D'}
```
However, if the second occurrence of `B` differs from the first in respect of one or other of its non-identifying
properties (for example, changing the `name` of the second `B` to `b`, then an exception is raised:
```
terminusdb_client.errors.DatabaseError: Schema check failure
{
    "@type": "api:ReplaceDocumentErrorResponse",
    "api:error": {
        "@type": "api:SchemaCheckFailure",
        "api:witnesses": [
            {
                "@type": "instance_not_cardinality_one",
                "class": "http://www.w3.org/2001/XMLSchema#string",
                "instance": "bodleian://famous/data/Composite/B",
                "predicate": "http://bodleian.ox.ac.uk#name"
            }
        ]
    },
    "api:message": "Schema check failure",
    "api:status": "api:failure"
}
```
### Trivial WOQL query
[trivial_woql_query.py](trivial_woql_query.py) is a trivial example of using the Web Object Query Language (WOQL).
I'm struggling to find good resources on using WOQL. 
There is [this blog](https://terminusdb.com/blog/the-power-of-web-object-query-language/)
and there is a [single page tutorial](https://terminusdb.com/docs/index/terminusx-db/how-to-guides/perform-graph-queries)
The query is constructed as follows
```
query = WOQLQuery()\
        .triple("v:Subject", "name", {"@type": "xsd:string", "@value": "B"})\
        .read_document("v:Subject", "v:Full Record")
```
Notes:
- use of `v: Full Record` is inspired by the aforementioned blog. I am not aware it is otherwise documented.
- Using the simpler `triple("v:Subject", "name", "B")` would have been nice but didn't work for me.

This query returns:
```
{   '@type': 'api:WoqlResponse',
    'api:status': 'api:success',
    'api:variable_names': ['Subject', 'Full Record'],
    'bindings': [   {   'Full Record': {   '@id': 'Composite/B',
                                           '@type': 'Composite',
                                           'ark': 'B',
                                           'contains': ['Composite/C'],
                                           'name': 'B'},
                        'Subject': 'Composite/B'}],
    'deletes': 0,
    'inserts': 0,
    'transaction_retry_count': 0
}
```
### Using WOQLClient.query_document
For simple queries, `WOQLClient.query_document` may be easier than using WOQL. 
See [using_query_document.py](using_query_document.py) for an example. In this example, the query 
is expressed as follows:
```
documents = client.query_document({'@type': 'Composite', 'name': 'B'})
```
Note that `@type` is not optional.
This returns a sequence containing a single document:
```
{   '@id': 'Composite/B',
    '@type': 'Composite',
    'ark': 'B',
    'contains': ['Composite/C'],
    'name': 'B'
}
```

### References to existing subdocuments
It is possible to add documents that have connections to documents that have already been added
without having to entirely repeat the definition of the latter.
In [referencing_an_existing_subdocument.py](referencing_an_existing_subdocument.py), the graph
is updated twice, with the second updating being a Composite that contains an item
that has already been added (`Composite/B`). This second update only refers to `Composite/B` by its 
`@id`:
```
D_contains_B = [
    {
        "@type": "Composite",
        "name": "D",
        "ark": "D",
        "contains":
            [
                "Composite/B"
            ]
    }
]
```

### Miscellaneous Notes

This is how terminusdb states the basic unit of specification:
```
{ 
    "@type" : "Class",
    "@id"   : "Person",
    "name"  : "xsd:string" 
}
```

There's also the Context object, e.g.
```
{   "@type"            : "@context",
    "@schema"          : "http://terminusdb.com/schema/woql#",
    "@base"            : "terminusdb://woql/data/",
    "xsd"              : "http://www.w3.org/2001/XMLSchema#",
    "@documentation"   :
    {
        "@title"       : "WOQL schema",
        "@authors"     : ["Gavin Mendel-Gleason"],
        "@description" : "The WOQL schema providing a complete specification of the WOQL syntax.
                         This enables:
                         * WOQL queries to be checked for syntactic correctness.
                         * Storage and retrieval of queries.
                         * Queries to be associated with data products.
                         * Helps to prevent errors and detect conflicts in merge of queries.",
    }
}
```
`@schema` specifies the default URI expansion

For real, could be:
```
{   "@type"            : "@context",
    "@schema"          : "http://schema.org",
    "@base"            : "bodleian://data/"
}
```

Example of a manuscript description in JSON-LD:
```
{
  "@context": "https://schema.org",
  "@type": "Manuscript",
  "sdPublisher": {
    "name": "Special Collections",
    "email": "specialcollections.enquiries@bodleian.ox.ac.uk",
    "parentOrganization": {
      "name": "Bodleian Libraries",
      "parentOrganization": {
        "name": "University of Oxford"
      }
    }
  },
  "sdDatePublished": "2022-02-15",
  "material": "paper",
  "inLanguage": "Grek",
  "creator": {
    "familyName": "Tzetzes",
    "givenName": "Johannes"
  },
  "temporal": "16th centure (before 1573)",
  "description": "MS. Auct. T. 1. 10",
  "identifier": {
    "propertyID": "shelfmark",
    "value": "MS. Auct. T. 1. 10"
  },
  "name": "MS. Auct. T. 1. 10"
}
```

The above JSON-LD has been validated using the [JSON-LD Playground](https://json-ld.org/playground/)

