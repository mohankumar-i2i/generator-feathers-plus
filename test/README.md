# Testing

## Generator test suite

### Summary

`generators-feathers-plus` has a basic test suite that essentially tests `generate all`.
`npm run mocha:code` runs the test suite comparing generated code with expected code.
This suite runs quickly and success is generally indicative of the generator working properly.

`npm run mocha:tests` runs, for every generated app in the suite, the standard app test generated with the app.
This latter test requires all dependencies and starts up the app.
However it takes a long time.

`tests/writing.test.js` runs these tests. For test `service.test`,
it copies `test/service.test-copy/**` to the work directory.
This would always contain `feathers-gen-specs.json`.
It may also contain `name.schema.js` for services used in the generation,
so the test can extract the service schema from the module.

It then compares the contents generated in the work directly with `test/service.test-expected/**`
 
### Test descriptions
 
Comments associated with each test describe how the modules in `*-copy/**` would have been generated.
For example
```js
  // t3,z3 (z2 ->) Test middleware creation.
  //* generate app            # z-1, Project z-1, npm, src1, socketio (only)
  //* generate service        # NeDB, nedb1, /nedb-1, nedb://../data, auth N, graphql Y
  //* generate service        # NeDB, nedb2, /nedb-2,                 auth N, graphql Y
  //  generate middleware     # mw1, *
  //  generate middleware     # mw2, mw2
  'middleware.test',
```
describes the middleware test.

The app was generated with the parameters shown.
Services `nedb-1` and `nedb-2` were generated with the parameters shown.
Then middlewares `mw1` was generated for path `*`, and `mw2` for `mw2`.

The repo `feathers-x/generator-old-vs-new` contains the initial apps generated with these parameters
using both David's generator and this one.
A source comparision tool was used identify the diffs between the two.

In the above example, the app generated by David's app is in `feathers-x/generator-old-vs-new/z2`
and the new generator's app (as the generator generated code at that time) is in `feathers-x/generator-old-vs-new/z2`

### Existing authentication tests

```js
  // t5, z5 Test authentication scaffolding.
  //  generate app            # z-1, Project z-1, npm, src1, REST and socketio
  //  generate authentication # Local and Auth0, users1, Nedb, nedb://../data, graphql Y
  'authentication-1.test',
  
  // t6, z6 (z5 ->) Test creation of authenticated service with auth scaffolding.
  //* generate app            # z-1, Project z-1, npm, src1, REST and socketio
  //* generate authentication # Local and Auth0, users1, Nedb, nedb://../data, graphql Y
  //  generate service        # NeDB, nedb1, /nedb-1, nedb://../data, auth Y, graphql Y
  'authentication-2.test',
  
  // t7, z7 (z6 ->) Test creation of non-authenticated service with auth scaffolding.
  //* generate app            # z-1, Project z-1, npm, src1, REST and socketio
  //* generate authentication # Local and Auth0, users1, Nedb, nedb://../data, graphql Y
  //* generate service        # NeDB, nedb1, /nedb-1, nedb://../data, auth Y, graphql Y
  //  generate service        # NeDB, nedb2, /nedb-2, nedb://../data, auth >> N <<, graphql Y
  'authentication-3.test'
```

### What this means

There is little need to test what the generator tests already test for.

This is also **not** a suggestion that tests be added to the generator test suite.
Each test takes a long time automate.

However it's worthwhile to keep track of keys tests which I can later consider adding to the test suite.

### A reasonable methodology for testing

It makes sense to create apps, based on the same parameters, using both David's generator and the new one,
and them comparing the code with a source comparison tool.

## Template choices

The generator may use one of several templates to write a particular file depending on the specs.
For example the `src/services/$name$/$name$.service.js` is generated from one of
`src/services/_service/name.service-mongodb.ejs`,
 `src/services/_service/name.service-rethinkdb.ejs` or `src/services/name/name.service.ejs`.

Here is a list of such situational templates:
```text
generators/writing/templates
    _configs
        config.default.json.js      exports default for $app$/config/default.json
        config.production.json.js   exports default for $app$/config/production.json
        package.json.js             exports default for $app$/package.json
    src
        _adapters
            knex.js                 `src/knex.js`
            mongodb.js              `src/mongodb.js`    
            mongoose.js             `src/mongoose.js`
            rethinkdb.js            `src/rethinkdb.js`
            sequelize.js            `src/sequelize.js`       adapter = Sequelize & database != MSSql 
            sequelize-mssql.js      `src/sequelize-mssql.js` adapter = Sequelize & database = MSSql 
        services
            _model
                knex.ejs
                knex-user.ejs
                mongoose.ejs        `src/models/$name$.model.js` 
                mongoose-user.ejs   `src/models/$name$.model.js` if auth enabled and service is the user-entity
                nedb.ejs
                nedb-user.ejs
                sequelize.ejs
                sequelize-user.ejs
            _service
                name.service-mongodb.ejs    `src/services/$name$/$name$.service.js` for mongodb
                name.service-rethinkdb.ejs  `src/services/$name$/$name$.service.js` for rethinkdb
                name.service.ejs            `src/services/$name$/$name$.service.js` for other than mongodb & rethinkdb

            name
                name.class.js       `src/services/$name$/$name$.class.js` for generic service if Node ver missing async/await support
                name.class-async.js `src/services/$name$/$name$.class.js` for generic service if Node ver has async/await support
```

This design is taken from David's generator in order to maximize compatibility.
David's templates however also include `hooks.js` and `hooks-user.js`.
We have consolidated both of those in our `hooks.ejs`.

## Testing checklist

> The following describes a pretty exhaustive set of tests. It doubtful we can go this far. Common sense should be used to test the most common cases.

Testing the generator comes down to, first, testing each of the situational templates.
A service should be generated for each adapter,
making sure that MongoDB and RethinkDB service use their templates in `src/services/_service`
while the others use `src/services/_service/name.service.ejs`.
This would also check the right `src/services/_model/$adapter$.ejs` template is used.

Apps using authentication should be generated where their user-entities are each of the adapters
to check the right `src/services/_model/$adapter$-user.ejs` template is used.

Secondly, because of regeneration, its possible for services to switch roles.
For example, we can generate an app where service `nedb1` is the user-entity,
and `nedb2` is an unauthenticated service.
We then rerun `generate authentication` and make `nedb2` the user entity.
`nedb1` would no longer be the user-entity and the generated code must reflect that.

Interactions like this can get complex and its likely we can't handle every possible case.

## What to test first

The above is all well and good, but what do we test first?

I suggest basic authentication for NeDB, Mongoose and MongoDB.
Generate app, authentication, and a service.
Do a source compare with a similar app generated with David's generator.
Then run our app to make sure it boots.

Then take a trip to the wild side.
Regenerate one of the apps, changing which service is the user-entity.     