---
id: federated-directives
title: Federated Directives
---
`graphql-kotlin` supports a number of directives that can be used to annotate a schema and direct certain behaviors.

For more details, see the [Apollo Federation Specification](https://www.apollographql.com/docs/federation/subgraph-spec/).

## `@authenticated` directive

:::info
Available since Federation v2.5
:::

```graphql
directive @authenticated on
    ENUM
  | FIELD_DEFINITION
  | INTERFACE
  | OBJECT
  | SCALAR
```

Directive that is used to indicate that the target element is accessible only to the authenticated supergraph users. For more granular access control, see the
[`@requiresScopes`[#requirescope-directive] directive usage. Refer to the [Apollo Router documentation](https://www.apollographql.com/docs/router/configuration/authorization#authenticated)
for additional details.

## `@composeDirective` directive

:::info
Available since Federation v2.1
:::

```graphql
directive @composeDirective(name: String!) repeatable on SCHEMA
```

By default, Supergraph schema excludes all custom directives. The `@composeDirective` is used to specify custom directives that should be exposed in the Supergraph schema.

In order to use composed directive, you subgraph needs to
1. contain your custom directive definition
2. import your custom directive from a corresponding link spec
3. apply `@composeDirective` with custom directive name on your schema

Example:
Given `@custom` directive we can preserve it in the Supergraph schema

```kotlin
// 1. directive definition
@GraphQLDirective(name = "custom", locations = [Introspection.DirectiveLocation.FIELD_DEFINITION])
annotation class CustomDirective

@LinkDirective(url = "https://myspecs.dev/myCustomDirective/v1.0", import = ["@custom"]) // 2. import custom directive from a spec
@ComposeDirective(name = "custom") // 3. apply @composeDirective to preserve it in the schema
class CustomSchema

class SimpleQuery {
  @CustomDirective
  fun helloWorld(): String = "Hello World"
}
```

it will generate following schema

```graphql
schema
@composeDirective(name: "@custom")
@link(import : ["@custom"], url: "https://myspecs.dev/myCustomDirective/v1.0")
@link(url : "https://specs.apollo.dev/federation/v2.5")
{
   query: Query
}

directive @custom on FIELD_DEFINITION

type Query {
  helloWorld: String! @custom
}
```

See [@composeDirective definition](https://www.apollographql.com/docs/federation/federated-types/federated-directives/#composedirective) for more information.

## `@contact` directive

```graphql
directive @contact(
  "Contact title of the subgraph owner"
  name: String!
  "URL where the subgraph's owner can be reached"
  url: String
  "Other relevant notes can be included here; supports markdown links"
  description: String
) on SCHEMA
```

Contact schema directive can be used to provide team contact information to your subgraph schema. This information is automatically parsed and displayed by Apollo Studio.
See [Subgraph Contact Information](https://www.apollographql.com/docs/studio/federated-graphs/#subgraph-contact-info) for additional details.

#### Example usage on the schema class:

```kotlin
@ContactDirective(
  name = "My Team Name",
  url = "https://myteam.slack.com/archives/teams-chat-room-url",
  description = "send urgent issues to [#oncall](https://yourteam.slack.com/archives/oncall)."
)
class MySchema
```

will generate

```graphql
schema @contact(description : "send urgent issues to [#oncall](https://yourteam.slack.com/archives/oncall).", name : "My Team Name", url : "https://myteam.slack.com/archives/teams-chat-room-url"){
  query: Query
}
```

## `@extends` directive

:::caution
**`@extends` directive is deprecated**. Federation v2 no longer requires `@extends` directive due to the smart entity type
merging. All usage of `@extends` directive should be removed from your Federation v2 schemas.
:::

```graphql
directive @extends on OBJECT | INTERFACE
```

`@extends` directive is used to represent type extensions in the schema. Native type extensions are currently
unsupported by the `graphql-kotlin` libraries. Federated extended types should have corresponding `@key` directive
defined that specifies primary key required to fetch the underlying object.

#### Example

```kotlin
@KeyDirective(FieldSet("id"))
@ExtendsDirective
class Product(@ExternalDirective val id: String) {
   fun newFunctionality(): String = "whatever"
}
```

will generate

```graphql
type Product @key(fields : "id") @extends {
  id: String! @external
  newFunctionality: String!
}
```

## `@external` directive

```graphql
directive @external on OBJECT | FIELD_DEFINITION
```

The `@external` directive is used to mark a field as owned by another service. This allows service A to use fields from
service B while also knowing at runtime the types of that field. All the external fields should either be referenced from
the `@key`, `@requires` or `@provides` directives field sets.

Due to the smart merging of entity types, Federation v2 no longer requires `@external` directive on `@key` fields and can
be safely omitted from the schema. `@external` directive is only required on fields referenced by the `@requires` and
`@provides` directive.

#### Example

```kotlin
@KeyDirective(FieldSet("id"))
class Product(val id: String) {
  @ExternalDirective
  var externalField: String by Delegates.notNull()

  @RequiresDirective(FieldSet("externalField"))
  fun newFunctionality(): String { ... }
}
```

will generate

```graphql
type Product @key(fields : "id") {
  externalField: String! @external
  id: String!
  newFunctionality: String!
}
```

## `@inaccessible` directive

:::info
Available since Federation v2.0
:::

```graphql
directive @inaccessible on FIELD_DEFINITION
    | OBJECT
    | INTERFACE
    | UNION
    | ENUM
    | ENUM_VALUE
    | SCALAR
    | INPUT_OBJECT
    | INPUT_FIELD_DEFINITION
    | ARGUMENT_DEFINITION
```

Inaccessible directive marks location within schema as inaccessible from the GraphQL Gateway. While `@inaccessible` fields are not exposed by the gateway to the clients,
they are still available for query plans and can be referenced from `@key` and `@requires` directives. This allows you to not expose sensitive fields to your clients but
still make them available for computations. Inaccessible can also be used to incrementally add schema elements (e.g. fields) to multiple subgraphs without breaking composition.

See [@inaccessible specification](https://specs.apollo.dev/inaccessible/v0.2) for additional details.

:::caution
Location within schema will be inaccessible from the GraphQL Gateway as long as **ANY** of the subgraphs marks that location as `@inacessible`.
:::

#### Example

```kotlin
class Product(
  val id: String,
  @InaccessibleDirective
  val secret: String
)
```

will be generated by the subgraph as

```graphql
type Product {
  id: String!
  secret: String! @inaccessible
}
```

but will be exposed on the GraphQL Gateway as

```graphql
type Product {
  id: String!
}
```

## `@interfaceObject` directive

:::info
Available since Federation v2.3
:::

```graphql
directive @interfaceObject on OBJECT
```

This directive provides meta information to the router that this entity type defined within this subgraph is an interface in the supergraph. This allows you to extend functionality
of an interface across the supergraph without having to implement (or even be aware of) all its implementing types.

Example:
Given an interface that is defined somewhere in our supergraph

```graphql
interface Product @key(fields: "id") {
  id: ID!
  description: String
}

type Book implements Product @key(fields: "id") {
  id: ID!
  description: String
  pages: Int!
}

type Movie implements Product @key(fields: "id") {
  id: ID!
  description: String
  duration: Int!
}
```

We can extend `Product` entity in our subgraph and a new field directly to it. This will result in making this new field available to ALL implementing types.

```kotlin
@InterfaceObjectDirective
@KeyDirective(fields = FieldSet("id"))
data class Product(val id: ID) {
    fun reviews(): List<Review> = TODO()
}
```

Which generates the following subgraph schema

```graphql
type Product @key(fields: "id") @interfaceObject {
  id: ID!
  reviews: [Review!]!
}
```

## `@key` directive

```graphql
directive @key(fields: FieldSet!, resolvable: Boolean = true) repeatable on OBJECT | INTERFACE
```

The `@key` directive is used to indicate a combination of fields that can be used to uniquely identify and fetch an
object or interface. The specified field set can represent single field (e.g. `"id"`), multiple fields (e.g. `"id name"`) or
nested selection sets (e.g. `"id user { name }"`). Multiple keys can be specified on a target type.

Key directives should be specified on all entities (objects that can resolve its fields across multiple subgraphs). Key
fields specified in the directive field set should correspond to a valid field on the underlying GraphQL interface/object.

#### Basic Example

```kotlin
@KeyDirective(FieldSet("id"))
@KeyDirective(FieldSet("upc"))
class Product(val id: String, val upc: String, val name: String)
```

will generate

```graphql
type Product @key(fields: "id") @key(fields: "upc") {
  id: String!
  name: String!
  upc: String!
}
```

#### Referencing External Entities

Entity types can be referenced from other subgraphs without contributing any additional fields, i.e. we can update type within our schema with a reference to a federated type. In order to generate
a valid schema, we need to define **stub** for federated entity that contains only key fields and also mark it as not resolvable within our subgraph. For example, if we have `Review` entity defined
in our supergraph, we can reference it in our product schema using following code

```kotlin
@KeyDirective(fields = FieldSet("id"))
class Product(val id: String, val name: String, val reviews: List<Review>)

// review stub referencing just the key fields
@KeyDirective(fields = FieldSet("id"), resolvable = false)
class Review(val id: String)
```

which will generate

```graphql
type Product @key(fields: "id") {
  id: String!
  name: String!
  reviews: [Review!]!
}

type Review @key(fields: "id", resolvable: false) {
  id: String!
}
```

This allows end users to query GraphQL Gateway for any product review fields and they will be resolved by calling the appropriate subgraph.

## `@link` directive

:::info
Available since Federation v2.0
:::

:::caution
While both custom namespace (`as`) and `import` arguments are optional in the schema definition, due to [#1830](https://github.com/ExpediaGroup/graphql-kotlin/issues/1830)
we currently always require those values to be explicitly provided.
:::

```graphql
directive @link(url: String!, as: String, import: [Import]) repeatable on SCHEMA
scalar Import
```

The `@link` directive links definitions within the document to external schemas. See [@link specification](https://specs.apollo.dev/link/v1.0) for details.

External schemas are identified by their `url`, which ends with a name and version with the following format: `{NAME}/v{MAJOR}.{MINOR}`,
e.g. `url = "https://specs.apollo.dev/federation/v2.5"`.

External types are associated with the target specification by annotating it with `@LinkedSpec` meta annotation. External
types defined in the specification will be automatically namespaced (prefixed with `{NAME}__`) unless they are explicitly
imported. Namespace should default to the specification name from the imported spec url. Custom namespace can be provided
by specifying `as` argument value.

External types can be imported using the same name or can be aliased to some custom name.

```kotlin
@LinkDirective(`as` = "custom", imports = [LinkImport(name = "@foo"), LinkImport(name = "@bar", `as` = "@myBar")], url = "https://myspecs.dev/custom/v1.0")
class MySchema
```

This will generate following schema:

```graphql
schema @link(as: "custom", import : ["@foo", { name: "@bar", as: "@myBar" }], url : "https://myspecs.dev/custom/v1.0") {
    query: Query
}
```

### `@LinkedSpec` annotation

When importing custom specifications, we need to be able to identify whether given element is part of the referenced specification.
`@LinkedSpec` is a meta annotation that is used to indicate that given directive/type is associated with imported `@link` specification.

In order to ensure consistent behavior, `@LinkedSpec` value have to match default specification name as it appears in the
imported url and not the aliased value.

Example usage:

```
@LinkedSpec("custom")
@GraphQLDirective(
    name = "foo",
    locations = [DirectiveLocation.FIELD_DEFINITION]
)
annotation class Foo
```

In the example above, we specify that `@foo` directive is part of the `custom` specification. We can then reference `@foo`
in the `@link` specification imports

```graphql
schema @link(as: "custom", import : ["@foo"], url : "https://myspecs.dev/custom/v1.0") {
    query: Query
}

directive @foo on FIELD_DEFINITION
```

If we don't import the directive, then it will automatically namespaced to the spec

```graphql
schema @link(as: "custom", url : "https://myspecs.dev/custom/v1.0") {
    query: Query
}

directive @custom__foo on FIELD_DEFINITION
```

## `@override` directive

:::info
Available since Federation v2.0
:::

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive is used to indicate that the current subgraph is taking responsibility for resolving the marked field away from the subgraph specified in the `from` argument,
i.e. it is used for migrating a field from one subgraph to another. Name of the subgraph to be overriden has to match the name of the subgraph that was used to publish their schema. See
[Publishing schema to Apollo Studio](https://www.apollographql.com/docs/rover/subgraphs/#publishing-a-subgraph-schema-to-apollo-studio) for additional details.

:::caution
Only one subgraph can `@override` any given field. If multiple subgraphs attempt to `@override` the same field, a composition error occurs.
:::

#### Example

Given `SubgraphA`

```graphql
type Product @key(fields: "id") {
    id: String!
    description: String!
}
```

We can override gateway `description` field resolution to resolve it in the `SubgraphB`

```graphql
type Product @key(fields: "id") {
    id: String!
    name: String!
    description: String! @override(from: "SubgraphA")
}
```

## progressive `@override` directive

:::info
Available since Federation v2.0
:::

```graphql
directive @override(from: String!, label: String) on FIELD_DEFINITION
```

The progressive @override feature enables the gradual, progressive deployment of a subgraph with an @override field. As a subgraph developer, you can customize the percentage of traffic that the overriding and overridden subgraphs each resolve for a field. Please read mmore about the [progressive @override feature](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/reference/directives) in the Apollo Router documentation.

:::caution
It is an Enterprise feature of the GraphOS Router and requires an organization with a GraphOS Enterprise plan.
Only one subgraph can `@override` any given field. If multiple subgraphs attempt to `@override` the same field, a composition error occurs.
:::

#### Example

Given `SubgraphA`:

```graphql
type Product @key(fields: "id") {
    id: String!
    description: String!
}
```

We can override the `description` field resolution in `SubgraphB`:

```graphql
type Product @key(fields: "id") {
    id: String!
    name: String!
    description: String! @override(from: "SubgraphA", label: "percent(1)")
}
```

## `@policy` directive

:::info
Available since Federation v2.6
:::

```graphql
directive @policy(policies: [[Policy!]!]!) on
    ENUM
  | FIELD_DEFINITION
  | INTERFACE
  | OBJECT
  | SCALAR
```

Directive that is used to indicate that access to the target element is restricted based on authorization policies that are evaluated in a Rhai script or coprocessor. Refer to the
[Apollo Router documentation](https://www.apollographql.com/docs/router/configuration/authorization#policy) for additional details.


## `@provides` directive

```graphql
directive @provides(fields: FieldSet!) on FIELD_DEFINITION
```

The `@provides` directive is a router optimization hint specifying field set that can be resolved locally at the given subgraph through this particular query path. This allows you to
expose only a subset of fields from the underlying entity type to be selectable from the federated schema without the need to call other subgraphs. Provided fields specified in the
directive field set should correspond to a valid field on the underlying GraphQL interface/object type. `@provides` directive can only be used on fields returning entities.

:::info
Federation v2 does not require `@provides` directive if field can **always** be resolved locally. `@provides` should be omitted in this situation.
:::

#### Example 1:

We might want to expose only name of the user that submitted a review.

```kotlin
@KeyDirective(FieldSet("id"))
class Review(val id: String) {
  @ProvidesDirective(FieldSet("name"))
  fun user(): User = getUserByReviewId(id)
}

@KeyDirective(FieldSet("userId"))
class User(
  val userId: String,
  @ExternalDirective val name: String
)
```

will generate

```graphql
type Review @key(fields : "id") {
  id: String!
  user: User! @provides(fields : "name")
}

type User @key(fields : "userId") {
  userId: String!
  name: String! @external
}
```

#### Example 2:

Within our service, one of the queries could resolve all fields locally while other requires resolution from other subgraph

```graphql
type Query {
  remoteResolution: Foo
  localOnly: Foo @provides("baz")
}

type Foo @key("id") {
  id: ID!
  bar: Bar
  baz: Baz @external
}
```

In the example above, if user selects `baz` field, it will be resolved locally from `localOnly` query but will require another subgraph invocation from `remoteResolution` query.

## `@requires` directive

```graphql
directive @requires(fields: FieldSet!) on FIELD_DEFINITON
```

The `@requires` directive is used to specify external (provided by other subgraphs) entity fields that are needed to resolve target field. It is used to develop a query plan where
the required fields may not be needed by the client, but the service may need additional information from other subgraphs. Required fields specified in the directive field set should
correspond to a valid field on the underlying GraphQL interface/object and should be instrumented with `@external` directive.

All the leaf fields from the specified in the `@requires` selection set have to be marked as `@external` OR any of the parent fields on the path to the leaf is marked as `@external`.

Fields specified in the `@requires` directive will only be specified in the queries that reference those fields. This is problematic for Kotlin as the non-nullable primitive properties
have to be initialized when they are declared. Simplest workaround for this problem is to initialize the underlying property to some default value (e.g. null) that will be used if
it is not specified. This approach might become problematic though as it might be impossible to determine whether fields was initialized with the default value or the invalid/default
value was provided by the federated query. Another potential workaround is to rely on delegation to initialize the property after the object gets created. This will ensure that exception
will be thrown if queries attempt to resolve fields that reference the uninitialized property.

#### Example

```kotlin
@KeyDirective(FieldSet("id"))
class Product(val id: String) {
  @ExternalDirective
  var weight: Double by Delegates.notNull()

  @RequiresDirective(FieldSet("weight"))
  fun shippingCost(): String { ... }
}
```

will generate

```graphql
type Product @key(fields : "id") {
  id: String!
  shippingCost: String! @requires(fields : "weight")
  weight: Float! @external
}
```

## `@requiresScopes` directive

:::info
Available since Federation v2.5
:::

```graphql
directive @requiresScopes(scopes: [[Scope!]!]!) on
    ENUM
  | FIELD_DEFINITION
  | INTERFACE
  | OBJECT
  | SCALAR
```

Directive that is used to indicate that the target element is accessible only to the authenticated supergraph users with the appropriate JWT scopes. Refer to the
[Apollo Router documentation](https://www.apollographql.com/docs/router/configuration/authorization#requiresscopes) for additional details.

## `@shareable` directive

:::info
Available since Federation v2.0
:::

```graphql
directive @shareable repeatable on FIELD_DEFINITION | OBJECT
```

Shareable directive indicates that given object and/or field can be resolved by multiple subgraphs. If an object is marked as `@shareable` then all its fields are automatically shareable without the
need for explicitly marking them with `@shareable` directive. All fields referenced from `@key` directive are automatically shareable as well.

:::caution
Objects/fields have to specify same shareability (i.e. `@shareable` or not) mode across ALL subgraphs.
:::

#### Example

```graphql
type Product @key(fields: "id") {
  id: ID!                           # shareable because id is a key field
  name: String                      # non-shareable
  description: String @shareable    # shareable
}

type User @key(fields: "email") @shareable {
  email: String                    # shareable because User is marked shareable
  name: String                     # shareable because User is marked shareable
}
```

## `@tag` directive

```graphql
directive @tag(name: String!) repeatable on FIELD_DEFINITION
    | OBJECT
    | INTERFACE
    | UNION
    | ARGUMENT_DEFINITION
    | SCALAR
    | ENUM
    | ENUM_VALUE
    | INPUT_OBJECT
    | INPUT_FIELD_DEFINITION
```

Tag directive allows users to annotate fields and types with additional metadata information. Used by [Apollo Contracts](https://www.apollographql.com/docs/studio/contracts/) to expose
different graph variants to different customers. See [@tag specification](https://specs.apollo.dev/tag/v0.2/) for details.

#### Example

```graphql
type Product @tag(name: "MyCustomTag") {
    id: String!
    name: String!
}
```

:::caution
Apollo Contracts behave slightly differently depending on which version of Apollo Federation your graph uses (1 or 2). See [documentation](https://www.apollographql.com/docs/studio/contracts/#federation-1-limitations)
for details.
:::
