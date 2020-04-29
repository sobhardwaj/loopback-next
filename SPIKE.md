# Run LB3 tests from LB4 when LB3 is mounted on the LB4 app

Since LoopBack 4 offers a way to mount LoopBack 3 applications on a LoopBack 4
project with the use of
[`@loopback/booter-lb3app`](https://github.com/strongloop/loopback-next/tree/master/packages/booter-lb3app),
there should also be a way for users to run their tests as part of LoopBack 4's
`npm test` command.

We want the LoopBack 3 tests to use the LoopBack 4 server rather than the
LoopBack 3 application. This spike aims to test running both acceptance and
integration LoopBack 3 tests.

## All Tests

In order to run LoopBack 3's tests from their current folder, add LB3 tests'
path to `test` entry in package.json:

- `"test": "lb-mocha \"dist/**tests**/\*_/_.js\" \"lb3app/test/\*.js\""`

In this case, the test folder is
[`/lb3app/test`](https://github.com/strongloop/loopback-next/tree/spike/lb3test/examples/lb3-application/lb3app/test)
from the root of the LoopBack 4 project.

This will run LoopBack 4 tests first then LoopBack 3 tests.

## Acceptance Tests

First, move any LoopBack 3 test dependencies to package.json's devDependencies
and run:

```sh
npm install
```

In your test file:

- Update to use the LB4 Express server when doing requests:

  ```ts
  // can use lb4's testlab's supertest as the dependency is already installed
  const request = require('@loopback/testlab').supertest;
  const assert = require('assert');
  const should = require('should');
  const ExpressServer = require('../../dist/server').ExpressServer;

  let app;

  function json(verb, url) {
    // use the LB4 express server
    return request(app.server)
      [verb](url)
      .set('Content-Type', 'application/json')
      .set('Accept', 'application/json')
      .expect('Content-Type', /json/);
  }
  ```

- Boot and start the LB4 app in your before hook, and stop the app in the after
  hook:

  ```ts
  describe('LoopBack 3 style tests', function () {
    before(async function () {
      app = new ExpressServer();
      await app.boot();
      await app.start();
    });

    after(async () => {
      await app.stop();
    });

    // your tests here
  });
  ```

- Example of this use can be seen in
  [`examples/lb3-application/lb3app/test/acceptance.js`](https://github.com/strongloop/loopback-next/blob/spike/lb3test/examples/lb3-application/lb3app/test/acceptance.js)
  which has the same tests as
  [`src/__tests__/acceptance/lb3app.acceptance.ts`](https://github.com/strongloop/loopback-next/blob/spike/lb3test/examples/lb3-application/src/__tests__/acceptance/lb3app.acceptance.ts),
  but in LB3 style.

Now when you run `npm test` your LoopBack 3 tests should be run along with any
LoopBack 4 tests you have.

Optional: Another option is to migrate your tests to use LoopBack 4 style of
testing, similar to `src/__tests__/acceptance/lb3app.acceptance.ts`.
Documentation for LoopBack testing can be found in
https://loopback.io/doc/en/lb4/Testing-your-application.html.

## Integration Tests

For the integration tests, LoopBack 3 models were bound to the LoopBack 4
application in order to allow JavaScript API to call application logic such as
`Model.create()`. This can be seen in
[`packages/booter-lb3app/src/lb3app.booter.ts`](https://github.com/strongloop/loopback-next/blob/spike/lb3test/packages/booter-lb3app/src/lb3app.booter.ts#L76-L85).

In order to retrieve the model from the application's context,
`getValueOrPromise()` can be used as follows:

```ts
describe('LoopBack 3 style integration tests', function () {
  let app;
  let CoffeeShop;

  before(async function () {
    app = new ExpressServer();
    await app.boot();
    await app.start();
  });

  before(() => {
    // follow the syntax: models.lb3-{ModelName}
    CoffeeShop = app.lbApp.getValueOrPromise('models.lb3-CoffeeShop');
  });

  after(async () => {
    await app.stop();
  });

  // your tests here
});
```

Example integration tests can be found in
[`examples/lb3-application/lb3app/test/integration.js`](https://github.com/strongloop/loopback-next/blob/spike/lb3test/examples/lb3-application/lb3app/test/integration.js).
