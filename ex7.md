Use an image from Docker Hub
==================================

This tutorial covers:

1. Creating a new system
2. Hooking up a new container definition from the Registry
3. Building a new container
4. Run!

Create a new System
-------------------

As done in [ex2.md](https://github.com/nearform/nscale-workshop/blob/master/ex2.md), let's go ahead and create a new system with:

```bash
$ nsd sys create
prompt: name:  registry
prompt: namespace:  nscale
create system: workshop with namespace: nscale?
prompt: confirm (y/n):  y
ok
```

Now we can check that everything is as expected:

```bash
$ nsd sys list
Name                           Id
registry                       2de30af9-fdc4-41ff-9b88-cd47eacb7f77
```

Add a registry container definition
-----------------------------------

Let's open `definitions/services.js` in you favorite editor. It
currently looks like:

```js
exports.root = {
  type: 'container'
};

// add here more definitions
```

To begin defininig our system, we need to change it to:

```js
exports.root = {
  type: 'container'
};

exports.web = {
  type: 'registry',
  specific: {
    name: 'redis',
    execute: {
      args: '-d -p 6379:6379'
    }
  }
};
```

Build a container
-----------------

Let's open `system.js` in you favorite editor. It currently looks like this:

```js
exports.name = 'registry';
exports.namespace = 'nscale';
exports.id = '2de30af9-fdc4-41ff-9b88-cd47eacb7f77';

exports.topology = {
  local: {
  }
};
```

This system is empty, let's add our containers:

```js
exports.name = 'registry';
exports.namespace = 'nscale';
exports.id = '2de30af9-fdc4-41ff-9b88-cd47eacb7f77';

exports.topology = {
  local: {
    root: ['redis']
  }
};
```

```js
nsd system compile workshop local
```

Now, let's buid our containers:

```bash
$ nsd container buildall
```

We'll check the revision list again:

```bash
$ nsd rev list workshop
revision             deployed who                                                     time                      description
15dffc828de9d971ba1… false    Matteo Collina <hello@matteocollina.com>                2014-10-30T10:54:04.000Z  built container: redis
a

```

Deploy
------

All we have to do now is deploy:

```bash
$ nsd rev deploy workshop 15dff
```

Our container should be running just fine, we can use the following to see it in action:

OS X : 
```bash
$ redis-cli -h `boot2docker ip`
```

Linux:
```bash
$ redis-cli
```
