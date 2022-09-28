![Sangria Logo](https://sangria-graphql.github.io/assets/img/sangria-logo.svg)

# sangria-federated

![Continuous Integration](https://github.com/sangria-graphql/sangria-federated/workflows/Continuous%20Integration/badge.svg)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.sangria-graphql/sangria-federated_2.13/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.sangria-graphql/sangria-federated_2.13)
[![License](http://img.shields.io/:license-Apache%202-brightgreen.svg)](http://www.apache.org/licenses/LICENSE-2.0.txt)
[![Scaladocs](https://www.javadoc.io/badge/org.sangria-graphql/sangria-federated_2.13.svg?label=docs)](https://www.javadoc.io/doc/org.sangria-graphql/sangria-federated_2.13)
[![Join the chat at https://gitter.im/sangria-graphql/sangria](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/sangria-graphql/sangria?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

**sangria-federated** is a library that allows sangria users to implement services that adhere to [Apollo's Federation Specification](https://www.apollographql.com/docs/federation/federation-spec/), and can be used as part of a federated data graph.

SBT Configuration:

```scala
libraryDependencies += "org.sangria-graphql" %% "sangria-federated" % "<latest version>"
```

## How does it work?

The library adds [Apollo's Federation Specification](https://www.apollographql.com/docs/federation/federation-spec/) on top of the provided sangria graphql schema.

To make it possible to use `_Any` as a scalar, the library upgrades the used marshaller.

## implementation of the Apollo Federation Subgraph Compatibility

A good example showing all different features is the [sangria implementation of the Apollo Federation Subgraph Compatibility](https://github.com/apollographql/apollo-federation-subgraph-compatibility/tree/main/implementations/sangria).

## Example how to use it

All the code of this example is available [here](./example).

To be able to communicate with [Apollo's federation gateway](https://www.apollographql.com/docs/federation/gateway/), the graphql sangria service should be using both the federated schema and unmarshaller.

As an example, let's consider the following services:
- a review service provides a subgraph for review
- a state service provides a subgraph for states. This state is used by reviews.
- both subgraphs are composed into one supergraph that is the only graph to use by users. This supergraph is one GraphQ schema that exposes reviews and states. 

### The state service

The state service defines the state entity annotated with `@key("id")`.

For each entity, we need to define an [entity resolver](https://www.apollographql.com/docs/federation/entities/#resolving).

[Implementation of State](./example/state/src/main/scala/state/State.scala).

The entity resolver implements:
- the deserialization of the fields in `_Any` object to the EntityArg.
- how to fetch the proper Entity (in our case `State`) based on the EntityArg.

The [GraphQL query type defines a simple query](./example/state/src/main/scala/state/StateAPI.scala).

In the definition of the GraphQL server, we federate the Query type and the unmarshaller while supplying the entity resolvers.
Then, we use both the federated schema and unmarshaller as arguments for the server:

```scala
def graphQL[F[_]: Async]: GraphQL[F, StateService] = {
  val (schema, um) = Federation.federate[StateService, Any, Json](
    Schema(StateAPI.Query),
    sangria.marshalling.circe.CirceInputUnmarshaller,
    stateResolver)

  GraphQL(schema, env.pure[F])(Async[F], um)
}
```

The GraphQL server uses the provided schema and unmarshaller as arguments for the sangria executor:
[implementation](./example/common/src/main/scala/common/GraphQL.scala)
  
### The review service

- The review service defines the `Review` type, which has a reference to the `State` type.

  [implementation of Review](./example/review/src/main/scala/review/Review.scala)

- As `State` is implemented by the state service, we don't need to implement the whole state in the review service.
Instead, for each entity implemented by another service, a [stub type](https://www.apollographql.com/docs/federation/entities/#referencing) should be created (containing just the minimal information that will allow to reference the entity).

  [implementation of the stub type State](./example/review/src/main/scala/review/State.scala)
(notice the usage of the @external directive).

- In the end, the same code used to federate the state service is used to federate the review service.

### Federation service

The sangria GraphQL services endpoints can now be configured in the ```serviceList``` of [Apollo's Gatewqay](https://www.apollographql.com/docs/federation/gateway/#setup) as follows:
```
const gateway = new ApolloGateway({
  serviceList: [
    { name: 'states', url: 'http://localhost:9081/api/graphql'},
    { name: 'reviews', url: 'http://localhost:9082/api/graphql'}
  ],
  debug: true
})
```

## Caution 🚨🚨

- **This is a technology preview. We are actively working on it and cannot promise a stable API yet**.
- The library upgrades the marshaller to map values scalars (e.g. json objects as scalars). This can lead to security issues as discussed [here](http://www.petecorey.com/blog/2017/06/12/graphql-nosql-injection-through-json-types/).

## Contribute

Contributions are warmly desired 🤗. Please follow the standard process of forking the repo and making PRs 🤓

## License

**sangria-federated** is licensed under [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
