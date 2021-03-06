# Recipe Book

[![oppity](https://circleci.com/gh/oppity/recipe-book.svg?style=svg)](https://app.circleci.com/github/oppity/recipe-book/pipelines)

Recipe Book is a lightweight mock server that takes a provided request/response cycle and returns it when called.  This allows for easy mocking of services to improve the speed, reliability and thoroughness of functional testing.  Mocks can be created on the fly as part of an automated test so that it is clear what scenario is being tested right inside the test.

**Why "Recipe Book"?**

Recipes provide instruction for how to prepare something.  In that way, creating a mock is like making a recipe... you define the ingredients and instructions and the service will replay them to produce an output you want... but with fewer calories.

**Why not store mocks?**

Caching mocks with something like Redis is on the TODO list which should make life easier when using Recipe Book for local development.  

As far as relying on a database, it is my belief that the value in Recipe Book is the flexibility provided by mocks that can be created on the fly, before tests or for specific development work.  This way, we can adjust request/response cycles as needed for edge cases without worrying about the number of variations being stored or whether a scenario has already been created.

<br>

## Running Recipe Book

Starting Recipe Book in Docker is simple:

    npm run start:docker

It can be run locally as well:

    npm run start:local

## How Does This Fit Into CI?

A healthy delivery pipeline should include some form of UI testing to validate that the frontend is doing what is expected.  However, functional tests are notoriously unreliable due to complexities with accessing UI elements and stability of test APIs.  In some cases, it may be difficult to validate specific use cases because the APIs don't have the right data.

Recipe Book allows for full customization of request and response to match whatever scenario needs to be tests.  It can be pulled down as part of a CI pipeline and run as a service in Docker allowing for UI testing to be completed more reliably even as part of a PR.

**Note:** It's probably good practice to run your same automated tests against a live API at some point since mocks can go stale.

## Usage

1.  The application being tested should point all requests to Recipe Book url.
2.  Before a test, create the required mocks by issuing a request to the endpoint.
3.  Run the UI tests.

**Note:** Though not required, it's useful to clear out all mocks after a test has run to avoid any potential collisions and inadvertently broken tests.

### REST Interface

Recipe Book exposes a basic REST API for managing mocks on the service.

| Endpoint | Method | Use                                | Body                   | Parameters |
| -------- | ------ | ---------------------------------- | ---------------------- | ---------- |
| `/mock`  | GET    | List available mocks on the server | None                   | None       |
| `/mock`  | POST   | Create a new mock                  | `{ mock: [{ Mock }] }` | None       |
| `/mock`  | DELETE | Clear all mocks                    | None                   | None       |

### Formatting a mock

Mocks can be submitted as either an object or an array.  Using an array will save time if a particular test requires significant setup.

A new mock should follow a particular format:

| Attribute             | Description                                                     | Format  | Options                       |
| --------------------- | --------------------------------------------------------------- | ------- | ----------------------------- |
| `name`                | A friendly name to reference a mock                             | String  |                               |
| `request.path`        | The url being requested.  Will match against `originalUrl`      | String  | Should start with `/`         |
| `request.method`      | The http method used in the request                             | String  | GET, PUT, POST, PATCH, DELETE |
| `response.body`       | The body that will be returned when this mock is called.        | Object  |                               |
| `response.statusCode` | The status code that will be returned when this mock is called. | Integer | Any valid status code         |
| `response.timeout`    | A time for the Recipe Book to wait before responding.           | Integer | Measured in milliseconds      |

**!!Important Note:** Responses can either be a response object (`{ body, statusCode, timeout }`) or they can be an array of response objects.  Please see [Creating Mocks That Respond In Sequence](#creating-mocks-that-respond-in-sequence) for more detail.

### Matching A Mock

Mocks are matched based on `request.path` and `request.method` so path should match `originalUrl` from Express.js.  This will include query parameters if submitted, e.g. `/hello?q=world`.

### Creating Mocks That Respond In Sequence

Recipe Book supports mocks that enable a particular flow that is dependent on other requests.  It will treat an array of response objects as a sequence and return them in order starting with index 0 until the array is empty.  Once it is empty it will return a `NotFoundError`.

For example, say your application is processing a transaction and makes the following steps:

```javascript
1. GET /balance // result: 100
2. POST /credit // { body: { amount: 25 } }
3. GET /balance // result: 125
```

To model the above behavior, create two mocks:

```javascript
{ 
    name: 'balances',
    request: {
        path: '/balance',
        method: 'GET'
    },
    response: [
        {
            body: { balance: 100 },
            statusCode: 200
        },
        {
            body: { balance: 125 },
            statusCode: 200
        }
    ]
}

{
    name: 'createCredit',
    request: {
        path: '/credit',
        method: 'GET'
    },
    response: {
        body: {},
        statusCode: 201
    }
}
```

## Example

Refer to the `/example` directory for usage.

## TODO:

-   [x]  Improved logging
-   [ ]  Match mock on header values
-   [ ]  Delete or update a specific mock based on name
-   [x]  Allow for delaying a response
-   [ ]  Request passthrough to service
-   [ ]  "Sessions" to avoid collisions if Recipe Book is deployed
-   [ ]  Enable caching for _temporary_ storage of mocks 
