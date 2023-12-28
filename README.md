# twitstream #

A super-simple asynchronous python library for speaking with Twitter's
[streaming API][]. Implemented basic authentication and non-authenticating
proxy support in the rudimentary HTTP client.

The idea was to make the least effort possible to get things working. All of
the standard HTTP client libraries seemed to block until the end of
transmission, making them inappropriate for use with the streaming API.

For a Twisted solution, see [twitty-twister][]. Other interesting Python
modules using the streaming API via different approaches are [tweepy][] and
[instwitter-py][].

[streaming API]: http://apiwiki.twitter.com/Streaming-API-Documentation
[twitty-twister]: http://github.com/dustin/twitty-twister
[tweepy]: http://github.com/joshthecoder/tweepy
[instwitter-py]: http://github.com/sovnarkom/instwitter-py

## Requirements ##

Python 2.5 or higher. If using Python 2.5, also uses [simplejson][] (which is
included in Python 2.6 as [json][]). In order to achieve SSL support, the 
asyncore implementation now depends upon [tlslite][]. 
There are no requirements beyond that.

The more elaborate example programs `fixreplies.py` and `textori.py` require
the [python-twitter][] library.

Because I like to offer flexibility, and started out not having the utmost
confidence in my original hacked-together HTTP client (although it's stood up
fairly well), you have a _choice_ of asynchronous IO "engines." If you have
[PycURL][] installed, you may choose its time-tested, non-blocking HTTP client
implementation with a `--curl` command-line option used with any of the
example applications. If you have Facebook/FriendFeed's open-source
[Tornado][], then you can choose its [iostream][] sub-module with a
`--tornado` option. In all other cases, it falls back to the basic
implementation built upon the standard library invoked with `--async`. The
`twitasync.py`, `twittornado.py`, and `twitcurl.py` modules transparently
expose the same basic interface to the main `twitstream` module: all typical
usage will focus on the `twitstream` module.

[simplejson]: http://pypi.python.org/pypi/simplejson/
[json]: http://docs.python.org/library/json.html
[tlslite]: http://pypi.python.org/pypi/tlslite
[python-twitter]: http://code.google.com/p/python-twitter/
[PycURL]: http://pycurl.sourceforge.net/
[Tornado]: http://www.tornadoweb.org/
[IOStream]: http://github.com/facebook/tornado/blob/master/tornado/iostream.py

# Usage #

Twitstream-test is usable from the command line as a very rudimentary client:

    twitstream-test.py spritzer
    twitstream-test.py track ftw fail
    twitstream-test.py follow 12 13 15 16 20 87

Every usage of the streaming API requires authentication against a user
account. The methods available to the general public are `spritzer`, `track`,
and `follow`.

## Example applications ##
### textori ###

As a simple implementation of a tweet display roughly modeled on [twistori][],
`textori.py` takes in keywords and pretty-prints a live `track`ing stream from
the keywords entered. The below-listed keywords are used as the default
setting, when no keywords are entered.

    textori.py love hate think believe feel wish

The code in this example is most notable for the tweet text unescaping and
parsing all accomplished in a single (lengthy) callable.

[twistori]: http://twistori.com/

### fixreplies ###

As a proof-of-concept, there's the modestly-named `fixreplies.py`, which mines
your friends, followers, favorites and/or conversations to derive a list of
people to follow (which can cause a lot of API calls at startup). It then uses
the [streaming API][]'s `follow` method to get all tweets to and from those
chosen users. For example, the following command line will check your latest
500 status messages for people to whom you've replied, and filters out the
people you do not already follow as well as a couple celebrities that everyone
seems to empathize with:

    fixreplies.py --pages 5 --chat --friends --exclude=stephenfry,Oprah

For Mac users, there is a `--growl` option, which uses the [Growl][]
notification framework and its Python interface available in the 
[Growl SDK][]. The class does its best at distinguishing between categories 
of status messages, allowing a user to change display options.

This code example uses a variant upon the status pretty-printing of the
textori example. The chief purpose of this example is to use Twitter's
traditional API in order to get more use out of the streaming API. 

[Growl]: http://growl.info/
[Growl SDK]: http://growl.info/downloads_developers.php

### stats ###

A proof-of-concept showing that you don't need to print out every tweet in the
callback. `stats.py` sets up a counter/histogram on the status characteristic
desired. When halted (interrupted with ctrl-C or a `KeyboardInterrupt`), it
prints a summary of the statistic collected.

    stats.py friends
    stats.py timezone --max 15

### warehouse ###

If you want to examine statistics off-line, the latest batch of schema-free
JSON document stores, like [MongoDB][] or Apache [CouchDB][], make for good
candidates. `warehouse.py` runs the `spritzer` method and stores each status
message in the designated data store. The implementation currently includes
adaptors for MongoDB and CouchDB, and would welcome models for your favorite
ORM+RDBMS.

    warehouse.py
    warehouse.py mongo://localhost:27017/db/twitcollection

The most notable addition in this example is the correct handling of `delete`
updates: it attempts to delete the referenced status message if it is in the
database.

[MongoDB]: http://www.mongodb.org/
[CouchDB]: http://couchdb.apache.org/

# Programming #

The interface provides relatively low-level specialized streaming HTTP GET and
POST classes (currently geared specifically towards Twitter, and provided in
[asynchat][], [tornado.iostream][], and [libcurl][] flavors), a general
`twitstream` function that accepts an API method name and routes the software
there, and individual functions that match the API methods (including
`spritzer`, `track`, and `follow`). Each of these returns a request-like
object that, when invoked with the `run()` method, opens the connection and
continues into a loop until interrupted. The only programming you need to
provide is a function (or callable) that gets called with a dictionary
containing the latest single status.

[asynchat]: http://docs.python.org/library/asynchat.html
[libcurl]: http://curl.haxx.se/libcurl/
[tornado.iostream]: http://github.com/facebook/tornado/blob/master/tornado/iostream.py

For example, the basic
[spritz.py](http://github.com/atl/twitstream/blob/master/spritz.py) example
shows the minimum amount of work needed to have a fully working program, using
some built-in facilities for command-line option processing, documentation,
and prompting for a username and password. For something truly minimal, you
could use something like this:

    #!/usr/bin/env python
    
    import twitstream
    
    USER = 'test'
    PASS = 'test'
    
    # Define a function/callable to be called on every status:
    def callback(status):
        print "%s:\t%s\n" % (status.get('user', {}).get('screen_name'), status.get('text'))
    
    if __name__ == '__main__':
        # Call a specific API method from the twitstream module: 
        stream = twitstream.spritzer(USER, PASS, callback)
        
        # Loop forever on the streaming call:
        stream.run()
