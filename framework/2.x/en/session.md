---
layout: docs2-en
title: Session
permalink: /framework/2.x/en/session/
previous_page: Internationalization
previous_page_url: /framework/2.x/en/intl/
next_page: Command line / console
next_page_url: /framework/2.x/en/cli/
---

# Session

Provides session management functionality, including lazy session starting,
session segments, next-request-only ("flash") values, and CSRF tools.

## Installation

We are going to install `aura/session` version `2.0.*@beta` .

Add to your `composer.json`.

{% highlight json %}
{
    "require": {
        //
        "aura/session": "2.0.*@beta"
    }
}
{% endhighlight %}

and run

{% highlight bash %}
composer update
{% endhighlight %}

## Service

Aura.Session already have `aura/session:session` service which is an object of _Aura\\Session\\Session_ . You can get inject the service to responder or view helper and make use of the _Aura\\Session\\Session_ object.

## Segments

In normal PHP, we keep session values in the `$_SESSION` array. However, when different libraries and projects try to modify the same keys, the resulting conflicts can result in unexpected behavior. To resolve this, we use _Segment_ objects. Each _Segment_ addresses a named key within the `$_SESSION` array for deconfliction purposes.

For example, if we get a _Segment_  for `Vendor\Package\ClassName`, that _Segment_ will contain a reference to `$_SESSION['Vendor\Package\ClassName']`. We can then `set()` and `get()` values on the _Segment_, and the values will reside in an array under that reference.

{% highlight php %}
<?php
// get a _Segment_ object
$segment = $session->getSegment('Vendor\Package\ClassName');

// try to get a value from the segment;
// if it does not exist, return an alternative value
echo $segment->get('foo'); // null
echo $segment->get('baz', 'not set'); // 'not set'

// set some values on the segment
$segment->set('foo', 'bar');
$segment->set('baz', 'dib');

// the $_SESSION array is now:
// $_SESSION = array(
//      'Vendor\Package\ClassName' => array(
//          'foo' => 'bar',
//          'baz' => 'dib',
//      ),
// );

// try again to get a value from the segment
echo $segment->get('foo'); // 'bar'

// because the segment is a reference to $_SESSION, we can modify
// the superglobal directly and the segment values will also change
$_SESSION['Vendor\Package\ClassName']['zim'] = 'gir'
echo $segment->get('zim'); // 'gir'
?>
{% endhighlight %}

The benefit of a session segment is that we can deconflict the keys in the `$_SESSION` superglobal by using class names (or some other unique name) for the segment names. With segments, different packages can use the `$_SESSION` superglobal without stepping on each other's toes.

To clear all the values on a _Segment_, use the `clear()` method.


## Lazy Session Starting

Merely instantiating the _Session_ manager and getting a _Segment_ from it does *not* call `session_start()`. Instead, `session_start()` occurs only in certain circumstances:

- If we *read* from a _Segment_ (e.g. with `get()`) the _Session_ looks to see if a session cookie has already been set. If so, it will call `session_start()` to resume the previously-started session. If not, it knows there are no previously existing `$_SESSION` values, so it will not call `session_start()`.

- If we *write* to a _Segment_ (e.g. with `set()`) the _Session_ will always call `session_start()`. This will resume a previous session if it exists, or start a new one if it does not.

This means we can create each _Segment_ at will, and `session_start()` will not be invoked until we actually interact with a _Segment_ in a particular way. This helps to conserve the resources involved in starting a session.

Of course, we can force a session start or reactivation by calling the _Session_ `start()` method, but that defeats the purpose of lazy-loaded sessions.


## Saving, Clearing, and Destroying Sessions

> These methods apply to all session data and flashes across all segments.

To save the session data and end its use during the current request, call the `commit()` method on the _Session_ manager:

{% highlight php %}
<?php
$session->commit(); // equivalent of session_write_close()
?>
{% endhighlight %}

> The `commit()` method is the equivalent of `session_write_close()`.
> If you do not commit the session, its values will not be available when we
> continue the session later.

To clear all session data, but leave the session active during the current request, use the `clear()` method on the _Session_ manager.

{% highlight php %}
<?php
$session->clear();
?>
{% endhighlight %}

To clear all flash values on a segment, use the `clearFlash()` method:

To clear the data *and* terminate the session for this and future requests, thereby destroying it completely, call the `destroy()` method:

{% highlight php %}
<?php
$session->destroy(); // equivalent of session_destroy()
?>
{% endhighlight %}

Calling `destroy()` will also delete the session cookie via `setcookie()`. If we have an alternative means by which we delete cookies, we should pass a callable as the second argument to the _SessionFactory_ method `newInstance()`. The callable should take three parameters: the cookie name, path, and domain.

{% highlight php %}
<?php
// assume $response is a framework response object.
// this will be used to delete the session cookie.
$delete_cookie = function ($name, $path, $domain) use ($response) {
    $response->cookies->delete($name, $path, $domain);
}

$session = $session_factory->newInstance($_COOKIE, $delete_cookie);
?>
{% endhighlight %}

## Session Security

### Session ID Regeneration

Any time a user has a change in privilege (that is, gaining or losing access
rights within a system) be sure to regenerate the session ID:

{% highlight php %}
<?php
$session->regenerateId();
?>
{% endhighlight %}

> The `regenerateId()` method also regenerates the CSRF token value.

### Cross-Site Request Forgery

A "cross-site request forgery" is a security issue where the attacker, via
malicious JavaScript or other means, issues a request in-the-blind from a
client browser to a server where the user has already authenticated. The
request *looks* valid to the server, but in fact is a forgery, since the user
did not actually make the request (the malicious JavaScript did).

<http://en.wikipedia.org/wiki/Cross-site_request_forgery>

#### Defending Against CSRF

To defend against CSRF attacks, server-side logic should:

1. Place a token value unique to each authenticated user session in each form;
   and

2. Check that all incoming POST/PUT/DELETE (i.e., "unsafe") requests contain
   that value.

> If our application uses GET requests to modify resources (which
> incidentally is an improper use of GET), we should also check for CSRF on
> GET requests from authenticated users.

For this example, the form field name will be `__csrf_value`. In each form
we want to protect against CSRF, we use the session CSRF token value for that
field:

{% highlight php %}
<?php
/**
 * @var Vendor\Package\User $user A user-authentication object.
 * @var Aura\Session\Session $session A session management object.
 */
?>
<form method="post">

    <?php if ($user->auth->isValid()) {
        $csrf_value = $session->getCsrfToken()->getValue();
        echo '<input type="hidden" name="__csrf_value" value="'
           . htmlspecialchars($csrf_value, ENT_QUOTES, 'UTF-8')
           . '"></input>';
    } ?>

    <!-- other form fields -->

</form>
{% endhighlight %}

When processing the request, check to see if the incoming CSRF token is valid
for the authenticated user:

{% highlight php %}
<?php
/**
 * @var Vendor\Package\User $user A user-authentication object.
 * @var Aura\Session\Session $session A session management object.
 */

$unsafe = $_SERVER['REQUEST_METHOD'] == 'POST'
       || $_SERVER['REQUEST_METHOD'] == 'PUT'
       || $_SERVER['REQUEST_METHOD'] == 'DELETE';

if ($unsafe && $user->auth->isValid()) {
    $csrf_value = $_POST['__csrf_value'];
    $csrf_token = $session->getCsrfToken();
    if (! $csrf_token->isValid($csrf_value)) {
        echo "This looks like a cross-site request forgery.";
    } else {
        echo "This looks like a valid request.";
    }
} else {
    echo "CSRF attacks only affect unsafe requests by authenticated users.";
}
?>
{% endhighlight %}

#### CSRF Value Generation

For a CSRF token to be useful, its random value must be cryptographically
secure. Using things like `mt_rand()` is insufficient. Aura.Session comes with
a `Randval` class that implements a `RandvalInterface`, and uses either the
`openssl` or the `mcrypt` extension to generate a random value. If you do not
have one of these extensions installed, you will need your own random-value
implementation of the `RandvalInterface`. We suggest a wrapper around
[RandomLib](https://github.com/ircmaxell/RandomLib).


## Flash Values

_Segment_ values persist until the session is cleared or destroyed. However, sometimes it is useful to set a value that propagates only through the next request, and is then discarded. These are called "flash" values.

### Setting And Getting Flash Values

To set a flash value on a _Segment_, use the `setFlash()` method.

{% highlight php %}
<?php
$segment = $session->getSegment('Vendor\Package\ClassName');
$segment->setFlash('message', 'Hello world!');
?>
{% endhighlight %}

Then, in subsequent requests, we can read the flash value using `getFlash()`:

{% highlight php %}
<?php
$segment = $session->getSegment('Vendor\Package\ClassName');
$message = $segment->getFlash('message'); // 'Hello world!'
?>
{% endhighlight %}

> As with `get()`, we can provide an alternative value if the flash key
> does not exist. For example, `getFlash('foo', 'not set')` will
> return 'not set' if there is no 'foo' key available.

Using `setFlash()` makes the flash value available only in the *next* request, not the current one. To make the flash value available immediately as well as in the next request, use `setFlashNow($key, $val)`.

Using `getFlash()` returns only the values that are available now from having been set in the previous request. To read a value that will be available in the next request, use `getFlashNext($key, $alt)`.

### Keeping and Clearing Flash Values

Sometimes we will want to keep the flash values in the current request for the next request.  We can do so on a per-segment basis by calling the _Segment_ `keepFlash()` method, or we can keep all flashes for all segments by calling the _Session_ `keepFlash()` method.

Similarly, we can clear flash values on a per-segment basis or a session-wide bases.  Use the `clearFlash()` method on the _Segment_ to clear flashes just for that segment, or the same method on the _Session_ to clear all flash values for all segments.
