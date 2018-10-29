<!--@h1([pkg.name])-->
# marpat
<!--/@-->

[![Build Status](https://travis-ci.org/Luidog/fms-api-client.png?branch=master)](https://travis-ci.org/Luidog/marpat) [![Known Vulnerabilities](https://snyk.io/test/github/Luidog/marpat/badge.svg?targetFile=package.json)](https://snyk.io/test/github/Luidog/marpat?targetFile=package.json) [![Coverage Status](https://coveralls.io/repos/github/Luidog/marpat/badge.svg?branch=document-update)](https://coveralls.io/github/Luidog/marpat?branch=document-update) [![GitHub](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/Luidog/marpat/blob/master/LICENSE.md)

Marpat is lightweight object modeling tool that uses ES6 classes to model data. This is a fork of [Camo](https://github.com/scottwrobinson/camo). When integrating with FileMaker servers I found a need for a local datastore alongside filemaker that could be encrypted and could scale effectively.

## Jump To

- <a href="#advantages">Advantages</a>
- <a href="#install-and-run">Install and Run</a>
- <a href="#quick-start">Quick Start</a>
  - <a href="#connect-to-the-database">Connect to the Database</a>
  - <a href="#declaring-your-document">Declaring Your Document</a>
    - <a href="#embedded-documents">Embedded Documents</a>
  - <a href="#creating-and-saving">Creating and Saving</a>
  - <a href="#loading">Loading</a>
  - <a href="#deleting">Deleting</a>
  - <a href="#counting">Counting</a>
  - <a href="#hooks">Hooks</a>
  - <a href="#misc">Misc.</a>
- <a href="#transpiler-support">Transpiler Support</a>
- <a href="#contributing">Contributing</a>
- <a href="#copyright-license">Copyright & License</a>

## Advantages

So, why use Marpat?

- **ES6**: ES6 features are quickly being added to Node, especially now that it has merged with io.js. With all of these new features being released, Marpat is getting a head start in writing tested and proven ES6 code. This also means that native Promises are built-in to Marpat, so no more `promisify`-ing your ODM or waiting for Promise support to be added natively.
- **Easy to use**: While JavaScript is a great language overall, it isn't always the easiest for beginners to pick up. Marpat aims to ease that transition by providing familiar-looking classes and a simple interface. Also, there is no need to install a full MongoDB instance to get started thanks to the support of NeDB.
- **Multiple backends**: Marpat was designed and built with multiple Mongo-like backends in mind, like NeDB, MongoDB, and Firebase. With NeDB support, for example, you don't need to install a full MongoDB instance for development or for smaller projects. This also allows you to use Marpat in the browser, since databases like NeDB supports in-memory storage.
- **Lightweight**: Marpat is just a very thin wrapper around the backend databases, which mean you won't be sacrificing performance.

## Install and Run

To use Marpat, you must first have installed **Node >=8.0.x**, then run the following commands:

    npm install marpat --save

And at least ONE of the following:

    npm install nedb --save

    OR

    npm install mongodb --save

## Quick Start

Marpat was built with ease-of-use and ES6 in mind, so you might notice it has more of an OOP feel to it than many existing libraries and ODMs. Don't worry, focusing on object-oriented design doesn't mean we forgot about functional techniques or asynchronous programming. Promises are built-in to the API. Just about every call you make interacting with the database (find, save, delete, etc) will return a Promise.

### Connect to the Database

Before using any document methods, you must first connect to your underlying database. All supported databases have their own unique URI string used for connecting. The URI string usually describes the network location or file location of the database. However, some databases support more than just network or file locations. NeDB, for example, supports storing data in-memory, which can be specified to Marpat via `nedb://memory`. See below for details:

- MongoDB:
  - Format: mongodb://[username:password@]host[:port][/db-name]
  - Example: `const uri = 'mongodb://Ben:abc123@localhost:27017/animals';`
- NeDB:
  - Format: nedb://[directory-path] OR nedb://memory
  - Example: `const uri = 'nedb://data';`

So to connect to an NeDB database, use the following:

```javascript
const { connect } = require('marpat')

const database;
const uri = 'nedb://data';
connect(uri).then(function(db) {
    database = db;
});
```

### Declaring Your Document

All models must inherit from the `Document` class, which handles much of the interface to your backend NoSQL database.

```javascript
const { Document } = require('marpat');

class Company extends Document {
    constructor() {
        super();

        this.name = String;
        this.valuation = {
            type: Number,
            default: 10000000000,
            min: 0
        };
        this.employees = [String];
        this.dateFounded = {
            type: Date,
            default: Date.now
        };
    }

    static collectionName() {
        return 'companies';
    }
}
```

Notice how the schema is declared right in the constructor as member variables. All _public_ member variables (variables that don't start with an underscore [\_]) are added to the schema.
The name of the collection can be set by overriding the `static collectionName()` method, which should return the desired collection name as a string. If one isn't given, then Marpat uses the name of the class and naively appends an 's' to the end to make it plural. Schemas can also be defined using the `this.schema()` method. For example, in the `constructor()` method you could use:

```javascript
this.schema({
    name: String,
    valuation: {
        type: Number,
        default: 10000000000,
        min: 0
    },
    employees: [String],
    dateFounded: {
        type: Date,
        default: Date.now
    }
});
```

Currently supported variable types are:

- `String`
- `Number`
- `Boolean`
- `Buffer`
- `Date`
- `Object`
- `Array`
- `EmbeddedDocument`
- Document Reference

Arrays can either be declared as either un-typed (using `Array` or `[]`), or typed (using the `[TYPE]` syntax, like `[String]`). Typed arrays are enforced by Marpat on `.save()` and an `Error` will be thrown if a value of the wrong type is saved in the array. Arrays of references are also supported.

To declare a member variable in the schema, either directly assign it one of the types listed above, or assign it an object with options, like this:

```javascript
this.primeNumber = {
    type: Number,
    default: 2,
    min: 0,
    max: 25,
    choices: [2, 3, 5, 7, 11, 13, 17, 19, 23],
    unique: true
};
```

The `default` option supports both values and no-argument functions (like `Date.now`). Currently the supported options/validators are:

- `type`: The value's type _(required)_
- `default`: The value to be assigned if none is provided _(optional)_
- `min`: The minimum value a Number can be _(optional)_
- `max`: The maximum value a Number can be _(optional)_
- `choices`: A list of possible values _(optional)_
- `match`: A regex string that should match the value _(optional)_
- `validate`: A 1-argument function that returns `false` if the value is invalid _(optional)_
- `unique`: A boolean value indicating if a 'unique' index should be set _(optional)_
- `required`: A boolean value indicating if a key value is required _(optional)_

To reference another document, just use its class name as the type.

```javascript
class Dog extends Document {
    constructor() {
        super();

        this.name = String;
        this.breed = String;
    }
}

class Person extends Document {
    constructor() {
        super();

        this.pet = Dog;
        this.name = String;
        this.age = String;
    }

    static collectionName() {
        return 'people';
    }
}
```

#### Embedded Documents

Embedded documents can also be used within `Document`s. You must declare them separately from the main `Document` that it is being used in. `EmbeddedDocument`s are good for when you need an `Object`, but also need enforced schemas, validation, defaults, hooks, and member functions. All of the options (type, default, min, etc) mentioned above work on `EmbeddedDocument`s as well.

```javascript
var { Document } = require('Marpat');
var { EmbeddedDocument } = require('Marpat');

class Money extends EmbeddedDocument {
    constructor() {
        super();

        this.value = {
            type: Number,
            choices: [1, 5, 10, 20, 50, 100]
        };

        this.currency = {
            type: String,
            default: 'usd'
        };
    }
}

class Wallet extends Document {
    constructor() {
        super();
        this.contents = [Money];
    }
}

const wallet = Wallet.create();
wallet.contents.push(Money.create());
wallet.contents[0].value = 5;
wallet.contents.push(Money.create());
wallet.contents[1].value = 100;

wallet.save().then(function() {
    console.log('Both Wallet and Money objects were saved!');
});
```

### Creating and Saving

To create a new instance of our document, we need to use the `.create()` method, which handles all of the construction for us.

```javascript
const lassie = Dog.create({
    name: 'Lassie',
    breed: 'Collie'
});

lassie.save().then(function(l) {
    console.log(l._id);
});
```

Once a document is saved, it will automatically be assigned a unique identifier by the backend database. This ID can be accessed by the `._id` property.

If you specified a default value (or function) for a schema variable, that value will be assigned on creation of the object.

An alternative to `.save()` is `.findOneAndUpdate(query, update, options)`. This static method will find and update (or insert) a document in one atomic operation (atomicity is guaranteed in MongoDB only). Using the `{upsert: true}` option will return a new document if one is not found with the given query.

### Loading

Both the find and delete methods following closely (but not always exactly) to the MongoDB API, so it should feel fairly familiar.

If querying an object by `id`, you _must_ use `_id` and **not** `id`.

To retrieve an object, you have a few methods available to you.

- `.findOne(query, options)` (static method)
- `.find(query, options)` (static method)

The `.findOne()` method will return the first document found, even if multiple documents match the query. `.find()` will return all documents matching the query. Each should be called as static methods on the document type you want to load.

```javascript
Dog.findOne({ name: 'Lassie' }).then(function(l) {
    console.log('Got Lassie!');
    console.log('Her unique ID is', l._id);
});
```

`.findOne()` currently accepts the following option:

- `populate`: Boolean value to load all or no references. Pass an array of field names to only populate the specified references
  - `Person.findOne({name: 'Billy'}, {populate: true})` populates all references in `Person` object
  - `Person.findOne({name: 'Billy'}, {populate: ['address', 'spouse']})` populates only 'address' and 'spouse' in `Person` object
  - `select`: removes all properties from the returning data except those that match an array of values, or \_id and \_schema
  - `Person.findOne({}, {select: ['name']})` returns data with properties \_id, \_schema, and name. 

`.find()` currently accepts the following options:

- `populate`: Boolean value to load all or no references. Pass an array of field names to only populate the specified references
  - `Person.find({lastName: 'Smith'}, {populate: true})` populates all references in `Person` object
  - `Person.find({lastName: 'Smith'}, {populate: ['address', 'spouse']})` populates only 'address' and 'spouse' in `Person` object
- `sort`: Sort the documents by the given field(s)
  - `Person.find({}, {sort: '-age'})` sorts by age in descending order
  - `Person.find({}, {sort: ['age', 'name']})` sorts by ascending age and then name, alphabetically
- `limit`: Limits the number of documents returned
  - `Person.find({}, {limit: 5})` returns a maximum of 5 `Person` objects
- `skip`: Skips the given number of documents and returns the rest
  - `Person.find({}, {skip: 5})` skips the first 5 `Person` objects and returns all others
- `select`: removes all properties from the returning data except those that match an array of values, or \_id and \_schema
  - `Person.find({}, {select: ['name']})` returns data with properties \_id, \_schema, and name. 

`.findOneAndUpdate()` currently accepts the following options:

- `populate`: Boolean value to load all or no references.
  - `Person.findOneAndUpdate({name: 'Ben'},{name: 'Ben Danger'},{lastName: 'Smith'}, {populate: true})` populates all references in `Person` object
- `select`: removes all properties from the returning data except those that match an array of values, or \_id and \_schema
  - `Person.findOneAndUpdate({name: 'Ben'},{name: 'Ben Danger'}, {select: ['name']})` returns data with properties \_id, \_schema, and name. 

### Deleting

To remove documents from the database, use one of the following:

- `.delete()`
- `.deleteOne(query, options)` (static method)
- `.deleteMany(query, options)` (static method)
- `.findOneAndDelete(query, options)` (static method)

The `.delete()` method should only be used on an instantiated document with a valid `id`. The other three methods should be used on the class of the document(s) you want to delete.

```javascript
Dog.deleteMany({ breed: 'Collie' }).then(function(numDeleted) {
    console.log('Deleted', numDeleted, 'Collies from the database.');
});
```

### Counting

To get the number of matching documents for a query without actually retrieving all of the data, use the `.count()` method.

```javascript
Dog.count({ breed: 'Collie' }).then(function(count) {
    console.log('Found', count, 'Collies.');
});
```

### Hooks

Marpat provides hooks for you to execute code before and after critical parts of your database interactions. For each hook you use, you may return a value (which, as of now, will be discarded) or a Promise for executing asynchronous code. Using Promises throughout Marpat allows us to not have to provide separate async and sync hooks, thus making your code simpler and easier to understand.

Hooks can be used not only on `Document` objects, but `EmbeddedDocument` objects as well. The embedded object's hooks will be called when it's parent `Document` is saved/validated/deleted (depending on the hook you provide).

In order to create a hook, you must override a class method. The hooks currently provided, and their corresponding methods, are:

- pre-init: `preInit(data)`
- pre-validate: `preValidate()`
- post-validate: `postValidate()`
- pre-save: `preSave()`
- post-save: `postSave()`
- pre-delete: `preDelete()`
- post-delete: `postDelete()`
- post-find: `postFind()`

Here is an example of using a hook (pre-delete, in this case):

```javascript
class Company extends Document {
    constructor() {
        super();

        this.employees = [Person];
    }

    static collectionName() {
        return 'companies';
    }

    preDelete() {
        var deletes = [];
        this.employees.forEach(function(e) {
            var p = new Promise(function(resolve, reject) {
                resolve(e.delete());
            });

            deletes.push(p);
        });

        return Promise.all(deletes);
    }
}
```

The code above shows a pre-delete hook that deletes all the employees of the company before it itself is deleted. As you can see, this is much more convenient than needing to always remember to delete referenced employees in the application code.

**Note**: The `.preDelete()` and `.postDelete()` hooks are _only_ called when calling `.delete()` on a Document instance. Calling `.deleteOne()` or `.deleteMany()` will **not** trigger the hook methods.

### Misc.

- `Marpat.Client()`: Retrieves the marpat database client
- `Marpat.Client().driver()`: Retrieves the underlying database driver (`MongoClient` or a map of NeDB collections)
- `Document.toJSON()`: Serializes the given document to just the data, which includes nested and referenced data

## Transpiler Support

This is a work in progress. You can use the camo transpilers for now. While many transpilers won't have any problem with Marpat, some need extra resources/plugins to work correctly:

- Babel
  - [babel-preset-camo](https://github.com/luidog/babel-preset-camo): Babel preset for all es2015 plugins supported by Camo
- TypeScript
  - [DefinitelyTyped/camo](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/master/camo): camo declaration file (h/t [lucasmciruzzi](https://github.com/lucasmciruzzi))
  - [IndefinitivelyTyped/camo](https://github.com/IndefinitivelyTyped/camo): Typings support for camo (h/t [WorldMaker](https://github.com/WorldMaker))

## Contributing

Feel free to open new issues or submit pull requests for Marpat. If you'd like to contact me before doing so, feel free to get in touch (see Contact section below).

Before opening an issue or submitting a PR, I ask that you follow these guidelines:

**Issues**

- Please state whether your issue is a question, feature request, or bug report.
- Always try the latest version of Marpat before opening an issue.
- If the issue is a bug, be sure to clearly state your problem, what you expected to happen, and what all you have tried to resolve it.
- Always try to post simplified code that shows the problem. Use Gists for longer examples.

**Pull Requests**

- If your PR is a new feature, please consult with me first.
- Any PR should contain only one feature or bug fix. If you have more than one, please submit them as separate PRs.
- Always try to include relevant tests with your PRs. If you aren't sure where a test should go or how to create one, feel free to ask.
- Include updates to the README when needed.

## Tests

```sh
npm install
npm test
```

<!--@execute('npm run test',[])-->
```default
> marpat@1.18.2 test /Users/luidelaparra/Documents/Development/marpat
> nyc _mocha --recursive ./test --timeout=30000 --exit



  Base Client
    Required Methods
      ✓ should require a save method
      ✓ should require a delete method
      ✓ should require a delete method
      ✓ should require a deleteMany method
      ✓ should require a findOne method
      ✓ should require a findOneAndUpdate method
      ✓ should require a findOneAndDelete method
      ✓ should require a find method
      ✓ should require a find method
      ✓ should require a count method
      ✓ should require a createIndex method
      ✓ should require a (static) connect method
      ✓ should require a close method
      ✓ should require a clearCollection method
      ✓ should require a dropDatabase method
      ✓ should require a toCanonicalId method
      ✓ should require a isNativeId method
      ✓ should require a toNativeId method
      ✓ should require a nativeIdType method
      ✓ should require a driver method

  Base MongoDB Client
    #save()
      ✓ should reject if it can not update the object (138ms)
    #deleteMany()
      ✓ should reject if it can not delete the object (58ms)
    #delete()
      ✓ should reject if it can not delete the object (53ms)
    #deleteOne()
      ✓ should reject if it can not delete the object (51ms)
    #findOne()
      ✓ should reject if it can not delete the object (56ms)
    #count()
      ✓ should reject an invalid count query (56ms)
    #findOneAndDelete()
      ✓ should reject an invalid findOneAndDelete query (73ms)
      ✓ should return zero if no results are found (70ms)
    #findOneAndupdate()
      ✓ should reject an invalid findOneAndUpdate query (55ms)
      ✓ should return zero if no results are found (72ms)
    #clearCollection()
      ✓ should reject an invalid findOneAndDelete query (56ms)

  MongoDB Client
    #save()
      ✓ should persist the object and its members to the database (53ms)
    #findOne()
      ✓ should load a single object from the collection (59ms)
      ✓ should populate all fields (181ms)
      ✓ should not populate any fields (177ms)
      ✓ should populate specified fields (296ms)
      ✓ should select only the specified fields (177ms)
    #findOneAndUpdate()
      ✓ should return null if there is no document to update (61ms)
      ✓ should load and update a single object from the collection (52ms)
      ✓ should populate all fields (175ms)
      ✓ should not populate any fields (175ms)
      ✓ should return only the selected information (54ms)
      ✓ should insert a single object to the collection (187ms)
      ✓ requires at least two arguments (52ms)
    #findOneAndDelete()
      ✓ requires at least one argument (57ms)
      ✓ should load and delete a single object from the collection (68ms)
    #find()
      ✓ should load multiple objects from the collection
      ✓ should load all objects when query is not provided
      ✓ should sort results in ascending order
      ✓ should sort results in descending order
      ✓ should sort results using multiple keys
      ✓ should sort results using multiple keys
      ✓ should limit number of results returned
      ✓ should skip given number of results
      ✓ should populate all fields (303ms)
      ✓ should not populate any fields (179ms)
      ✓ should populate specified fields (168ms)
      ✓ should select specified fields (176ms)
    #count()
      ✓ should return 0 objects from the collection (54ms)
      ✓ should disregard unsupported options (158ms)
      ✓ should return 2 matching objects from the collection (57ms)
    #delete()
      ✓ should remove instance from the collection (55ms)
      ✓ should return zero if there are no items to remove
    #deleteOne()
      ✓ should remove the object from the collection (56ms)
      ✓ should return zero if there are no items to remove
    #deleteMany()
      ✓ should remove multiple objects from the collection (52ms)
      ✓ should remove all objects when query is not provided (53ms)
    #clearCollection()
      ✓ should remove all objects from the collection (59ms)
    id
      ✓ should allow custom _id values (55ms)
    query
      ✓ should automatically cast string ID in query to ObjectID (53ms)
      ✓ should automatically cast string ID in query to ObjectID (53ms)
      ✓ should automatically cast string IDs in '$in' operator to ObjectIDs (59ms)
      ✓ should automatically cast string IDs in '$nin' operator to ObjectIDs (56ms)
      ✓ should automatically cast string IDs in '$not' operator to ObjectIDs (58ms)
      ✓ should automatically cast string IDs in deep query objects (58ms)
    indexes
      ✓ should reject documents with duplicate values in unique-indexed fields (250ms)
      ✓ should accept documents with duplicate values in non-unique-indexed fields (55ms)

  Base NeDB Client
    #save()
      ✓ should reject if it can not update the object
    #delete()
      ✓ should reject if it can not delete the object
      ✓ should return 0 if the id is null
      ✓ should return 0 if the id is null
    #dropDatabase()
      ✓ should only remove Files if the are already there
    #count()
      ✓ should reject an invalid count query (104ms)
    #findOneAndDelete()
      ✓ should reject an invalid findOneAndDelete query
      ✓ should return zero if no results are found
    #findOneAndupdate()
      ✓ should reject an invalid findOneAndUpdate query
      ✓ should return zero if no results are found
    #clearCollection()
      ✓ should reject an invalid findOneAndDelete query

  NeDB In Memory Capabilities
    #save()
      ✓ should persist the object and its members to the database
    #findOne()
      ✓ should load a single object from the collection
      ✓ should populate all fields
      ✓ should not populate any fields
      ✓ should populate specified fields
      ✓ should select only the specified fields
    #findOneAndUpdate()
      ✓ should return null if there is no document to update
      ✓ should load and update a single object from the collection
      ✓ should populate all fields
      ✓ should not populate any fields
      ✓ should return only the selected information
      ✓ should insert a single object to the collection
      ✓ requires at least two arguments
    #findOneAndDelete()
      ✓ requires at least one argument
      ✓ should load and delete a single object from the collection
    #find()
      ✓ should load multiple objects from the collection
      ✓ should load all objects when query is not provided
      ✓ should sort results in ascending order
      ✓ should sort results in descending order
      ✓ should sort results using multiple keys
      ✓ should sort results using multiple keys
      ✓ should limit number of results returned
      ✓ should skip given number of results
      ✓ should populate all fields
      ✓ should not populate any fields
      ✓ should populate specified fields
      ✓ should select specified fields
    #count()
      ✓ should return 0 objects from the collection
      ✓ should disregard unsupported options
      ✓ should return 2 matching objects from the collection
    #delete()
      ✓ should remove instance from the collection
    #deleteOne()
      ✓ should remove the object from the collection
    #deleteMany()
      ✓ should remove multiple objects from the collection
      ✓ should remove all objects when query is not provided
    #clearCollection()
      ✓ should remove all objects from the collection

  NeDB File System Capabilities
    ✓ should create a file based store (46ms)
    ✓ should return collections as a driver
    ✓ should remove files when used on the File System
    ✓ should not delete files if there are no collections

  NeDbClient - old
    id
      ✓ should allow custom _id values
    indexes
      ✓ should reject documents with duplicate values in unique-indexed fields
      ✓ should accept documents with duplicate values in non-unique-indexed fields

  Connect Capability
    Ensure A Connection
      ✓ should throw an error if connect is not called
    Connect to stores
      ✓ should connect to an nedb connection
      ✓ should connect to a mongodb connection
      ✓ should reject if it can not connect to a mongodb connection
      ✓ should reject an unrecognized connection

  Base Document
    ✓ should throw an error if the static documentClass is not defined
    ✓ should throw an error if the documentClass is not defined

  Cyclical Dependencies
    schema
      ✓ should allow cyclic dependencies

  Document Capabilities
    instantiation
      ✓ should allow creation of instance
      ✓ should allow schema declaration via method
      ✓ should allow creation of instance with data
      ✓ should allow creation of instance with references
    class
      ✓ should allow use of member variables in getters
      ✓ should allow use of member variables in setters
      ✓ should allow use of member variables in methods
      ✓ should allow schemas to be extended
      ✓ should allow schemas to be overridden
      ✓ should provide default collection name based on class name
      ✓ should provide default collection name based on subclass name
      ✓ should allow custom collection name
    types
      ✓ should allow reference types
      ✓ should allow array of references
      ✓ should allow references to be saved using the object or its id
      ✓ should allow array of references to be saved using the object or its id
      ✓ should allow circular references
      ✓ should allow string types
      ✓ should allow number types
      ✓ should allow boolean types
      ✓ should allow date types
      ✓ should allow object types
      ✓ should allow buffer types
      ✓ should allow array types
      ✓ should allow typed-array types
      ✓ should reject objects containing values with different types
      ✓ should reject typed-arrays containing different types
      ✓ should reject unsupported types
    defaults
      ✓ should assign default value if unassigned
      ✓ should assign default value via function if unassigned
      ✓ should be undefined if unassigned and no default is given
    choices
      ✓ should accept value specified in choices
      ✓ should reject values not specified in choices
    min
      ✓ should accept value > min
      ✓ should accept value == min
      ✓ should reject value < min
    max
      ✓ should accept value < max
      ✓ should accept value == max
      ✓ should reject value > max
    match
      ✓ should accept value matching regex
      ✓ should reject value not matching regex
    validate
      ✓ should accept value that passes custom validator
      ✓ should reject value that fails custom validator
    canonicalize
      ✓ should ensure timestamp dates are converted to Date objects
      ✓ should ensure date strings are converted to Date objects
    required
      ✓ should accept empty value that is not required
      ✓ should accept value that is not undefined
      ✓ should accept an empty value if default is specified
      ✓ should accept boolean value
      ✓ should accept date value
      ✓ should accept any number value
      ✓ should reject value that is undefined
      ✓ should reject value if specified default empty value
      ✓ should reject value that is null
      ✓ should reject value that is an empty array
      ✓ should reject value that is an empty string
      ✓ should reject value that is an empty object
    hooks
      ✓ should call all pre and post functions
      ✓ should call postFind when findOne is used
      ✓ should not call postFind when find is used
    serialize
      ✓ should serialize data to JSON
      ✓ should serialize data to JSON
      ✓ should serialize data to JSON
      ✓ should serialize data to JSON and ignore methods

  Embedded
    general
      ✓ should not have an _id
    types
      ✓ should allow embedded types
      ✓ should allow array of embedded types
      ✓ should save nested array of embeddeds
      ✓ should allow nested initialization of embedded types
      ✓ should allow initialization of array of embedded documents
    defaults
      ✓ should assign defaults to embedded types
      ✓ should assign defaults to array of embedded types
    validate
      ✓ should validate embedded values
      ✓ should accept a validation function
      ✓ should accept a validation function
      ✓ should accept an array of validation functions
      ✓ should validate array of embedded values
    canonicalize
      ✓ should ensure timestamp dates are converted to Date objects
    hooks
      ✓ should call all pre and post functions on embedded models
      ✓ should call all pre and post functions on array of embedded models
    serialize
      ✓ should serialize data to JSON
      ✓ should serialize data to JSON and ignore methods

  Issues
    #4
      ✓ should not load duplicate references in array when only one reference is present
    #5
      ✓ should allow multiple references to the same object in same array
    #8
      ✓ should use virtuals when initializing instance with data
    #20
      ✓ should not alias _id to id in queries and returned documents
    #43
      ✓ should save changes made in postValidate hook
      ✓ should save changes made in preSave hook
    #53
      ✓ should validate Array types properly
      ✓ should validate [] types properly
    #55
      ✓ should return updated data on findOneAndUpdate when updating nested data
    #57
      ✓ should not save due to Promise.reject in hook

  Utilities
    deepTraverse()
      ✓ should iterate over all keys nested in an object

  Validation Utility Tests
    ✓ should throw an error if the type is unsupported
    Type Detection
      ✓ should detect Arrays
      ✓ should detect Embedded Documents
      ✓ should reject undefined
      ✓ should reject null
    Supported Type Detection
      ✓ should support strings
      ✓ should support numbers
      ✓ should support object
      ✓ should support Arrays
      ✓ should support booleans
      ✓ should support Documents


  242 passing (7s)

-----------------------|----------|----------|----------|----------|-------------------|
File                   |  % Stmts | % Branch |  % Funcs |  % Lines | Uncovered Line #s |
-----------------------|----------|----------|----------|----------|-------------------|
All files              |     98.7 |    97.66 |    99.59 |      100 |                   |
 marpat                |      100 |      100 |      100 |      100 |                   |
  index.js             |      100 |      100 |      100 |      100 |                   |
 marpat/lib            |      100 |      100 |      100 |      100 |                   |
  base-document.js     |      100 |      100 |      100 |      100 |                   |
  db.js                |      100 |      100 |      100 |      100 |                   |
  document.js          |      100 |      100 |      100 |      100 |                   |
  embedded-document.js |      100 |      100 |      100 |      100 |                   |
  errors.js            |      100 |      100 |      100 |      100 |                   |
  index.js             |      100 |      100 |      100 |      100 |                   |
  util.js              |      100 |      100 |      100 |      100 |                   |
  validate.js          |      100 |      100 |      100 |      100 |                   |
 marpat/lib/clients    |    97.08 |    92.03 |     99.2 |      100 |                   |
  client.js            |      100 |      100 |      100 |      100 |                   |
  index.js             |      100 |      100 |      100 |      100 |                   |
  mongoclient.js       |    98.71 |    97.06 |      100 |      100 |           223,327 |
  nedbclient.js        |    94.97 |    86.36 |    98.04 |      100 |... 12,227,313,439 |
-----------------------|----------|----------|----------|----------|-------------------|
```
<!--/@-->

<!--@dependencies()-->
## <a name="dependencies">Dependencies</a>

- [lodash](https://github.com/lodash/lodash): Lodash modular utilities.
- [nedb](https://github.com/louischatriot/nedb): File-based embedded data store for node.js
- [mongodb](https://github.com/mongodb/node-mongodb-native): The official MongoDB driver for Node.js

<!--/@-->

<!--@devDependencies()-->
## <a name="dev-dependencies">Dev Dependencies</a>

- [chai](https://github.com/chaijs/chai): BDD/TDD assertion library for node.js and the browser. Test framework agnostic.
- [chai-as-promised](https://github.com/domenic/chai-as-promised): Extends Chai with assertions about promises.
- [coveralls](https://github.com/nickmerwin/node-coveralls): takes json-cov output into stdin and POSTs to coveralls.io
- [eslint](https://github.com/eslint/eslint): An AST-based pattern checker for JavaScript.
- [eslint-config-google](https://github.com/google/eslint-config-google): ESLint shareable config for the Google style
- [eslint-config-prettier](https://github.com/prettier/eslint-config-prettier): Turns off all rules that are unnecessary or might conflict with Prettier.
- [eslint-plugin-prettier](https://github.com/prettier/eslint-plugin-prettier): Runs prettier as an eslint rule
- [jsdocs](https://github.com/xudafeng/jsdocs): jsdocs
- [mocha](https://github.com/mochajs/mocha): simple, flexible, fun test framework
- [mocha-lcov-reporter](https://github.com/StevenLooman/mocha-lcov-reporter): LCOV reporter for Mocha
- [mos](https://github.com/mosjs/mos): A pluggable module that injects content into your markdown files via hidden JavaScript snippets
- [mos-plugin-dependencies](https://github.com/mosjs/mos/tree/master/packages/mos-plugin-dependencies): A mos plugin that creates dependencies sections
- [mos-plugin-execute](https://github.com/team-767/mos-plugin-execute): Mos plugin to inline a process output
- [mos-plugin-license](https://github.com/mosjs/mos-plugin-license): A mos plugin for generating a license section
- [nyc](https://github.com/istanbuljs/nyc): the Istanbul command line interface
- [prettier](https://github.com/prettier/prettier): Prettier is an opinionated code formatter
- [varium](https://npmjs.org/package/varium): A strict parser and validator of environment config variables

<!--/@-->

<!--@license()-->
## License

MIT © [Lui de la Parra](http://mutesymphony.com)
<!--/@-->
