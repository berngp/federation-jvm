[![MIT License](https://img.shields.io/github/license/apollographql/federation-jvm.svg)](LICENSE)
[![Maven Central](https://img.shields.io/maven-central/v/com.apollographql.federation/federation-graphql-java-support.svg)](https://maven-badges.herokuapp.com/maven-central/com.apollographql.federation/federation-graphql-java-support)
[![CircleCI](https://circleci.com/gh/apollographql/federation-jvm.svg?style=svg)](https://circleci.com/gh/apollographql/federation-jvm)

# Apollo Federation on the JVM

Packages published to Maven Central; release notes in [RELEASE_NOTES.md](RELEASE_NOTES.md). Note that older versions of
this package may only be available in JCenter, but we are planning to republish these versions to Maven Central.

An example of [graphql-spring-boot](https://www.graphql-java-kickstart.com/spring-boot/) microservice is available
in [spring-example](spring-example).

## Getting started

### Dependency management with Gradle

Make sure Maven Central is among your repositories:

```groovy
repositories {
    mavenCentral()
}
```

Add a dependency to `graphql-java-support`:

```groovy
dependencies {
    implementation 'com.apollographql.federation:federation-graphql-java-support:0.6.4'
}
```

### graphql-java schema transformation

`graphql-java-support` produces a `graphql.schema.GraphQLSchema` by transforming your existing schema in accordance to
the [federation specification](https://www.apollographql.com/docs/apollo-server/federation/federation-spec/). It follows
the `Builder` pattern.

Start with `com.apollographql.federation.graphqljava.Federation.transform(…)`, which can receive either:

- A `GraphQLSchema`;
- A `TypeDefinitionRegistry`, optionally with a `RuntimeWiring`;
- A String, Reader, or File declaring the schema using
  the [Schema Definition Language](https://www.apollographql.com/docs/apollo-server/essentials/schema/#schema-definition-language),
  optionally with a `RuntimeWiring`;

and returns a `SchemaTransformer`.

If your schema does not contain any types annotated with the `@key` directive, nothing else is required. You can build a
transformed `GraphQLSchema` with `SchemaTransformer#build()`, and confirm it exposes `query { _schema { sdl } }`.

Otherwise, all types annotated with `@key` will be part of the `_Entity` union type, and reachable
through `query { _entities(representations: [Any!]!) { … } }`. Before calling `SchemaTransformer#build()`, you will also
need to provide:

- A `TypeResolver` for `_Entity` using `SchemaTransformer#resolveEntityType(TypeResolver)`;
- A `DataFetcher` or `DataFetcherFactory` for `_entities`
  using `SchemaTransformer#fetchEntities(DataFetcher|DataFetcherFactory)`.

A minimal but complete example is available in
[AppConfiguration](spring-example/src/main/java/com/apollographql/federation/springexample/graphqljava/AppConfiguration.java).

### Federated tracing

To make your server generate performance traces and return them along with responses to the Apollo Gateway (which then
can send them to Apollo Graph Manager), install the `FederatedTracingInstrumentation` into your `GraphQL` object:

```java
GraphQL graphql = GraphQL.newGraphQL(graphQLSchema)
        .instrumentation(new FederatedTracingInstrumentation())
        .build();
```

It is generally desired to only create traces for requests that actually come from Apollo Gateway, as they aren't
helpful if you're connecting directly to your backend service for testing. In order
for `FederatedTracingInstrumentation` to know if the request is coming from Gateway, you need to give it access to the
HTTP request's headers, by making the `context` part of your `ExecutionInput` implement the `HTTPRequestHeaders`
interface. For example:

```java
HTTPRequestHeaders context = new HTTPRequestHeaders() {
    @Override
    public @Nullable
    String getHTTPRequestHeader(String caseInsensitiveHeaderName) {
        return myIncomingHTTPRequest.getHeader(caseInsensitiveHeaderName);
    }
};
graphql.execute(ExecutionInput.newExecutionInput(queryString).context(context));
```

Alternatively, if you are using libraries or frameworks whose `context` do not / are not able to implement
the `HTTPRequestHeaders` interface, you can construct the `FederatedTracingInstrumentation` using an `Options` object
with a predicate that takes the `ExecutionInput` object and returns a boolean indicating whether a trace should be
created for the request being processed. For example:

```java
Options options = new Options(false, (ExecutionInput executionInput) -> {
    // Apollo Gateway indicates that a trace must be computed when the HTTP header "apollo-federation-include-trace"
    // exists and has the value "ftv1".
    //
    // It is expected for you to store the above information in some manner within ExecutionInput or its context, and
    // for this function to extract that information and return whether the trace must be computed.
    if (executionInput.getContext() instanceof MySpecialExecutionContext) {
        return FEDERATED_TRACING_HEADER_VALUE.equals(
                ((MySpecialExecutionContext) executionInput.getContext()).getHeader(FEDERATED_TRACING_HEADER_NAME));
    }
    return false;
});

GraphQL graphql = GraphQL
        .newGraphQL(graphQLSchema)
        .instrumentation(new FederatedTracingInstrumentation(options))
        .build();
```
