API Spec
========

This API runs over HTTP, preferably HTTPS.

Design Goals
------------

* Discovery endpoint
* Unified API Scheme for Daemons & Console.
  Think of the Console Server as a proxy for all your pdns deployments.
* Have API docs (this!) for other consumers

Data format
-----------

Input data format: JSON.

Output data formats: JSON, JSONP

All GET requests support appending a `_callback` URL parameter, which, if
present, will turn the response into a JSONP response.

The `Accept:` header determines the output format. An unknown value or
`\*/*` will cause a `400 Bad Request`.

All text is UTF-8 and HTTP headers will reflect this.

Data types:

  * empty fields: `null` but present
  * Regex: implementation defined
  * Dates: ISO 8601


REST
----

* GET: List/Retrieve. Success reply: `200 OK`
* POST: Create. Success reply: `201 Created`, `Location:` header points
  to new object
* PUT: Update. Success reply: `204 No Content`
* DELETE: Delete. Success reply: `200 OK`

not-so-REST
-----------

For interactions that do not directly map onto CRUD, we use these:

* GET: Query. Success reply: `200 OK`
* PUT: Action/Execute. Success reply: `200 OK`


Authentication
--------------

Clients SHOULD support:

* HTTP Basic Auth (used by pdns, pdnsmgrd)
* OAuth (used by pdnscontrol)
  * **TODO**: still very much open.

Errors
------

Response code `4xx` or `5xx`, depending on the situation. Never return `2xx`
for an error!

* Invalid JSON body from client: `400 Bad Request`
* JSON body from client not a hash: `400 Bad Request`
* Input validation failed: `422 Unprocessable Entity`

Error responses have a JSON body of this format:

    {
      "message": "short error message",
      "errors": [
        { ... },
      ]
    }

Where `errors` is optional, and the contents are error-specific.


URL: /
------

Allowed methods: `GET`

    {
      "server_url": "/servers{/server}",
      "api_features": [],
    }

**TODO**:

* `api_features`
  * `servers_modifiable`
  * `oauth`


General Collections Interface
=============================

Collections generally support `GET` and `POST` with these meanings:

GET
---

Retrieve a list of all entries.

The special `type` and `url` fields are included in the response objects:

  * `type`: name of the resource type
  * `url`: url to the object


Response format:

    [
      obj1
      [, further objs]
    ]

Example:

    [
      {
        "type": "AType",
        "id": "anid",
        "url": "/atype/anid",
        "a_field": "a_value"
      },
      {
        "type": "AType",
        "id": "anotherid",
        "url": "/atype/anotherid",
        "a_field": "another_value"
      }
    ]


POST
----

Create a new entry. The client has to supply the entry in the request body,
in JSON format. `application/x-www-form-urlencoded` data MUST NOT be sent.

Clients SHOULD not send the 'url' field.

Client body:

    obj1

Example:

    {
      "type": "AType",
      "id": "anewid",
      "a_field": "anew_value"
    }




Servers
=======

**TODO**: further routes


server_resource
---------------

Example with server `"localhost"`, which is the only server returned by pdns.

pdnsmgrd and pdnscontrol MUST NOT return “localhost”, but SHOULD return
other servers.

    {
      "type": "Server",
      "id": "localhost",
      "url": "/servers/localhost",
      "daemon_type": "recursor",
      "config_url": "/servers/localhost/config{/config_setting}",
      "zones_url": "/servers/localhost/zones{/zone}",
    }

Note: On a pdns server, the servers collection is read-only, and the only
allowed returned server is read-only as well.
On a pdnscontrol server, the servers collection is read-write, and the
returned server resources are read-write as well. Write permissions may
depend on the credentials you have supplied.

URL: /servers
-------------

Collection access.

Allowed REST methods:

* pdns: `GET`
* pdnsmgrd: `GET`
* pdnscontrol: `GET`, `PUT`, `POST`, `DELETE`


URL: /servers/:server\_id
-------------------------

Returns a single server_resource.



Config
======


config\_setting\_resource
-------------------------

    {
       "type": "ConfigSetting",
       "name": "config_setting_name",
       "value": "config_setting_value"
    }

**TODO**: do we know the type of the config value? - no, internally everything is string. Could be changed.

URL: /servers/:server\_id/config
--------------------------------

Collection access.

Allowed REST methods: `GET`, `POST`

#### POST

Creates a new config setting. This is useful for creating configuration for new backends.


URL: /servers/:server\_id/config/:config\_setting\_name
-------------------------------------------------------

Allowed REST methods: `GET`, `PUT`


Zones
=====

Authoritative DNS Zones.

zone_collection
---------------

    {
      "name": "zone_name",
      "type": "Zone",
      "url": "/servers/:server_id/zones/:zone_name",
      "kind": "<zone_kind>",
      "serial": <int>,
      "notified_serial": <int>,
      "master": "<ip>",
      "dnssec": <bool>,
      "nsec3param": "<nsec3param record>",
      "nsec3narrow": <bool>,
      "presigned": <bool>
    }


##### Parameters:

* `kind`
  `<zone_kind>`: `Native`, `Master` or `Slave`

* `dnssec`
  inferred from `presigned` being `true` XOR presence of at
  least one cryptokey with `active` being `true`.

  Switching `dnssec` to `true` (from `false`) sets up DNSSEC signing
  based on the other flags, this includes running the equivalent of
  `secure-zone` and `rectify-zone`. This also applies to newly created
  zones.
  If `presigned` is `true`, no DNSSEC changes will be made to the zone
  or cryptokeys.

* `notified_serial`, `serial` MUST NOT be sent in client bodies.


##### Notes:

Turning on DNSSEC with custom keys: just create the zone with `dnssec`
set to `false`, and add keys using the cryptokeys REST interface. Have
at least one of them `active` set to `true`.

Changes made through the Zones API will always yield valid zone data,
and the zone will be properly "rectified". If changes are made through
other means (e.g. direct database access), this is not guranteed to be
true and clients SHOULD trigger rectify.

Backends might implement additional features (by coincidence or not).
These things are not supported through the API. (gsqlbackend multiple
masters is an example of this.)


URL: /servers/:server\_id/zones
-------------------------------

Allowed REST methods: `GET`, `POST`

#### POST
Creates a new domain.

* `dnssec`, `nsec3narrow`, `presigned`, `nsec3param`, `active-keys` are OPTIONAL.
* `dnssec`, `nsec3narrow`, `presigned` default to `false`.
* The server MUST create a SOA record. The created SOA record SHOULD have
serial set to 1 and use the nameserver name specified in `default-soa-name`
in the pdns configuration.


URL: /servers/:server\_id/zones/:zone\_name
-------------------------------------------

Allowed methods: `GET`, `PUT`, `DELETE`

#### GET
Returns zone information.

#### DELETE
Deletes this zone and all attached metadata, rrsets.


URL: /servers/:server\_id/zones/:zone\_name/notify
--------------------------------------------------

Allowed methods: `PUT`

Send a DNS NOTIFY to all slaves.

Fails when zone kind is not `Master` or `Slave`, or `master` and `slave` are 
disabled in pdns configuration. Only works for `Slave` if renotify is on.

Not supported for recursors.

Clients MUST NOT send a body.


URL: /servers/:server\_id/zones/:zone\_name/axfr-retrieve
---------------------------------------------------------

Allowed methods: `PUT`

Retrieves the zone from the master.

Fails when zone kind is not `Slave`, or `slave` is disabled in pdns
configuration.

Not supported for recursors.

Clients MUST NOT send a body.


URL: /servers/:server\_id/zones/:zone\_name/rectify
---------------------------------------------------

Allowed methods: `PUT`

Rectifies a zone, regarding to auth. Server MUST NOT depend on consumers
ever sending this, AS LONG AS the server is the only thing ever writing
to the datastore.

If the datastore has been written to using other means than this API,
consumers MUST trigger a rectify, using either this API call or any 
other method.

Clients MUST NOT send a body.


URL: /servers/:server\_id/zones/:zone\_name/check
-------------------------------------------------

Allowed methods: `GET`

Verify zone contents/configuration.

Return format:

    {
      "zone": "<zone_name>",
      "errors": ["error message1", ...],
      "warnings": ["warning message1", ...]
    }


Zone Record Names
=================

**TODO**: This section is underdeveloped.

URL: /servers/:server\_id/zones/:zone\_name/names/:name
--------------------------------------------------------

Allowed methods: `GET`, `POST`, `DELETE`. Returns collection of RRsets keyed by type.

URL: /servers/:server\_id/zones/:zone\_name/names/:name/rrsets/:rrtype
-----------------------------------------------------------------------

Allowed methods: `GET`, `PUT`, `DELETE`. PUT replaces full rrset!

    {
      "rrs": 
        [
          {
           "content":"1.1.1.1",
           "name":"foo.ds9b.nl",
           "priority":"0",
           "ttl":"0",
           "type":"A"
          }
        ]
    }

Having `type` inside an RR differ from :rrtype in the URL is an error.

Zone Metadata
=============

zone\_metadata\_resource
------------------------

    {
      "type": "Metadata",
      "kind": <metadata_kind>,
      "metadata": [
        "value1",
        ...
      ]
    }

Valid values for `<metadata_kind>` are specified in <http://doc.powerdns.com/domainmetadata.html>.

Clients MUST NOT modify `NSEC3PARAM`, `NSEC3NARROW` or `PRESIGNED`
through this interface. The server SHOULD reject updates to these
metadata.


URL: /servers/:server\_id/zones/:zone\_name/metadata
----------------------------------------------------

Collection access.

Allowed methods: `GET`, `POST`

URL: /servers/:server\_id/zones/:zone\_name/metadata/:metadata\_kind
--------------------------------------------------------------------

Allowed methods: `GET`, `PUT`, `DELETE`


CryptoKeys
==========

cryptokey\_resource
-------------------

    {
      "type": "CryptoKey",
      "id": <int>,
      "active": <bool>,
      "flags": [<key_flag>, ...]
      "content": <string>
    }


##### Parameters:

`id`: read-only.

`flags`: `<key_flag>` is one of the following: `ksk` or `zsk`, and they are
both mutually exclusive. All other flags are reserved.


URL: /servers/:server\_id/zones/:zone\_name/cryptokeys
------------------------------------------------------

Collection access.

Allowed methods: `GET`, `POST`

#### POST

Creates a new, single cryptokey.

##### Parameters:

`content`: if `null`, pdns generates a new key. In this case, the
following additional fields MAY be supplied:

* `bits`: `<int>`
* `algo`: `<algo>`

Where `<algo>` is one of the supported key algos in lowercase OR the
numeric id, see
[http://rtfm.powerdns.com/pdnssec.html](http://rtfm.powerdns.com/pdnssec.html)

URL: /servers/:server\_id/zones/:zone\_name/cryptokeys/:cryptokey\_id
---------------------------------------------------------------------

Allowed methods: `GET`, `PUT`, `DELETE`

**TODO**: only give out private key data if client ask explicitly, otherwise 
stick to public part.

Cache Access
============

**TODO**: Peek at the cache, clear the cache, possibly dump it into a file?


Logging & Statistics
====================

URL: /servers/:server\_id/search-log?q=:search\_term
----------------------------------------------------

Allowed methods: `GET` (Query)

#### GET (Query)

Query the log, filtered by `:search_term`. Response body:

    {
      "q": "<search_term>",
      "log": [
        "<log_line>",
        ...
      ]
    }


URL: /servers/:server\_id/statistics
------------------------------------

* Top-X domains?
* Auth: ?
* Recursor: ?


URL: /servers/:server\_id/trace
-------------------------------

#### PUT (Configure)

Configure query tracing.

Client body:

    {
      "domains": "<regex_string>"
    }

Set `domains` to `null` to turn off tracing.

#### GET (Query)

Retrieve query tracing log and current config. Response body:

    {
      "domains": "<Regex>",
      log: [
        "<log_line>",
        ...
      ]
    }


URL: /servers/:server\_id/failures
----------------------------------

#### PUT

Configure query failure logging.

Client body:

    {
      "top-domains": <int>,
      "domains": "<Regex>",
    }

##### Parameters:

`top-domains` are the number of top resolved domains that are
automatically monitored for failures.

`domains` is a Regex of domains that are additionally monitored for
resolve failures.


#### GET

Retrieve query failure logging and current config.

Response body:

    {
      "top-domains": <int>,
      "domains": "<Regex>",
      "log": [
        {
          "first_occurred": <timestamp>,
          "domain": "<full domain>",
          "qtype": "<qtype>",
          "failure": <failure_code>,
          "failed_parent": "<full parent domain>",
          "details": "<log message>",
          "queried_servers": [
             {
               "name": <name>,
               "address": <address>
             }, ...
          ]
        },
        ...
      ]
    }

##### Parameters:

`failed_parent` is generally OPTIONAL.

Where `<failure_code>` is one of these:

  + `dnssec-validation-failed`

    DNSSEC Validation failed for this domain.

  + `dnssec-parent-validation-failed`

    DNSSEC Validation failed for one of the parent domains. Response
    MUST contain failed\_parent.

  + `nxdomain`

    This domain was not present on the authoritative nameservers.

  + `nodata`
  + `all-servers-unreachable`

    All auth nameservers that have been tried did not respond.

  + `parent-unresolvable`

    Response MUST contain `failed_parent`.

  + `refused`

    All auth nameservers that have been tried responded with refused.

  + `servfail`

    All auth nameservers that have been tried responded with servfail.

  + **TODO**: further failures

Data Overrides
==============

override\_type
--------------

`created` is filled by the Server.


    {
      "type": "Override",
      "id": <int>,
      "override": "ignore-dnssec",
      "domain": "nl",
      "until": <timestamp>,
      "created": <timestamp>
    }


    {
      "type": "Override",
      "id": <int>,
      "override": "replace",
      "domain": "www.cnn.com",
      "rrtype": "AAAA",
      "values": ["1.1.1.1", "2.2.2.2"],
      "until": <timestamp>,
      "created": <timestamp>
    }

**TODO**: what about validation here?

    {
      "type": "Override",
      "id": <int>,
      "override": "purge",
      "domain": "example.net",
      "created": <timestamp>
    }

Clears recursively all cached data ("plain" DNS + DNSSEC)

**TODO**: should this be stored? (for history)

URL: /servers/:server\_id/overrides
----------------------------------

Collection access.

Allowed Methods: `GET`, `POST`

URL: /servers/:server\_id/overrides/:override\_id
-------------------------------------------------

Allowed methods: `GET`, `PUT`, `DELETE`
