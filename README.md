# Umzug [![Build Status](https://travis-ci.org/sequelize/umzug.svg?branch=master)](https://travis-ci.org/sequelize/umzug)
The *umzug* lib is a framework agnostic migration tool for Node.JS. The tool itself is not specifically related to databases but basically provides a clean API for running and rolling back tasks.

## Persistence
In order to keep track of already executed tasks, *umzug* logs successfully executed migrations. This is done in order to allow rollbacks of tasks. There are multiple storage presets available, from which you can choose. Adding a custom is  super simple as well.

## Storages

### JSONStorage
Using the [JSONStorage](src/storages/JSONStorage.js) will create a JSON file which will contain an array with all the executed migrations. You can specify the path to the file. The default for that is `umzug.json` in the working directory of the process.

#### Options

```js
{
  // The path to the json storage.
  // Defaults to process.cwd() + '/umzug.json';
  path: process.cwd() + '/db/sequelize-meta.json'
}
```

### SequelizeStorage
Using the [SequelizeStorage](src/storages/SequelizeStorage.js) will create a table in your SQL database called `SequelizeMeta` containing an entry for each executed migration. You will have to pass a configured instance of Sequelize or an existing Sequelize model. Optionally you can specify the model name, table name, or column name. All major Sequelize versions are supported.

#### Options

```js
{
  // The configured instance of Sequelize.
  // Optional if `model` is passed.
  sequelize: instance,

  // The to be used Sequelize model.
  // Must have column name matching `columnName` option
  // Optional if `sequelize` is passed.
  model: model,

  // The name of the to be used model.
  // Defaults to 'SequelizeMeta'
  modelName: 'Schema',

  // The name of table to create if `model` option is not supplied
  // Defaults to `modelName`
  tableName: 'Schema',

  // The name of table column holding migration name.
  // Defaults to 'name'.
  columnName: 'migration',

  // The type of the column holding migration name.
  // Defaults to `Sequelize.STRING`
  columnType: new Sequelize.STRING(100)
}
```

### MongoDBStorage
Using the [MongoDBStorage](src/storages/MongoDBStorage.js) will create a collection in your MongoDB database called `migrations` containing an entry for each executed migration. You will have either to pass a MongoDB Driver Collection as `collection` property. Alternatively you can pass a established MongoDB Driver connection and a collection name.

#### Options

```js
{
  // a connection to target database established with MongoDB Driver
  connection: MongoDBDriverConnection,

  // name of migration collection in MongoDB
  collectionName: 'migrations',

  // reference to a MongoDB Driver collection
  collection: MongoDBDriverCollection
}
```
#### Events

Umzug is an EventEmitter. Each of the following events will be called with `name, migration` as arguments. Events are a convenient place
to implement application-specific logic that must run around each migration:

* *migrating* - A migration is about to be executed.
* *migrated* - A migration has successfully been executed.
* *reverting* - A migration is about to be reverted.
* *reverted* - A migration has successfully been reverted.

### Storage
If want to run migrations without storing them anywhere, you can use the [Storage](src/storages/Storage.js).

### Custom
In order to use custom storage, you have two options:

#### Way 1: Pass instance to constructor

You can pass your storage instance to Umzug constructor.
```js
class CustomStorage {
  constructor(...) {...}
  logMigration(...) {...}
  unlogMigration(...) {...}
  executed(...) {...}
}
let umzug = new Umzug({ storage: new CustomStorage(...) })
```

#### Way 2: Require external module from npmjs.com

Create and publish a module which has to fulfill the following API. You can just pass the name of the module to the configuration and *umzug* will require it accordingly. The API that needs to be exposed looks like this:

```js
var Bluebird = require('bluebird');
var redefine = require('redefine');

module.exports = redefine.Class({
  constructor: function ({ option1: 'defaultValue1' } = {}) {
    this.option1 = option1;
  },

  logMigration: function (migrationName) {
    return new Bluebird(function (resolve, reject) {
      // This function logs a migration as executed.
      // It will get called once a migration was
      // executed successfully.
    });
  },

  unlogMigration: function (migrationName) {
    return new Bluebird(function (resolve, reject) {
      // This function removes a previously logged migration.
      // It will get called once a migration has been reverted.
    });
  },

  executed: function () {
    return new Bluebird(function (resolve, reject) {
      // This function lists the names of the logged
      // migrations. It will be used to calculate
      // pending migrations. The result has to be an
      // array with the names of the migration files.
    });
  }
});
```

## Migrations
Migrations are basically files that describe ways of executing and reverting tasks. In order to allow asynchronicity, tasks return a Promise object which provides a `then` method.

### Format
A migration file ideally contains an `up` and a `down` method, which represent a function which achieves the task and a function that reverts a task. The file could look like this:

```js
'use strict';

var Bluebird = require('bluebird');

module.exports = {
  up: function () {
    return new Bluebird(function (resolve, reject) {
      // Describe how to achieve the task.
      // Call resolve/reject at some point.
    });
  },

  down: function () {
    return new Bluebird(function (resolve, reject) {
      // Describe how to revert the task.
      // Call resolve/reject at some point.
    });
  }
};
```

## Examples

- [sequelize-migration-hello](https://github.com/abelnation/sequelize-migration-hello)

## Usage

### Installation
The *umzug* lib is available on npm:

```js
npm install umzug
```

### API
The basic usage of *umzug* is as simple as:

```js
var Umzug = require('umzug');
var umzug = new Umzug({});

umzug.someMethod().then(function (result) {
  // do something with the result
});
```

#### Executing migrations
The `execute` method is a general purpose function that runs for every specified migrations the respective function.

```js
umzug.execute({
  migrations: ['some-id', 'some-other-id'],
  method: 'up'
}).then(function (migrations) {
  // "migrations" will be an Array of all executed/reverted migrations.
});
```

#### Getting all pending migrations
You can get a list of pending/not yet executed migrations like this:

```js
umzug.pending().then(function (migrations) {
  // "migrations" will be an Array with the names of
  // pending migrations.
});
```

#### Getting all executed migrations
You can get a list of already executed migrations like this:

```js
umzug.executed().then(function (migrations) {
  // "migrations" will be an Array of already executed migrations.
});
```

#### Executing pending migrations
The `up` method can be used to execute all pending migrations.

```js
umzug.up().then(function (migrations) {
  // "migrations" will be an Array with the names of the
  // executed migrations.
});
```

It is also possible to pass the name of a migration in order to just run the migrations from the current state to the passed migration name (inclusive).

```js
umzug.up({ to: '20141101203500-task' }).then(function (migrations) {});
```

You also have the ability to choose to run migrations *from* a specific migration, excluding it:

```js
umzug.up({ from: '20141101203500-task' }).then(function (migrations) {});
```

In the above example umzug will execute all the pending migrations found **after** the specified migration. This is particularly useful if you are using migrations on your native desktop application and you don't need to run past migrations on new installs while they need to run on updated installations.

You can combine `from` and `to` options to select a specific subset:

```js
umzug.up({ from: '20141101203500-task', to: '20151201103412-items' }).then(function (migrations) {});
```

Running specific migrations while ignoring the right order, can be done like this:

```js
umzug.up({ migrations: ['20141101203500-task', '20141101203501-task-2'] });
```

There are also shorthand version of that:

```js
umzug.up('20141101203500-task'); // Runs just the passed migration
umzug.up(['20141101203500-task', '20141101203501-task-2']);
```

Running

#### Reverting executed migration
The `down` method can be used to revert the last executed migration.

```js
umzug.down().then(function (migration) {
  // "migration" will the name of the reverted migration.
});
```

It is possible to pass the name of a migration until which (inclusive) the migrations should be reverted. This allows the reverting of multiple migrations at once.

```js
umzug.down({ to: '20141031080000-task' }).then(function (migrations) {
  // "migrations" will be an Array with the names of all reverted migrations.
});
```

To revert all migrations, you can pass 0 as the `to` parameter:

```js
umzug.down({ to: 0 });
```

Reverting specific migrations while ignoring the right order, can be done like this:

```js
umzug.down({ migrations: ['20141101203500-task', '20141101203501-task-2'] });
```

There are also shorthand version of that:

```js
umzug.down('20141101203500-task'); // Runs just the passed migration
umzug.down(['20141101203500-task', '20141101203501-task-2']);
```

### Configuration

It is possible to configure *umzug* instance by passing an object to the constructor. The possible options are:

```js
{
  // The storage.
  // Possible values: 'none', 'json', 'mongodb', 'sequelize', an argument for `require()`, including absolute paths
  storage: 'json',

  // The options for the storage.
  // Check the available storages for further details.
  storageOptions: {},

  // The logging function.
  // A function that gets executed everytime migrations start and have ended.
  logging: false,

  // The name of the positive method in migrations.
  upName: 'up',

  // The name of the negative method in migrations.
  downName: 'down',

  // (advanced) you can pass an array of Migration instances instead of the options below
  migrations: {
    // The params that gets passed to the migrations.
    // Might be an array or a synchronous function which returns an array.
    params: [],

    // The path to the migrations directory.
    path: 'migrations',

    // The pattern that determines whether or not a file is a migration.
    pattern: /^\d+[\w-]+\.js$/,

    // Search for migrations matching the pattern in the subdirectories
    traverseDirectories: false,

    // A function that receives and returns the to be executed function.
    // This can be used to modify the function.
    wrap: function (fun) { return fun; },

    // A function that maps a file path to a migration object in the form
    // { up: Function, down: Function }. The default for this is to require(...)
    // the file as javascript, but you can use this to transpile TypeScript,
    // read raw sql etc.
    // See https://github.com/sequelize/umzug/tree/master/test/fixtures
    // for examples.
    customResolver: function (sqlPath)  {
        return { up: () => sequelize.query(require('fs').readFileSync(sqlPath, 'utf8')) }
    }
  }
}
```

## License
The MIT License (MIT)

Copyright (c) 2014-2017 Sequelize contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
