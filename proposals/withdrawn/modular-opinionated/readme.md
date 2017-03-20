A fork of the modular proposal that is highly opinionated and more aggressive in it's transforms and dynamic structure. 


 
Introduction
============
Trying to create a canonical JSON representation for a thirty year old standard is fraught with implementation hazards.
Application and vendor-specific representations abound and RFC 1035 itself has alternate representations of different
 data types.  Attempting to accommodate every use-case would result in many incomplete implementations.
 
This proposal attempts to define a lowest-common-denominator "Baseline" format optimized for clarity and
 serialization/deserialization and an "Extended" format designed to accommodate existing application-specific
 representations.  Although the extended format can represent the same data in several ways, they can all be exported to
 a single canonical baseline representation.
 
Although the data structures and values may differ, the representation of field names are identical in both formats.

Baseline
========

Records
-------

    {
      "name": "example.com.",
      "type": "AAAA",
      "rdata": "2001:db8:85a3::8a2e:370:7334",
      "[ttl]": int|null,
    }

Lowercasing the TTL field is a deliberate choice, as all but two of the REST APIs used uppercase values. A `null` or
 missing TTL value indicates usage of default TTL value.  Outside of zones and vendor-specific values, the default TTL
 is 900 seconds (15 minutes) - the "magic" value for network cache hit rates according to various academic studies. 
 
To provide consistent access to URI values across A, AAAA, MX, SRV, NAPTR other record types, the extended version
 extracts out the URI value into a `target` field.

    {
      "name": "example.com.",
      "type": "AAAA",
      "rdata": "2001:db8:85a3::8a2e:370:7334",
      "[target]": "2001:db8:85a3::8a2e:370:7334"
    }

Application specific data is encoded in the `name` field via standard underscore prefixes and delimited using spaces in
 `rdata`.  The extended version can break out type-specific information in lower cased field names.
    
    {
      "name": "_sip._tcp.example.com.",
      "[service]": "sip",
      "[proto]": "tcp",
      "type": "SRV",
      "rdata": "0 5 5060 sipserver.example.com.",
      "[prio]": 0, 
      "[weight]": 5,
      "[port]": 5060,
      "[target]": "sipserver.example.com."
    }
    
The `preference` value of MX records is broken out as `prio` in the extended format.  The API survey showed this to be 
a popular choice and it simplifies modeling of SRV and MX record types.
 
    {
      "name": "_sip._tcp.example.com.",
      "type": "SRV",
      "rdata": "0 5 5060 sipserver.example.com.",
      "[prio]": 0, 
      "[weight]": 5,
      "[port]": 5060,
      "[target]": "sipserver.example.com." //optional
    }

`rdata` spanning multiple TXT records are concatenated into a single string.  An optional `strings` array is available 
in the extended version of `TXT` records.  In practice, the length of RDATA values are limited by the 10-record 
recursion maximum, which is the maximum length of the `strings` array and 2,550 byte limit of the `rdata` value.

Query
-----
Queries are distinct from messages and are designed to be efficiently implemented as a query string.

    {
      "[duid]": "string",
      "[recursion]": true,        //RD - Recursion desired.
      "[dnssec]": true,           //!CD - DNSSEC validation desiered.
      "name": "example.com.",
      "type": "AAAA",
      "[class]": "IN",
      "[padding]": random string
    }

Answers
-------
    
    {
      "[error]": null|"NXDOMAIN", //Status field, 0 for no error, Text based representation for others
      "[duid]: int|string,
      "[truncated]": false,     //TD
      "[recursion]": true,      //RA
      "[validated]": true,      //AD - Whether responses were validated using DNSSEC
      "name": "example.com.",
      "type": "AAAA",
      "[class]": "IN",
      "[answer]": Array<records>,
      "[additional]": Array<records>,
      "[authority]": Array<records>
    }
    
    