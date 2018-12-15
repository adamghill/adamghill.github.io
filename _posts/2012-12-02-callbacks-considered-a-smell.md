---
layout: post
title: "Callbacks considered a smell"
date: 2012-12-02 10:20:16 -0400
---

Here are the ways I have approached dealing with nested callbacks in my own JavaScript code. My samples all set 3 keys in a fake database, then retrieves those 3 keys and prints the values as a decent approximation of dealing with callbacks and I/O. These samples were part of a Intro to Node.js talk I gave at [The Motley Fool](http://culture.fool.com/). The code is located at https://github.com/adamghill/openspace/tree/master/node.js.

# Callbacks

This what was my JavaScript looked like when I first started writing callback-heavy code. It quickly gets unwieldy and hard to refactor because of the nested anonymous functions, and the implicit dependencies.

```javascript
var db = require('./db/callbackDb');

db.set('key1', 'value1', function(err) {
    if (err) throw err;

db.set('key2', 'value2', function(err) {
    if (err) throw err;

    db.set('key3', 'value3', function(err) {
        if (err) throw err;

        var str = '';
        db.get('key1', function(err, value) {
            if (err) throw err;
            str += value + ' - ';

            db.get('key2', function(err, value) {
                if (err) throw err;
                str += value + ' - ';

                db.get('key3', function(err, value) {
                    if (err) throw err;
                    str += value + ' - ';

                    console.log(str);
                });
            });
        });
    });
});
```

# Events

If you have the ability to change your underlying code, you can add an `EventEmitter` so you can follow the observer pattern. My fake evented database  has been updated with a wrapper that fires when the functions are finished.

```javascript
var db = require('./db/eventedDb');
var str = '';
var counter = 0;
db.eventEmitter.on('onGet', function(value) {
str += value + ' - ';
counter++;

if (counter === 3) {
	console.log(str);
}
});

db.eventEmitter.on('onSet', function(key) {
	db.get(key);
});

db.eventEmitter.on('onError', function(err) {
	throw err;
});

db.set('key1', 'value1');
db.set('key2', 'value2');
db.set('key3', 'value3');
```

# Promises

Promises (or defereds or futures) are a way to represent the outcome of 
an asynchronous call. They used to be included in the core of Node.js, 
however, they were pulled out around 0.2. However, There are a many 
user-land modules that attempt to follow the promises interface. For 
more details on promises, [this article](http://howtonode.org/promises) explains them in much more detail. For promises to work, you have to return a promise object from the underlying code, so my [fake db](https://github.com/adamghill/openspace/blob/master/node.js/src/04-callbacks/db/promisesDb.js) got some more love.

```javascript
var db = require('./db/promisesDb');
var str = '';

db.set('key1', 'value1').then(function(key) {
	return db.set('key2', 'value2');
}).then(function() {
	return db.set('key3', 'value3');
}).then(function() {
	return db.get('key1');
}).then(function(value) {
	str += value + ' - ';

	return db.get('key2');
}).then(function(value) {
	str += value + ' - ';

	return db.get('key3');
}).then(function(value) {
	str += value + ' - ';

	console.log(str);
}); 
```

# Control flow

In addition to the basic promises concept, there are many control flow modules that help sequence asynchronous calls. I used caolan’s [async](https://github.com/caolan/async) because that is the one I am most comfortable with, but there are a ton floating around. Most of them allow you to parallelize a set of async calls and wait for their end result, or force async calls into a particular series.

```javascript
var db = require('./db/callbackDb');
var async = require('async');

async.series([
  function(callback) {
    db.set('key1', 'value1', function(err) {
    	callback(err);
    });
  },
  function(callback) {
    db.set('key2', 'value2', function(err) {
    	callback(err);
    });
  },
	function(callback) {
    db.set('key3', 'value3', function(err) {
    	callback(err);
    });
  },
  function(callback) {
    db.get('key1', function(err, value) {
    	callback(err, value);
    });
  },
  function(callback) {
    db.get('key2', function(err, value) {
    	callback(err, value);
    });
  },
  function(callback) {
    db.get('key3', function(err, value) {
    	callback(err, value);
    });
  }
],
function(err, results) {
	// results = [ undefined, undefined, undefined, 'value1', 'value2', 'value3' ]
	var str = '';

	results.forEach(function(result) {
		if (result) {
			str += result + ' - ';
		}
	});

	console.log(str);
});
```



# Named functions and Modules

In my opinion, the best way to escape callback hell is to split your code into modules that do one particular thing with named functions. This is basic code refactoring, but it becomes pretty important when dealing with nested callbacks. My fake database has been wrapped once more  which lets our code be more succinct.

```javascript
var db = require('./db/wrapperDb');

function concatValues(keys, callback) {
	db.getValues(keys, function(err, values) {
		if (err) return callback(err);

		var str = '';

		values.forEach(function(value) {
			str += value + ' - ';
		});

		return callback(null, str);
	})
}

db.setValues(['key1', 'key2', 'key3'], ['value1', 'value2', 'value3'], function(err, keys) {
	if (err) throw err;

	concatValues(keys, function(err, str) {
		if (err) throw err;

		console.log(str);
	});
});
```

Hopefully this article gives developers some more ammunition when faced with nested anonymous functions in their own forays into JavaScript.
