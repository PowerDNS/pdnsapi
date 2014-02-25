PowerDNS API
============

PowerDNS features a built-in API. For the Authoritative Server, starting with
version 3.4, for the Recursor starting with version 3.6.

At the time of writing this, these versions were not released, but preliminary
support is available in git.


Try it
------

Install PowerDNS Authoritative with one of the gsql backends (i.e. MySQL,
PostgreSQL or SQLite3).

Then configure as follows:

    experimental-json-interface=yes
    webserver=yes
    webserver-password=changeme


After restarting `pdns_server`, the following examples should start working:

    curl -v http://a:changeme@127.0.0.1:8081/servers/localhost/zones | jq .
    curl -v http://a:changeme@127.0.0.1:8081/servers/localhost/zones/example.org | jq .

`jq` is a highly recommended tool for pretty-printing JSON. If you don't have
`jq`, try `json_pp` or `python -mjson.tool` instead.


Try it (Recursor edition)
-------------------------

Install PowerDNS Recursor, configured as follows:

    experimental-json-interface=yes
    experimental-webserver=yes
    experimental-webserver-password=changeme
    auth-zones=
    forward-zones=
    forward-zones-recurse=


After restarting `pdns_recursor`, the following examples should start working:

    curl -v http://a:changeme@127.0.0.1:8082/servers/localhost | jq .
    curl -v http://a:changeme@127.0.0.1:8082/servers/localhost/zones | jq .


API Specification
-----------------

The complete API docs are available in `api_specs.md`.


Additional help
---------------

For additional help, come to the `#powerdns` IRC channel on `irc.oftc.net`.
