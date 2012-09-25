URL
===
URL parsing done reasonably.

SEOmoz crawls. We crawl lots. In fact, you might say that crawling is our
business.

The internet's also a messy place. We've encountered some pretty crazy
implementations and servers and URLs and HTML. Over the course of this
discovery, we've found ourselves repeating certain URL sanitization tasks over
and over, so we've put them in a repo to share with the world.

At the heart of the `url` package is the `URL` object. You can get one by
passing in a unicode or string object into the top-level `parse` method. If the
string is encoded, you can provide that encoding (otherwise it's assumed to be
utf-8):

    import url

    # It knows about unicode
    myurl = url.parse(u'http://foo.com')

    # It knows about other encodings that Python supports
    myurl = url.parse(..., 'some encoding')

Internally, everything is stored as UTF-8 until you ask for a string back. The
workflow is that you'll chain a number of permutations together to get the type
of URL you're after, and then call a final method to give you a string.

    # Defrag, remove some parameters and give me a unicode string
    url.parse(...).defrag().deparam(['utm_source']).unicode()

    # Escape the path, and punycode the host, and give me a UTF-8 string
    url.parse(...).escape().punycode().utf8()

    # Give me the absolute path url as some encoding
    url.parse(...).abspath().encode('some encoding')

Chaining
========
Many of the methods on the `URL` class can be chained to produce a number of
effects in sequence:

    import url

    # Create a url object
    myurl = url.URL.parse('http://www.FOO.com/bar?utm_source=foo#what')
    # Remove some parameters and the fragment, spit out utf-8
    print myurl.defrag().lower().deparam(['utm_source']).utf8()

In fact, unless the function explicitly returns a string, then the method may
be chained:

`canonical`
-----------
According to the RFC, the order of parameters is not supposed to matter. In
practice, it can (depending on how the server matches URL routes), but it's
also helpful to be able to put parameters in a canonical ordering. This
ordering happens to be alphabetical order:

    >>> url.parse('http://foo.com/?b=2&a=1&d=3').canonical().utf8()
    'http://foo.com/?a=1&b=2&d=3'

`defrag`
--------
Remove any fragment identifier from the url. This isn't part of the reuqest
that gets sent to an HTTP server, and so it's often useful to remove the 
fragment when doing url comparisons.

    >>> url.parse('http://foo.com/#foo').defrag().utf8()
    'http://foo.com/'

`deparam`
---------
Some parameters are commonly added to urls that we may not be interested in. Or
they may be misleading. Common examples include referrering pages, `utm_source`
and session ids. To strip out all such parameters from your url:

    >>> url.parse('http://foo.com/?do=1&not=2&want=3&this=4').deparam(['do', 'not', 'want']).utf8()
    'http://foo.com/?this=4'

`abspath`
---------
Like its `os.path` namesake, this makes sure that the path of the url is
absolute. This includes removing redundant forward slashes, `.` and `..`.

    >>> url.parse('http://foo.com/foo/./bar/../a/b/c/../../d').abspath().utf8()
    'http://foo.com/foo/a/d'

`lower`
-------
The casing of the hostname is not supposed to matter, so...

    >>> url.parse('http://fOo.COM/bar').lower().utf8()
    'http://foo.com/bar'

`escape`
--------
Non-ASCII characters in the path are typically encoded as UTF-8 and then
escaped as `%HH` where `H` are hexidecimal values. It's important to note that
the `escape` function is idempotent, and can be called repeatedly

    >>> url.parse(u'http://foo.com/ümlaut').escape().utf8()
    'http://foo.com/%C3%BCmlaut'
    >>> url.parse(u'http://foo.com/ümlaut').escape().escape().utf8()
    'http://foo.com/%C3%BCmlaut'

`unescape`
----------
If you have a URL that might have been escaped before it was given to you, but
you'd like to display something a little more meaningful than `%C3%BCmlaut`, 
you can unescape the path:

    >>> print url.parse('http://foo.com/%C3%BCmlaut').unescape().unicode()
    http://foo.com/ümlaut

`relative`
----------
Evaluate a relative path given a base url:

    >>> url.parse('http://foo.com/a/b/c').relative('../foo').utf8()
    'http://foo.com/a/foo'

`punycode`
----------
For non-ASCII hostnames, they must be punycoded before a DNS request is made
for them. To this end, there's the punycode function:

    >>> url.parse('http://ümlaut.com').punycode().utf8()
    'http://xn--mlaut-jva.com/'

`unpunycode`
------------
If a url may have been punycoded before it's been handed to you, and you'd like
to be able to display something nicer than `http://xn--mlaut-jva.com/`:

    >>> print url.parse('http://xn--mlaut-jva.com/').unpunycode().utf8()
    http://ümlaut.com/

String Result
=============
Once you've done all the manipulation you're planning to do, you probably want
a string out of it at the end:

- `unicode()` -- return a unicode version of the url
- `utf8()` -- return a utf-8 verison of the url
- `encode(...)` -- return a version of the url in an arbitrary encoding

Contentious Issues
==================
Some questions that I still have outstanding:

Strip ?'s From Query Names?
---------------------------
If I have a query string `?a=1&?b=2`, and I sanitize the params, should the
resulting query string be `?a=1&?b=2` or `?a=1&b=2` (note the missing `?`
before the `b` in the second version).

If not in the above example, what about in `?????a=1`? Should the resulting
query string be a mere `?a=1`?

Authors
=======
This represents code samples, unit tests and functions from SEOmozzers,
including:

- David Barts
- Brandon Forehand
- Dan Lecocq