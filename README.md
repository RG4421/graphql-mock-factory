# graphql-mock-factory

A JavaScript library to easily generate mock GraphQL responses. This is useful to write robust tests with minimum boilerplate. It is similar to the mocking functionality of [graphql-tools](https://www.apollographql.com/docs/graphql-tools/mocking/) but allows to precisely customize the response of each query.

## Design goals

- Focus on the use case of generating precise mock data in tests.
- Make it completely obvious how to write a mock function.
- Deep merge mock values, objects and lists in a completely unambiguous manner.
- Provide helpful validations and error messages.
- Encourage teams to write and share sensible mock functions.

## Installation

[`graphql`](https://github.com/graphql/graphql-js) is a peer-dependency:
```sh
npm install graphql-mock-factory graphql
```

## Usage

### Building a mock server with `mockServer`

Define a mock server using your schema definition string.

```javascript
import { mockList, mockServer } from 'graphql-mock-factory';

// Replace with your schema definition string.
// It can be automatically generated by any GraphQL server.
const schemaDefinition = `
  type Query {
    viewer: User
  }

  type User {
    firstName: String
    lastName: String
  }
`;

// Initialize your mocked server.
const mockedServer = mockServer(schemaDefinition);

const query = `
  query {
    viewer {
      firstName
      lastName
    }
  }
`;

mockedServer(query);
// ->
// { data: { viewer: { 
//    firstName: 'lorem ipsum dolor sit amet', 
//    lastName: 'sed do eiusmod tempor incididunt' } } }
```

### Defining mock functions

By default, the server will generate valid random values for most field types. While this is helpful to quickly get started, it can be confusing to work with gibberish values. That's why we recommend you define realistic looking mock functions that you share with your team.

```javascript
// Here we use `faker-js` to generate realistic mock data.
const mocks = {
  User: {
    firstName: () => faker.name.firstName(),
    lastName: () => faker.name.findName(),
  }
};

const mockedServer = mockServer(schemaDefinition, mocks);

...

mockedServer(query);
// ->
// { data: { viewer: { 
//    firstName: 'Jason', lastName: 'Wilde' } } }
```

<details>
  <summary>How to disable default mock functions</summary>
  <p>

  ```js
  // In order to help you or your team define realistic mock functions 
  // progressively, you can disable all default mock functions.
  // After that, an error will be thrown if a queried field is not 
  // associated with a mock function. In other words, you won't have 
  // to define a mock function for a field until it is queried for
  // the first time.

  ...

  // Set `getMocks` (ie 3rd parameter) to null.
  const mockedServer = mockServer(schemaDefinition, {}, null);

  ...

  mockedServer(query);
  // Throws an error ->
  // There is no base mock for 'Viewer.firstName'. 
  // All queried fields must have a mock function.
  // ... OMITTED ...

  const mocks = {
    User: {
      firstName: () => faker.name.firstName(),
      lastName: () => faker.name.findName(),
    }
  };

  // Set `getMocks` (ie 3rd parameter) to null.
  const mockedServer = mockServer(schemaDefinition, mocks, null);

  ...

  mockedServer(query);
  // ->
  // { data: { viewer: { 
  //    firstName: 'Jason', lastName: 'Wilde' } } }
  ```
  </p>
</details>


### Overriding mocked values with `mockOverride`

When writing tests, you usually want to customize the server responses so you can test a specific case. You can easily do this by passing a `mockOverride` object.

```js
// Here we specify `viewer.firstName`. 
// The other response field (ie `viewer.lastName`) will be generated 
// by its corresponding mock function (ie `User.lastName`).

mockedServer(query, mocks, 
  // The `mockOverride` object is the 3rd parameter
  { 
    viewer: { 
      firstName: 'Oscar',
    }
  },
);
// ->
// { data: { 
//    viewer: { firstName: 'Oscar', lastName: 'Smith' } } }
```

All the values that are not specified in the `mockOverride` object will be generated by the mock functions.

<details>
  <summary>Example with an aliased field</summary>
  <p>

  ```js
  // Here we only specify `viewer.aliasedName`. 
  // `viewer.firstName` will be generated by its corresponding 
  // mock function (ie `User.firstName`).

  mockedServer(`
    query {
      viewer { 
        firstName 
        aliasedName: firstName
      }
    }`,
    {}, 
    { 
      viewer: { 
        firstName: 'Oscar' 
      }
    },
  );
  // ->
  // { data: { viewer: 
  //   { firstName: 'Oscar', aliasedName: 'Eryn' } } }
  ```
  </p>
</details>

<details>
  <summary>Example with nested fields</summary>
  <p>

  ```js
  // Here we only specify the `viewer.firstName`. 
  // `viewer.parent.firstName` will be generated by its corresponding 
  // mock function (ie `User.firstName`).

  const schemaDefinition = `
    ...

    type User {
      firstName: String
      parent: User
    }
  `;

  ...

  mockedServer(`
    query {
      viewer { 
        firstName
        parent {
          firstName
        }
      }
    }`,
    {}, 
    { 
      viewer: { 
        firstName: 'Oscar' 
      }
    },
  );
  // ->
  // { data: { viewer: 
  //   { firstName: 'Oscar', 
  //     parent: { firstName: 'Krystina' } } } }
  ```

  </p>
</details>

<details>
  <summary>Example with `null` and `undefined`</summary>
  <p>

  ```js
  // `undefined` is equivalent to not specifying a value.
  // `null` always nullifies the field.

  const schemaDefinition = `
    ...

    type User {
      firstName: String
      parent: User
    }
  `;

  ...

  mockedServer(`
    query {
      viewer { 
        firstName
        parent {
          firstName
        }
      }
    }`,
    {}, 
    { 
      viewer: {
        firstName: undefined,
        parent: null 
      }
    },
  );
  // ->
  // { data: { viewer: 
  //   { firstName: 'Raegan', parent: null } } }
  ```
  </p>
</details>

### Accessing field arguments

Field arguments can be accessed in mock functions and in `mockOverride` objects.

<details>
  <summary>Example in mock function</summary>
  <p>

  ```js
  // Field arguments are passed as named parameters to mock functions.

  const schemaDefinition = `
    type Query {
      echo(input: String): String
    }
  `;

  const mocks = {
    Query: {
      echo: ({input}) => `echo: ${input}`,
    }
  };

  ...

  mockedServer(`
    query {
      echo(input: "hello")
    }`
  );
  // ->
  // { data: { 
  //    echo: 'echo: hello' } }
  ```
  </p>
</details>

<details>
  <summary>Example in `mockOverride`</summary>
  <p>

  ```js
  // When `mockOverride` object contains functions, field arguments
  // are passed in as named parameters to mock functions.

  const schemaDefinition = `
    type Query {
      echo(input: String): String
    }
  `;

  mockedServer(`
    query {
      echo(input: "hello")
    }`,
    {},
    {
      echo: ({input}) => `repeat: ${input}`,
    }
  );
  // ->
  // { data: { 
  //    echo: 'repeat: hello' } }
  ```
  </p>
</details>

### Mocking nested fields

Objects returned by mock functions are deep merged with the return values of the mock functions of the nested fields.

<details>
  <summary>Example with one level of nesting</summary>
  <p>

  ```js
  // `searchUser.firstName` is generated by the mock function
  // of `Query.searchUser` because it returned an object with
  // a value for `firstName`. 
  
  // `searchUser.lastName` is generated by the mock function  
  // of `User.lastName` because the mock function of 
  // `Query.searchUser` returned an object that did not include 
  // a value for `lastName`.

  const schemaDefinition = `
    type Query {
      searchUser(name: String): User
    }

    type User {
      firstName: String
      lastName: String
    }
  `;

  const mocks = {
    Query: {
      searchUser: ({name}) => ({
        firstName: `${name}`
      }),
    },
    User: {
      firstName: () => faker.name.firstName(),
      lastName: () => faker.name.findName(),
    },
  };

  ...

  mockedServer(`
    query {
      searchUser(name: "Oscar") {
        firstName
        lastName
      }
    }
  `)
  // ->
  // { data: 
  //   { searchUser: 
  //      { firstName: "Oscar", lastName: "Schlosser" } } }
  ```
  </p>
</details>

<details>
  <summary>Example with multiple levels of nesting (less common)</summary>
  <p>

  ```js
  // `searchUser.address.country` is generated by the mock 
  // function for `Query.searchUser` instead of the mock function 
  // for `Address.country`.
  
  // `searchUser.firstName` is generated by the mock function  
  //  for `User.firstName` because the mock function for 
  // `Query.searchUser` did not include a value for `firstName`.

  const schemaDefinition = `
    type Query {
      searchUser(country: String): User
    }

    type User {
      firstName: String
      address: Address
    }

    type Address {
      country: String
    }
  `;

  const mocks = {
    Query: {
      searchUser: ({country}) => ({
        address: {
          country: `${country}`,
        },
      }),
    },
    User: {
      firstName: () => faker.name.firstName(),
    },
    Address: {
      country: () => faker.address.country(),
    },
  };

  ...

  mockedServer(`
    query {
      searchUser(name: "France") {
        firstName
        address {
          country
        }
      }
    }
  `)
  // ->
  // { data: 
  //   { searchUser: 
  //      { firstName: "Sam",  
  //        address: { country: "France" } } }
  ```
  </p>
</details>

### Mocking lists with `mockList`

Lists must be mocked with the `mockList` function.
<details>
  <summary>The first parameter sets the size of the list</summary>
  <p>

  ```js
  const schemaDefinition = `
    ...

    type User {
      name: String
      friends: [User]
    }
  `;

  const mocks = {
    User: {
      // Generates a list with 2 items.
      friends: mockList(2),
      name: () => faker.name.firstName(),
    }
  };

  mockedServer(`
    viewer {
      friends {
        name
      }
    }`
  );
  // ->
  // { data: 
  //   { viewer: 
  //      { friends: [ { name: 'Nikki' }, { name: 'Doug' } ] } } }
  ```
  </p>
</details>

<details>
  <summary>List items can customized with an optional function</summary>
  <p>

  ```js
  // The function will be called for each list item with the field arguments 
  // and the index of the item. 
  // The return values of the function will be deep merged with the results 
  // of the mock functions of the nested fields.

  const schemaDefinition = `
    type User {
      name: String
      friends(pageNumber: Int): [User]
    }

    ...
  `;

  const mocks = {
    User: {
      name: () => faker.name.firstName(),
      friends: mockList(2, ({pageNumber}, index) => ({
        name: `Friend #${index} - Page #${pageNumber}`,
      })),
    }
  };

  ...

  mockedServer(`
    viewer {
      friends(pageNumber: 0) {
        name
      }
    }`
  );
  // ->
  // { data: 
  //   { viewer: 
  //      { friends: [ { name: 'Friend #0 - Page #0' }, { name: 'Friend #1 - Page #0' } ] } } }
  ```
  </p>
</details>

A `mockList` can be overriden in 2 ways.

<details>
  <summary>Overriding `mockList` with an array</summary>
  <p>

  ```js
  // In most cases, a `mockList` is overriden with an array:
  // - The size of the array determines the length of the final array. 
  // - The item objects will be deep merged with the return value of 
  //   the `mockList` function if provided.

  import { mockList, mockServer } from 'graphql-mock-factory';

  ...

  const mocks = {
    User: {
      name: () => faker.name.firstName(),
      friends: mockList(2, ({}, index) => ({name: `Friend #${index}`})),
    }
  };

  ...

  // Here we specify that the list will be of size 3
  mockedServer(query, {}, {
    viewer: {
      friends: [
        // An empty object means the list item will be fully generated by 
        // its corresponding mock functions.
        {},
        // A partial object will be deep merged with the result of the 
        // corresponding mock functions.
        {'name': 'Oscar'}, 
        // `null` means the list item will be null. 
        null
      ],
    },
  });
  // ->
  // { data:
  //   { viewer:
  //      { friends: [ { name: 'Friend #0' }, { name: 'Oscar' }, null ] } } }
  ```
  </p>
</details>

<details>
  <summary>Overriding with another `mockList` (less common)</summary>
  <p>

  ```js
  // In some cases it might be more convenient to override a 
  // `mockList` with another `mockList`:
  // - The size of the `mockList` override determines the length 
  //   of the final array.
  // - The return values of the optional `mockList` functions 
  //   will be deep merged. 

  const mocks = {
    User: {
      name: () => faker.name.firstName(),
      friends: mockList(2, ({}, index) => ({name: `Friend #${index}`})),
    }
  };

  const query = `
    query {
      viewer {
        friends {
          name
          aliasedName1: name
          aliasedName2: name
        }
      }
    }
  `

  mockedServer(query, {}, {
    viewer: {
      // The list will be of size 1.
      friends: mockList(1, () => ({
        // This will be deep merged with the result of the other `mockList` function.
        aliasedName1: 'Aliased Name 1',
      })),
    },
  });
  // ->
  // { data:
  //   { viewer:
  //      { friends: [ { 
  //        name: 'Friend #0', aliasedName1: 'Aliased Name 1', aliasedName2: 'Katy' } ] } } }
  ```
  </p>
</details>

### Simulating server errors

Server errors can be simulated by including `Error` instances in `mockOverride` objects.

<details>
  <summary>Example</summary>
  <p>

  ```js
  mockedServer(`
    query {
      viewer {
        firstName
        lastName
      }
    }`, {}, {
    viewer: {
      // Errors shall not be thrown.
      firstName: Error('Could not fetch firstName.'),
    },
  });
  // ->
  // { errors: 
  //    [ { Error: Could not fetch firstName.
  //        ... OMITTED ...
  //        message: 'Could not fetch error',
  //        locations: [ { line: 4, column: 7 } ],
  //        path: [ 'viewer', 'firstName' ] } ],
  //   data:
  //    { viewer:
  //      { firstName: null, lastName: 'Gold' } } }
  ```
  </p>
</details>

### Mocking Relay connections with `mockConnection`

`mockConnection` is a convenience helper function to mock Relay connections.

<details>
  <summary>By default, it returns the number of requested nodes</summary>
  <p>

  ```js
  // By default, `mockConnection` returns the number of requested nodes.
  // Note that `hasNextPage` and `hasPreviousPage` behave as expected.

  import { mockServer, mockConnection, getRelayMock } from 'graphql-mock-factory';

  ...

  const schemaDefinition = `
    type User implements Node {
      id: ID!
      name: String
      friends(before: String, after: String, first: Int, last: Int): UserConnection
    }

    ...
  `;

  const mocks = {
    User: {
      friends: mockConnection(),
      name: () => faker.name.firstName(),
    }
  };

  const mockedServer = mockServer(
    schemaDefinition,
    mocks,
    // Adds mock functions for un-essential Relay fields
    [getRelayMock],
  );

  mockedServer(`
    query {
      viewer {
        friends(first: 2) {
          edges {
            node {
              name
            }
            cursor
          }
          pageInfo {
            hasNextPage
            hasPreviousPage
          }
        }
      }
    }`,
  );
  // ->
  // { data:
  //  { viewer:
  //     { friends:
  //        { edges:
  //           [ { node: { name: 'Milford' }, cursor: 'cursor_0' },
  //             { node: { name: 'Bennie' }, cursor: 'cursor_1' } ],
  //          pageInfo: { hasNextPage: true, hasPreviousPage: false } } } } }
  ```
  </p>
</details>

<details>
  <summary>The collection size can be set with `maxSize` option</summary>
  <p>

  ```js
  // The optional `maxSize` named parameter limits the number of returned items.
  // Note that `hasNextPage` and `hasPreviousPage` behave as expected.

  const mocks = {
    User: {
      // At most 1 item will be returned
      friends: mockConnection({maxSize: 1}),
      name: () => faker.name.firstName(),
    }
  };

  ...

  mockedServer(`
    query {
      viewer {
        friends(first: 2) {
          edges {
            node {
              name
            }
          }
          pageInfo {
            hasNextPage
            hasPreviousPage
          }
        }
      }
    }`,
  );
  // ->
  // { data:
  //  { viewer:
  //     { friends:
  //        { edges:
  //           [ { node: { name: 'Milford' } } ],
  //          pageInfo: { hasNextPage: false, hasPreviousPage: false } } } } }
  ```
  </p>
</details>

<details>
  <summary>Nodes can be customized with `nodeMock` option</summary>
  <p>

  ```js
  // The optional `nodeMock` named function customizes the requested nodes.
  // Like `mockList`, it is called with the connection field arguments and 
  // the index of the node.
  
  const mocks = {
    User: {
      friends: mockConnection({nodeMock: ({ first, last }, index) => ({
        name: `Friend ${index} / ${first || last}`,
      })}),
      name: () => faker.name.firstName(),
    }
  };

  ...

  mockedServer(`
    query {
      viewer {
        friends(first: 2) {
          edges {
            node {
              name
            }
          }
          pageInfo {
            hasNextPage
            hasPreviousPage
          }
        }
      }
    }`,
  );
  // ->
  // { data:
  //  { viewer:
  //     { friends:
  //        { edges:
  //           [ { node: { name: 'Friend 0 / 2' } },
  //             { node: { name: 'Friend 1 / 2' } } ] } } } }
  ```
  </p>
</details>

<details>
  <summary>Field arguments are validated</summary>
  <p>

  ```js
  // An error is returned if the arguments do not conform the the Relay specs.

  mockedServer(`
    query {
      viewer {
        friends(first: -2) {
          edges {
            node {
              name
            }
          }
        }
      }
    }`,
  );
  // ->
  // { errors:
  //    [ { Error: First and last cannot be negative.
  //        ... OMITTED ...
  //        message: 'First and last cannot be negative.',
  //        locations: [ { line: 4, column: 7 } ],
  //        path: [ 'viewer', 'friends' ] } ],
  //   data: { viewer: { friends: null } } }
  ```
  </p>
</details>

`mockConnection` can be overriden like `mockList` can.

<details>
  <summary>Overriding with an array</summary>
  <p>

  ```js
  // Like `mockList`, it can be overriden with an array.
  // This is because `mockConnection` is simply a wrapper around `mockList`.

  const mocks = {
    User: {
      friends: mockConnection(),
      name: () => faker.name.firstName(),
    }
  };

  ...

  mockedServer(`
    query {
      viewer {
        friends(first: 5) {
          edges {
            node {
              name
            }
            cursor
          }
          pageInfo {
            hasNextPage
            hasPreviousPage
          }
        }
      }
    }`, {} , {
      viewer: {
        friends: {
          // Here only 3 items will be returned even though 5 were requested.
          edges: [
            {}, 
            {node: {'name': 'Oscar'}}, 
            null,
          ],
          pageInfo: {
            hasNextPage: false,
          },
        }
      }
    }
  );
  // ->
  // { data:
  //  { viewer:
  //     { friends:
  //        { edges:
  //           [ { node: { name: 'Craig' }, cursor: 'cursor_0' },
  //             { node: { name: 'Oscar' }, cursor: 'cursor_1' },
  //             null ],
  //          pageInfo: { hasNextPage: false, hasPreviousPage: false } } } } }
  ```
  </p>
</details>

<details>
  <summary>Overriding with another `mockConnection`</summary>
  <p>

  ```js
  // TODO Add test and example
  ```
  </p>
</details>

## API Reference

**`mockServer(schemaDefinition, mocks, [getMocks]?)`**  
<details>
  <summary>Return a mocked server. This is the entry point of this lib.</summary>
  <p>

  ```js
  mockServer(
    /**
     * The schema definition string.
     * It can be automatically generated by any GraphQL server.
     * See https://graphql.org/learn/schema/#type-language
     */
    schemaDefinition: string, 
    )>

    /**
     * An object mapping to all the mock functions of each field.
     * mocks[objectTypeName][fieldName] = mockFunction
     * 
     * All queried fields are currently required to have a base mock function defined.
     * This will be probably relaxed in the future.
     *
     * TODO Document interface
     */
    mocks: {[string]: {[string]: MockFunction}}, 

    /**
     * Optional: A function that returns a mock function for a field.
     *
     * This is a hook to define to define mock functions in a programmatic way.
     * It will be called for each field that has not been associated to a mock 
     * function in `mocks`. If the function does not return anything for 
     * a field, then no mock function is attached to the field.
     * 
     * For example, `getRelayMock` can be passed in to 
     * automatically define all the Relay fields.
     */
    getMocks?: Array<
      (
        /**
        * The GraphQL type of the parent ObjectType containing the field.
        * @example: `User` from `User.name`
        */
        parentType : GraphQLType, 
        
        /**
        * The GraphQL field.
        * @example: `name` from `User.name`
        */
        field : GraphQLField,
      ) => MockFunction | void,
    ) : MockServer
  )>;

  /**
   * Mock function
   * 
   * @returns The return type has to match the type of the field it is mocking.
   *   If it returns an object, the object will be deep merged with the 
   *   return values of the mock functions of the nested fields.
   *   See "Usage" > "Mocking nested fields".
   */
  type MockFunction = (
    /**
     * The field arguments
     */
    params: {[string]: any}
  ) => any;

  /**
   * Mock server
   * 
   * @returns The GraphQL response 
   */
  type MockServer = (
    /**
     * The GraphQL query string.
     */
    query: string, 

    /**
     * The GraphQL variables for the query.
     */
    variables: {[string]: any}, 

    /**
     * An object that overrides the response generated by the 
     * mock functions defined in `mocks`.
     * See "Usage" > "Overriding mocked functions with `mockOverride`".
     */
    mockOverride: {[string]: any},
  ) => Object
  ```
  </p>
</details>

**`mockList(size, itemMock?)`**  
<details>
  <summary>Return a mock function for a list.</summary>
  <p>

  ```js
  mockList(
    /**
     * The size of the mocked list.
     */
    size: number,

    /**
     * Optional: A mock function called for each list item.
     * 
     * @returns The return type has to match the type of the 
     *   field it is mocking.
     */
    itemMock?: (
      /**
       * The field arguments if any
       */
      fieldArguments: {[string]: any},

      /**
       * The index of the item in the mock list.
       */
      index: number,
    ) => any 
  )

  ```
  </p>
</details>

**`mockConnection({maxSize?, nodeMock?}?)`**  
<details>
  <summary>Return a mock function for a Relay connection.</summary>
  <p>

  ```js
  mockConnection(
    /**
     * Optional: An object to configure the connection.
     */
    params?: {

      /**
       * Optional: The max size of the mocked collection. 
       */
      maxSize?: number, 

      /**
       * Optional: A mock function called for each node in 
       * the collection.
       * /
      nodeMock?: ({[string]: any}, index) => any,
    },
  )
  ```
  </p>
</details>

**`getRelayMock()`**  
<details>
  <summary>Automatically mock all Relay fields.</summary>
  <p>

  ```js
  /**
   * Strongly recommended if you use Relay.
   * 
   * Add mock functions to all the Relay-specific fields in your schema.
   * It is equivalent to calling `mockConnection` on all connection fields.
   * 
   * This will skip fields that have been mocked via the `mocks` 
   * parameter of `mockServer`. See "API Reference" > "mockServer".
   */
  getRelayMock()
  ```
  </p>
</details>

## TODO

- [x] Add extensive tests
- [x] Add helpful validation errors
- [x] Add helpers for Relay-compliant schemas
- [x] Add documentation
- [x] Add default mocks for standard scalar fields
- [ ] Add test for list of scalars
- [ ] Add support for Enum
- [ ] Add support for custom scalar fields
- [ ] Add FAQ: comparison with `graphql-tools`, etc
- [ ] Add recipes with common testing patterns
- [ ] `getConnection`: `maxSize` should work across pages
- [ ] Fix Flow types 
- [ ] Add credits

## Immediate TODO
- [ ] Add tests to check getRelayMock works with getDefaultMock

## License

[Apache License 2.0](/LICENSE)