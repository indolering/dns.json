Modular
=======

A modular standard with a 'baseline' format for data interchange and an 'extended' format for use by applications.

Introduction
------------

Trying to create a canonical JSON representation for a thirty year old plaintext "standard" is fraught with 
 implementation hazards.  Application and vendor-specific representations abound and RFC 1035 itself has alternate 
 representations of different data types.  Attempting to accommodate every use-case would result in many incomplete 
 implementations.
 
Instead, this format defines a lowest-common-denominator "Baseline" and optional "Extended" format.  The baseline format
 is intended for use at the boundaries of an application with a single set of fields and value types for all data 
 structures.  The extended format adds additional representations of fields to accommodate application specific use 
 cases while still mapping to a single canonical "Baseline" record.
 
For example, the Baseline representation of all DNS records are represented using a `(name, type, rdata, ttl)` tuple: 

    {
      "name": "_sip._tcp.example.com.",
      "type": "SRV",
      "rdata": "0 5 5060 sipserver.example.com.",
      "ttl": 600,
    }
 
The extended format supports distinct `service`, `proto`, `prio`, `weight`, and `port` fields:

    {
      "name": "_sip._tcp.example.com.",  //kept in sync with service and proto.
      "[service]": "sip",
      "[proto]": "tcp",
      "type": "SRV",
      "rdata": "0 5 5060 sipserver.example.com.", //kept in sync with prio, weight, port
      "[prio|priority|pref|preference]": 0, 
      "[weight]": 5,
      "[port]": 5060,
      "[target]": "sipserver.example.com.",
      "ttl": 600,
    }

It should be trivial for most vendors to convert their existing formats to the extended API.  From there, standard
 parsers can automatically convert to the baseline format and provide easy facilities for converting to other formats.
 
The author is willing to maintain a polygot Haxe library to simplify the process of converting existing APIs.
 
Baseline
========
Intended for use at the application boundary, it follows JSON conventions and is optimized for readability and
 serialization/deserialization.  While some progressive enhancements have been made, most changes are based on the
 plurality of choices found in the REST survey.  

The baseline format assumes a UTF-8 file encoded messages and requires Punycode representations of `name` and `rdata`
 values.

Record
-------

    {
      "name": "example.com.",
      "type": "AAAA", //always represented as an uppercase string
      "rdata": "2001:db8:85a3::8a2e:370:7334",
      "ttl": int|null,
    }
    
Lower casing the TTL field is a deliberate choice, as all but two of the REST APIs used uppercase.  In plaintext zone
 files, a default TTL is represented by omission, however, service providers tend to overload 0 or 1.  Neither of these
 approaches are satisfactory and a `null` sentinel value *must* be used to indicate the use of a default TTL.

In the baseline format, TXT records with long or multiple `rdata` values *must* be concatenated into a single string,
 without any delimiters to mark the boundaries between records:

    {
      "name": "example.com.",
      "type": "TXT",
      "rdata": "Lorem ipsum dolor sit amet, consectetur adipiscing elit ...",
      "ttl": 600,
    }

While there is no official limit on the length of `rdata` fields, in practice they are limited by the 10-record 
 recursion maximum for a total of 2,550 bytes.  Implementations *should* emit a warning and *may* truncate the string. 

Message
-------

    {
      "[id]": int,
      "[error]": false|"NXDOMAIN", //Replaces `rcode` field
      "[td]": false,     //TD - Whether reponse was truncated.
      "[ra]": true,      //RA - Whether recursion was available.
      "[rd]": true,      //RD - Whether recursion is requested.
      "[ad]": false,     //AD - Whether all responses requested validated using DNSSEC
      "[cd]": false,     //CD - Whether the client asked to disable DNSSEC
      "[*]": null,       //etc
      "type": "query"|"response"  //More legible version of QR
      "question": [
        {"name":"string", "type":"AAAA", "[class]": "IN"}
      ]
      "answer": Array<records>,
      "additional": Array<records>,
      "authority": Array<records>
    }
    
All fields have been lower-cased, the `rcode` field has been replaced with keyword errors, `QR` replaced with `type`,
 the q-prefix in the question section dropped, and various values given defaults.  The latter two changes may be
 controversial, but they would be big wins for readability and ergonomics.

Extended
========
The "Extended" format is designed to accommodate more traditional representations of DNS data, JSON REST APIs, and 
 application specific representations.  It should be trivial to map vendor-specific representations to the extended 
 format.
 
Hex representations of any field can be indicated by appending `Hex` to the key.  Conversion to the baseline format will
 fallback to the Hex field and invalid Hex values will simply be passed as a string.
 
Record
------
The extended record format retains the core `(name, type, rdata, ttl)` tuple.  Types can be represented as a string or
an integer without restrictions on casing, however, a fully compliant parser **must** perform case insensitive checking.
 
     {
       "name": "example.com.",
       "type": int|string,
       "rdata": "2001:db8:85a3::8a2e:370:7334",
       "ttl": null,
     }

Type specific fields are used to accommodate application specific use-cases.  The field names are generally lowercase
 versions of the RFC equivalent with any delimiters stripped from their values.  `host` and `target` can be used
 as an alternative to overloading the `name` and `rdata` fields.  However, parsers **must** check for `name` and `rdata`
 fields before attempting to parse type-specific fields.  Thus implementations **must** remove the equivalent `name` and
 `rdata` fields or keep their values in sync:
 
    { //in sync
      "name": "_sip._tcp.example.com.",
      "[service]": "sip",
      "[proto]": "tcp",
      "type": "SRV",
      "rdata": "0 5 5060 sipserver.example.com.",
      "[prio]": 0, 
      "[weight]": 5,
      "[port]": 5060,
      "[target]": "sipserver.example.com.",
      "ttl": 600,
    }

    { //name kept in sync while rdata is removed
      "name": "_sip._tcp.example.com.",  //
      "[host]" : "example.com",
      "[service]": "sip",
      "[proto]": "tcp",
      "type": "SRV",
      "target": "sipserver.example.com.", //rdata removed
      "prio": 0, 
      "weight": 5,
      "port": 5060,
      "ttl": 600,
    }
    
Implementations using the `host` or `target` fields **should** use them across all record types.  For example, the
 URI of A, AAAA, MX, SRV, and URL records should all be available in the `target` field.  

The `MX` record's `preference` field is inconsistent with other record types and most real-world APIs use `prio`.
 As such, the extended format includes `prio`, `priority`, `pref` and `preference` but standard parsers *must* give 
 priority to `prio`.

Implementations **must** represent TXT `rdata` as a single string or an array: 

    {
      "name": "example.com.",
      "type": "TXT",
      "rdata": [ //each string is a maximum of 255 bytes
        "Lorem ipsum dolor sit amet, consectetur adipiscing elit ...",
        "Suspendisse sit amet dapibus ligula, id convallis dolor ...",
      ],
      "ttl": 600,
    }

As with the baseline format, an `rdata` string field **may** be limited to 2,550 ASCII characters.  Strings in the
 `rdata` array **may** be limited to than 255 ASCII characters and the array **may** be limited to strings.  Parsers 
 **should** warn when longer strings are encountered and **may** truncate them.

Message
-------
Most implementations can support the Extended message format by simply making their keys lowercase:

    {
      "[rcode]": int,
      "[error]": int|bool|null|string
      "[id]": int|string,
      "[qr]": int|null,          //QR - Whether message is question (0) or answer (1)
      "[td]": int|bool|null,     //TD - Whether reponse was truncated.
      "[ra]": int|bool|null,     //RA - Whether recursion was available.
      "[rd]": int|bool|null,     //RD - Whether recursion is requested.
      "[ad]": int|bool|null,     //AD - Whether all responses requested validated using DNSSEC
      "[cd]": int|bool|null,     //CD - Whether the client asked to disable DNSSEC
      "[*]": null,               //etc
      "[qname]": string,
      "[qtype]": string,
      "[qclass]": int|string,
      "[type]": "query"|"response"|null //Non-standard replacement for QR
      "[question]":
        [
          {"qname|name": "example.com.", "qtype|name": int|string, "[qclass|class]":"IN"        
        ]
      "answer": Array<records>,
      "additional": Array<records>,
      "authority": Array<records>
    }

 

[edns-max]: https://tools.ietf.org/html/rfc2671#section-4.5.5