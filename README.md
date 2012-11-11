# router.js

`router.js` is a lightweight JavaScript library (under 1k!)
that builds on
[`route-recognizer`](https://github.com/tildeio/route-recognizer)
to provide an API for handling routes.

In keeping with the Unix philosophy, it is a modular library
that does one thing and does it well.

## Usage

Create a new router:

```javascript
var router = new Router();
```

Add a simple new route description:

```javascript
router.map(function(match) {
  match("/posts/:id").to("showPost");
  match("/posts").to("postIndex");
  match("/posts/new").to("newPost");
});
```

Add your handlers:

```javascript
router.handlers.showPost = {
  deserialize: function(params) {
    return App.Post.find(params.id);
  },

  setup: function(post) {
    // render a template with the post
  }
};

router.handlers.postIndex = {
  deserialize: function(params) {
    return App.Post.findAll();
  },

  setup: function(posts) {
    // render a template with the posts
  }
};

router.handlers.newPost = {
  setup: function(post) {
    // render a template with the post
  }
};
```

Use another modular library to listen for URL changes, and
tell the router to handle a URL:

```javascript
urlWatcher.onUpdate(function(url) {
  router.handleURL(url);
});
```

The router will parse the URL for parameters and then pass
the parameters into the handler's `deserialize` method. It
will then pass the return value of `deserialize` into the
`setup` method. These two steps are broken apart to support
async loading via **promises** (see below).

To transition into the state represented by a handler without
changing the URL, use `router.transitionTo`:

```javascript
router.transitionTo('showPost', post);
```

If you pass an extra parameter to `transitionTo`, as above,
the router will pass it to the handler's `serialize`
method to extract the parameters. Let's flesh out the
`showPost` handler:

```javascript
router.handlers.showPost = {
  // when coming in from a URL, convert parameters into
  // an object
  deserialize: function(params) {
    return App.Post.find(params.id);
  },

  // when coming in from `transitionTo`, convert an
  // object into parameters
  serialize: function(object) {
    return { id: post.id };
  },

  setup: function(post) {
    // render a template with the post
  }
};
```

## Changing the URL

As a modular library, `router.js` does not express an
opinion about how to reflect the URL on the page. Many
other libraries do a good job of abstracting `hash` and
`pushState` and working around known bugs in browsers.

The `router.updateURL` hook will be called to give you
an opportunity to update the browser's physical URL
as you desire:

```javascript
router.updateURL = function(url) {
  window.location.hash = url;
};
```

## Always In Sync

No matter whether you go to a handler via a URL change
or via `transitionTo`, you will get the same behavior.

If you enter a state represented by a handler through a
URL:

* the handler will convert the URL's parameters into an
  object, and pass it in to setup
* the URL is already up to date

If you enter a state via `transitionTo`:

* the handler will convert the object into params, and
  update the URL.
* the object is already available to pass into `setup`

This means that you can be sure that your application's
top-level objects will always be in sync with the URL,
no matter whether you are extracting the object from the
URL or if you already have the object.

## Asynchronous Loading

When extracting an object from the parameters, you may
need to make a request to the server before the object
is ready.

You can easily achieve this by returning a **promise**
from your `deserialize` method. Because jQuery's Ajax
methods already return promises, this is easy!

```javascript
router.handlers.showPost = {
  deserialize: function(params) {
    return $.getJSON("/posts/" + params.id).then(function(json) {
      return new App.Post(json.post);
    });
  },

  serialize: function(post) {
    return { id: post.get('id') };
  },

  setup: function(post) {
    // receives the App.Post instance
  }
};
```

You can register a `loading` handler for `router.js` to
call while it waits for promises to resolve:

```javascript
router.handlers.loading = {
  // no deserialize or serialize because this is not
  // a handler for a URL

  setup: function() {
    // show a loading UI
  }
}
```

## Nesting

You can nest routes, and each level of nesting can have
its own handler.

If you move from one child of a parent route to another,
the parent will not be set up again unless it deserializes
to a different object.

Consider a master-detail view.

```javascript
router.map(function(match) {
  match("/posts").to("posts", function(match) {
    match("/").to("postIndex");
    match("/:id").to("showPost");
  });
});

router.handlers.posts = {
  deserialize: function() {
    return $.getJSON("/posts").then(function(json) {
      return App.Post.loadPosts(json.posts);
    });
  },

  // no serialize needed because there are no
  // dynamic segments

  setup: function(posts) {
    var postsView = new App.PostsView(posts);
    $("#master").append(postsView.el);
  }
};

router.handlers.postIndex = {
  setup: function() {
    $("#detail").hide();
  }
};

router.handlers.showPost = {
  deserialize: function(params) {
    return $.getJSON("/posts/" + params.id, function(json) {
      return new App.Post(json.post);
    });
  }
};

router.handlers.loading = {
  setup: function() {
    $("#content").hide();
    $("#loading").show();
  },

  exit: function() {
    $("#loading").hide();
    $("#content").show();
  }
};
```

You can also use nesting to build nested UIs, setting up the
outer view when entering the handler for the outer route,
and setting up the inner view when entering the handler for
the inner route.

Routes at any nested level can deserialize parameters into a
promise. The router will remain in the `loading` state until
all promises are resolved. If a parent state deserializes
the parameters into a promise, that promise will be resolved
before a child route is handled.

### Nesting Without Handlers

You can also nest without extra handlers, for clarity.

For example, instead of writing:

```javascript
router.map(function(match) {
  match("/posts").to("postIndex");
  match("/posts/new").to("newPost");
  match("/posts/:id/edit").to("editPost");
  match("/posts/:id").to("showPost");
});
```

You could write:

```javascript
router.map(function(match) {
  match("/posts", function(match) {
    match("/").to("postIndex");
    match("/new").to("newPost");

    match("/:id", function(match) {
      match("/").to("showPost");
      match("/edit").to("editPost");
    });
  });
});
```

Typically, this sort of nesting is more verbose but
makes it easier to change patterns higher up. In this
case, changing `/posts` to `/pages` would be easier
in the second example than the first.

Both work identically, so do whichever you prefer.

## Route Recognizer

`router.js` uses `route-recognizer` under the hood, which
uses an [NFA](http://en.wikipedia.org/wiki/Nondeterministic_finite_automaton)
to match routes. This means that even somewhat elaborate
routes will work:

```javascript
router.map(function(match) {
  // this will match anything, followed by a slash,
  // followed by a dynamic segment (one or more non-
  // slash characters)
  match("/*page/:location").to("showPage");
});
```

If there are multiple matches, `route-recognizer` will
prefer routes with fewer dynamic segments, so
`/posts/edit` will match in preference to `/posts/:id`
if both match.

## More to Come

`router.js` is functional today. I plan to add more features
before a first official release:

* A `failure` handler if any of the promises are rejected
* An `exit` callback on a handler when the app navigates
  to a page no longer represented by the handler
* Improved hooks for external libraries that manage the
  physical URL.
* Testing support
* The ability to dispatch events to the current handler
  or parent handlers.

`router.js` will be the basis for the router in Ember.js.