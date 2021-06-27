## The Zed Type System

A key research question here is whether
it is possible to create self-describing data model that affords
an efficient implementation for applications like ETL and high-speed analytics,
requires no up front schema definitions, and allows for agile data introspection?
We felt the key to answering this question lied in the design of a new
data model's type system.

The concept of a "type system" is obviously mature and well understood.
The theory and practice behind type systems is as old as programming language
design itself.  Moreover, a large number of type systems have been explored and
implemented for data serialization formats.
Surely, it seemed, we could just adopt an existing approach for Zed.
Yet, after an extensive search, it was apparent that no existing system met
our design goals.

As far as we know, the previous approaches all fall broadly into two categories:
(1) value-based designs
where the type of the value is completely encoded in the value itself,
and (2) schema-based designs, where the types of data is specified by a schema
either embedded as meta-data or available via an external entity.

In value-based designs, each value is a standalone entity
and the data types that comprise the value are implied in the serialized format
of the encoded value, e.g, the JSON value `{"s":"hello, world", "n": 123}`
by definition is of type
"object with field s of type string and field n of type number".  JSON, BSON,
XML (without XML schema), FlexBuffers, MessagePack, and many more serialization
formats use this general approach.  The advantage here is that no
interdependency exists between the value's serialization
and some externally managed schema system, which simplifies implementation.
But a shortcoming here is the absence of a comprehensive type system
that relates the types of different values to one another --- any and all such
inference is left entirely to the application.

In schema-based designs, however, type information is specified explicitly
by schema definitions.
Here, type relationships can be determined very easily among the
explicit schemas.  For example, "column s of string and column n of number" would
represent a schema for a value "'hello, world', 123".  This is often cited as
having higher efficiency than the value-based approach
because verbose type information (like field names in an object) need not be
repeated in every value; furthermore, the approach amenable to highly efficient
columnar organizations (cite parquet, dremel).

A rich question here is how schema definitions correspond to instances of values and
there are many varied answers.
In relational systems, schemas attach to tables and values appear in tables.
In a Parquet file, the schema of all values must be pre-specified and
appears as metadata.  In systems based upon Avro, the schemas
are often managed in a centralized database (called a "schema registry")
and two communicating endpoints must coordinate with the schema registry
in order to serialize and deserialize their messages (cite KAVRO).
In Zeek logs, a schema is defined in a preamble of each log file.
In RPC-style systems (cite Protobufs, Thrift, Sun RPC), the schemas are presumed
to be known a priori to both ends and such coordination is outside the scope of
the serialization format (requiring this knowledge typically compiled
into the client and server code).

We wondered if we could have the best of both worlds: the advantages of the
value-based design where each value can be of arbitrarily complex type and
need not conform to a single, overarching schema,
and the advantages of the schema-based design where collections of values all
conform to a single schema for efficient implementation and rigid data-engineering
compliance?

After toiling around for a couple years and implementing various false starts,
we convereged upon a solution and the answer to our question turns out to be "yes".
The key to making this all work is merging the concept of the type system
from the programming-language landscape with a type system that defines
a new approach to data engineering.

### Type System Design

As with serialization formats like Avro and Parquet, Zed types are decomposed
into simple primitive types (e.g., string, integer, float, etc) and complex types
that are composed of primitive types and/or other complex types
(e.g., like record, set, array).

For example, in the human-readable ZSON serialization of the Zed data model,
a record type might look like this:
```
{ First:string, Last:string, Address:string, Zip:int64 }
```
while the type signature for an array of names looks like this;
```
[ {First:string, Last:string } ]
```
and a value of this type:
```
[ { First:"John", Last:"Doe" }, { First:"Mary", Last:"Smith" } ]
```
We considered the adoption of an "any" type or "bag" (i.e., generic container
for values of type "any").  While this approach affords great flexibility
and provides a means, for instance, to achieve semantic equivalence with
mixed-type arrays of JSON, we felt it would complicate an implementation compared
to a more determinant type system (which we elaborate upon in Section XXX).
Also, philosophically, adopting type "any"
felt more like we were simply co-implementing the document and relational
models together, as has been done in many existing systems (postgres, snowflake variant,
SQL++) instead of unifying the two models.

To this end, like Avro and Parquet, we instead adopted a "union" type in Zed.
Unions allow the expression
of mixed-type arrays but restrict such expression to a pre-determined set of types
(for any given type instance).  For example, the mixed-type JSON array
```
[ "John", 94105, true ]
```
has Zed type `[(string, float64, bool)]`, i.e., array of union of the three types,
where the ZSON syntax for a union type is simply a parenthesized list of types.

We also realized if we were to abandon externally imposed schema constraints of the
relational model in favor of data streams that described themselves and if those
data streams could have arbitrary heterogeneity and complexity that it would be
important to have a foundational means for to introspect the structure of the data.
To do so, the Zed type system includes first-class types, i.e., a type of type "type"
and we call the value of a type type a "type value".

Thus, if you query a system based on the Zed data model, e.g., to return a
count of the number of shapes of each value in a collection, the set of results could
then have the type signature
```
{ count:uint64, shape:type }
```
(XXX note power here is that the results and the origin data are stored using the
same Zed data model.  also we could mention shaping function using type values,
"messiness" metrics, distance metrics between types, etc... allude to how all this
is useful inn ETL)

Inspired by Avro's concept of a type "alias", which lets you assign a
name to a type, we felt named types would add flexibility providing a means to:
* create application-defined types that could carry custom type semantics
(as was the original goal in Avro),
* create a bridge between native language type names and Zed serialized data,
e.g., allowing a convenient mechanism to marshal and unmarshal serialized
data to and from native data structures, and
* provide a connection between the relational concept of a table name and
a named record type in Zed where all the values form a given record type
conform to a uniform, table-like schema name.

The ZSON syntax to create a type name is embedded in a type decorator, e.g.,
these two ZSON values
```
{ First:"John", Last:"Doe" } (=Name)
{ First:"Mary", Last:"Smith" } (Name)
```
create and then use the type "Name" binding it to the underlying type
`{First:string,Last:string}`.

Finally, we realized another important first-class type is "error".
It's often tricky and mysterious to debug errors that arise in large-scale
queries across massive amounts of data.  By embedding error values
in the results, debugging is often much easier.  This also enables live
systems that, for instance, perform ETL, to stream expected results as usual
but also embed exceptional error conditions as just another data value.
In this ways, errors can actually land on the target system and be queried
to greatly facilitate the debugging complex data pipelines when unexpected
changes occur.

Another example error scenario is a divide by zero error that
occurs rarely in a large aggregation with many rows of output.  Since
Zed can simply embed an error result in an otherwise uniform set of output rows,
the computation can proceed in the face of the error and the user can drill
down by further querying the output table and filtering on type error then
working backward with additional queries on the source data to isolate the problem.
This is far superior to simply exiting the query with the error message
"divide by zero".

Another powerful use of the error type is for the value "MISSING".
Unlike SQL, systems that support querying or processing semi-structured data
do not necessarily know up front what fields are present in the source data
and generally allow referencing arbitrary object keys.  In SQL, a reference
to an unknown column can be detected at compile-time but in semi-structured
systems, operations can be applied to values where references to non-existant
keys are allowed to proceed.  To accommodate this, systems like SQL++, PartiQL,
CouchDB, etc define the value MISSING to be the result of referencing a
non-existent object (which is distinct from the explicit NULL value).
Then the semantic of operators are defined with respect to MISSING values,
e.g., "true OR MISSING", "MISSING + 0", etc.

We find the idea of a MISSING value not only distasteful but oxymoronic.
Here I am but I'm not here.  And just what is the type of a MISSING value?
Armed with type "error" in Zed, we felt missing should simply be an error.
With an error called MISSING then, there is no special casing the error condition
in the output as it's just another Zed value, and a runtime can handle
"MISSING" errors however it would like, e.g., defining semantics similar to
other systems that use MISSING as a special value.

### The Type Context

By embracing a design based on self-describing types, the need arises to
construct the types in use at any given time.  This approach differs from
systems based on schema registries, which store the types for any and all
data referenced in some central database.  Zed, on the other hand, builds
up ephemeral state to represent only the types locally in use by extracting
type definitions incrementally from a sequence or collection of Zed values.

This state representing all of the types in use for a given set of Zed values
at a given point in time is called the Zed "type context".  As values are
processed by an entity reading Zed data, the type context is populated with bindings
between a reference (e.g., a small integer) and a definition (e.g., a
type declaration referring to a new type composed of other references) while
primitive types are defined by reserved references (e.g., in ZNG, primitive types
are integers 0 through 22).  In other words,
the type context is like a dictionary and values can simply reference their entry
in the dictionary.

For example, consider the following two record values represented as ZSON
```
{addr:192.168.1.1, port:80 (port=(uint16))} (=socket)
{addr:192.168.1.2, port:8080 } (socket)
```
Rather than parse this human-readable format a typical implementation would
store Zed data for query and computation in a serialized binary form,
e.g., ZNG for row-based organization and ZST for column-based organization
where the localized type context provides the necessary type system to
interpret the serialized values.

In ZNG, the two values above are encoded as follows, where we use a
pseudo-assembler syntax here to illustrate the concept:
```
tyepdef <23> alias "port" <uint16>
typedef <24> record "addr" <IP>, "port" <23>
typedef <25> alias "socket" <24>
value <25> 192.168.1.1 80
value <25> 192.168.1.2 8080
```
Here, `<23>`, `<24>`, and `<25>` are dynamically allocated type identifiers
and `<IP>` and `<uint16>` represent primitive type identifiers for the indicated
types.  (Dynamic IDs are numbered beginning at 23 as there are 23 primitive
types numbered 0-22.)  An entity that processes this stream of values would build
a type context with the follow state:
```
|-------------------------------------|
| 23 | alias "port" <uint16>          |
| 24 | record "addr" <IP> "port" <23> |
| 25 | alias "socket" <24>            |
|-------------------------------------|
```
It turns out that the implementation of the type context serves as a crucial
and efficient data structure in a Zed implementation: the type ID of each value
in a sequence of record values defines the entire type structure of the value and
the type can be efficiently determined by a lookup in the type context.
Type equivalence within a context is easily achieved by comparing type IDs.

This outermost record type definition is not unlike a schema, so in a sense, the
type context provides a means to map a dynamic problem --- managing arbitrarily
complex heterogeneous records as a sequence of values --- into a static problem
where the type ID determines a fixed schema for all the values that comprise
said type.

A prototypical challenge is how multiple data sources with different type contexts
are merged and processed together, e.g., when reading different files of Zed
data, or when joining data over a cluster from one data pool to another, etc.
With formats like JSON, this isn't a problem of course,
because each value describes its own structure; however, parsing each value to
determine its type and relate its type to other types that have been previously
encountered begets significant overhead.

With a type context, however, types are explicitly encoded and translating
value sequences from one context to another can be highly efficient through
the use of memoization.  Suppose a source value sequence with type context A
is processed by a target system with type context B.  To transmit a value
across this boundary, we must translate its type T from context A to context B.
Such a translation is a relatively heavy-weight operation
as we must first translate the type  
from context A into a canonical format then lookup that canonical format in
context B to determine the dynamic version of T relative to B.
Yet after this work is done once, we can remember that the particular type from
context A is the same as the type in context B.  This can be simply implemented
as a hash table of small-integer, dynamic type identifiers mapping an input
type ID to an output type in a very efficient per-value lookup;
furthermore, in our design, there is no need to translate values.

Another use of memoization is to create run-time optimizations.  For example,
components of a query plan could be dynamically optimized for each type
encountered, then execution logic stored in a memoization table for each type,
i.e., where the execution logic could JIT-compiled code (reference related work)
or even simple dynamic closures composed of type specific operators.
We use per-type closures and other per-type data structures extensively
in the Zed run time engine.
The Apache Drill system uses this approach but the internal representation
is a sequence of a dataframe-like data structure
as compared to heterogeneous records as in Zed.

### Random Access and Scale

A criticism often raised with Zed's self-describing type system is that
the various permutations relating to present and missing fields with arbitrary
key names can create an combinatoric explosion in the number of types.  While this
is true in theory, it is actually quite manageable for the follow reasons:
* The number of types is bounded by the number of values in the input and
are local in nature type contexts, i.e., there is no global schema registry
collecting up all types that have ever existed.
* The type context can be "reset" at any point in time (e.g., ZNG include
reset markers to do so), which clears the type context, requiring
new typedefs to be established from that point forward, thus limiting the
size of a type context further.
* Zed programs are often used to "shape" data to a small number of well understood
types using a system of type operators in the Zed language.

Reset markers may also be used to establish synchronization points to provide
seekable access points in the row-based ZNG format.  The Zed lake system uses
this capability to create seek indexes for the sort key of the data objects
that comprise the lake thereby providing efficient range scans for ranges
smaller than the target object size.

### Schema Versioning

Another criticism raised with the Zed type system is that it does not have
built-in support for schema versioning.  Our answer to this is that actually
it has fantastic support, it just appears in a different form than a data engineer
might be used to compared to the features in schema registeries.

In many cases, the Zed type system simply obviates the need for schema versioning.
An application can write data of schema A (say with columns x and y),
and a column (say z) to schema A to create
schema B, and write the data and the Zed data model simply embraces both types.
And the Zed language can query all the old columns of schema A simultaneous with
the new column of schema B and it "simply works".

For more sophisticated handling of schema versioning --- e.g., adding a default
non-null value for column z when ingesting schema A --- we propose such mechanism
can be implemented on top of a system based on Z with Zed shapers.  A shaper is
simply a Zed program that transform input to output thereby allowing schema
versioning policies to be implemented as Zed shapers.

To facilitate such practices, the Zed model allows type names to be configured
with a version number (e.g., schema A from above could be named type "S.0" and
schema B named type "S.1") and the Zed language includes operators that can
refer to all the variations of this type simply as "S".  While this by itself
is not comprehensive support for schema versioning, it provides a flexible
building block for implementing schema versioning on top of Zed.

XXX give example?  Also: this is TBD in the specs and implementation

### Type Context for Columnar Form

While the streaming-oriented type context model described above has a
clean interpretation in the context of a row-based format like ZNG, it is
less clear how such a scheme could be applied to a columnar arrangement of
data as with schemes like Dremel, Parquet, or ORC.

It turns out, however, that there is a remarkably elegant way to map the
Zed data model onto columnar form using the Zed type-context abstraction.
In this approach, we follow the lake-like models of the columnar formats
above where chunks of data are partitioned into individual files or storage objects
but unlike these previous approaches which all require a single schema
to appear in a metadata section of the object,
the Zed approach instead uses a type context.

In the Zed approach called "ZST", data is written in a single pass by
organizing column vectors one per leaf of each record type as they appear
in the Zed data stream.  Since the data stream is self-describing, there
is no need to define an upfront schema first.  Instead Zed records are written to a
ZST writer that organizes the sequence of records into data columns that
are buffered in memory and written to the storage object
incrementally as column vectors, where the "root vector" is simply a
list of the type IDs of each record in sequence.  When the last record
of the data object is written, the type context is simply appended
to the object along with the seek layout of the columns
together as the "metadata" for the ZST object.
