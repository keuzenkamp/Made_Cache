Magento Block Cache & Varnish extension
==

Why?
--
Few know that Magento out of the box actually doesn't cache any frontend blocks other than Navigation and Footer, which are basically static as they are. This module enhances performance by giving developers a simple interface for caching any block they want, and comes with good default settings.

Features
--
* Quick & Versatile Performance Boost
* Varnish + ESI support
* Unobtrusive & Future Proof
* Simple Configuration

Installation
--
Install this module using [modman](https://github.com/colinmollenhour/modman)

`modman clone git@github.com:madepeople/Made_Cache.git`

Or by downloading a copy from [Magento Connect](http://www.magentocommerce.com/magento-connect/made-cache-9281.html)

Magento Configuration
--
Most of the configuration is done via layout XML. In general it comes down to choosing which blocks to cache (or not), and which ones to fetch via ESI (or not).

For instance, to cache the products.list block on every cms\_page for 7200 seconds:

```xml
<layout version="0.1.0">
    <cms_page>
        <cache>
            <name lifetime="7200">products.list</name>
        </cache>
    <cms_page>
</layout>
```

These tags exist:

**cache** - Used to group which blocks should be cached

**nocache** - Used to group which block should _not_ be cached

**esi** (Varnish) - Generates an ESI tag in place of the block

**noesi** (Varnish) - Excludes a block from the ESI tag generation

**name** - Used inside the above to determine which blocks should be used

See [madecache.xml](https://github.com/madepeople/Made_Cache/blob/master/frontend/layout/madecache.xml) for more details.

Varnish, ESI & Sessions
==
A custom magento.vcl file is available in the etc/ directory of the module. With Varnish in front and using this VCL, you can harness full page caching.

ESI
--
In order to handle dynamic user-dependent blocks, something called ESI (Edge Side Includes) is used. With this in place, Varnish makes an extra request to the backend for each dynamic block, such as the cart, compared items, etc.

General Configuration
--

* Use magento.vcl with your Varnish instance and modify its IP settings in the top
* For ESI to work properly, it's a good idea to add `-p esi_syntax=0x1` to the Varnish command line
* Set up your Varnish server's IP in System / Configuration / Made People / Cache
* Enable "Varnish" in the Magento Cache Management page
* Flush everything

The layout handle _varnish\_enabled_ is added to every request when Varnish is in front.

Sessions
--
Because Varnish sits in front of Magento itself, we need to have a way to validate sessions, otherwise Varnish has to pass every request to the backend as soon as a session is in place. Out of the box, this module adds a special cookie AJAX request to the bottom of the page which will send a new session if the visitor doesn't have one. This approach means that there will *always* be a request to the backend. The effect is that the user gets the idea of a super-fast loading page, but your backend still gets hit and might not be able to handle the thousands of requests you want it to.

It is technically possible for Varnish to check the actual session storage directly in the actual VCL, but this methodThere are different sessions storage mechanisms available for Magento. Each with their own drawbacks:

* **Files** - Hard to distribute on a network, and has locking issues
* **Memcache** - Fast, but no persistence and no locking
* **MySQL** - Has persistence and locking, but performance is subject to all query noise that Magento creates

Apart from these drawbacks, making Varnish to talk to the different storages isn't very straight forward.

Given the above, I have experimented with using Redis and [CouchDB](https://github.com/madepeople/Made_CouchdbSession):

CouchDB pros:

* MVCC instead of locking gives performance and consistency
* Queried via a HTTP interface that Varnish can talk to easily using libvmod-curl
* A JavaScript _show_ function can be used to determine session validity inside of CouchDB itself, and return a boolean to Varnish. If the boolean is "true", we know it's safe to serve the visitor content from cache.

CouchDB cons:

* The compaction process takes a lot of time and can make the general performance suffer
* Every request means a new write, meaning the size of the database grows rapidly

Redis pros:

* Build-in optimistic locking
* In-memory with optional persistance
* [A Varnish Redis client exists](https://github.com/brandonwamboldt/libvmod-redis)

Redis cons:

* An improperly configured Redis daemon can out of memory crash and make break a site


Varnish Session Validation
==

Initial Setup
--
We will be using Debian/Ubuntu steps for reference. First of all the sources to a built Varnish package need to exist since we want to build VMODs.

```bash
apt-get update
apt-get install build-essential dpkg-dev debhelper libedit-dev libncurses-dev libpcre3-dev python-docutils xsltproc libvarnishapi-dev autoconf automake autotools-dev libtool pkg-config

mkdir -p varnish/out
cd varnish
apt-get -b source varnish
```

**IMPORTANT!** If an apt-get upgrade also upgrades varnish, you *have* to recompile libvmod-curl again, using the whole procedure from `apt-get -b source varnish` and forward.

CouchDB Configuration
--
First, install and configure [Made_CouchdbSession](https://github.com/madepeople/Made_CouchdbSession) and make sure it's working. After this, Varnish needs [libvmod-curl](https://github.com/varnish/libvmod-curl) installed. For reference, here are Debian instructions:

```bash
apt-get install libcurl3-dev
git clone https://github.com/varnish/libvmod-curl.git
cd libvmod-curl
./autogen.sh
./configure --prefix=$PWD/../out VARNISHSRC=../varnish-*
make
make install
```

With vmod-curl and the show function above in place, search for "curl" in the magento.vcl file and uncomment the affected lines. Then just restart Varnish and you should be good to go.

Redis Configuration
--

We just need to install the redis vmod the same way as the curl vmod

```bash
apt-get install libhiredis-dev
git clone https://github.com/brandonwamboldt/libvmod-redis.git
cd libvmod-redis
./autogen.sh
./configure --prefix=$PWD/../out VARNISHSRC=../varnish-*
make
make install
```

FAQ
==

Will Made\_Cache interfere with other modules?
--
Hopefully not. Events are used instead of block rewrites, and no core functionality is modified. This means that there will be less interference with other modules, and that manual block cache settings are preserved.

Another Varnish implementation?
--
That's right. The nice thing with this implementation is automatic ESI tag generation and session invalidation. We try to cache as much as we can without messing with standard installations. It also supports caching ESI requests on a user-level, meaning the majority of the requests come directly from Varnish (super fast).

License
--
This project is licensed under the 4-clause BSD License, see [LICENSE](https://github.com/madepeople/Made_Cache/blob/master/LICENSE)
