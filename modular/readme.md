* Name: Modular
* Homepage: (github.com/xyz)[]
Version: 0.1
* Slug: A conservative candidate that balances expressiveness and easy-of-use via baseline and extended formats.

Introduction
============
Trying to create a canonical JSON representation for a thirty year old standard is fraught with implementation hazards.
Application and vendor-specific representations abound and RFC 1035 itself has alternate representations of different
 data types.  Attempting to accommodate every use-case would result in many incomplete implementations.
 
This proposal attempts to define a lowest-common-denominator "Baseline" and optional "Extended" format.  Although the
 extended format can represent the same values using differ types and structures, the baseline format provides a single
 canonical representation.
 
To ensure interoperability, a Haxe-based polygot library can be used by servers or clients to translate between the 
 formats fairly trivially.
 
Baseline
========
Intented for use at the application boundary, the baseline format optimized for readability and
 serialization/deserialization.  While some progressive enhancements have been made, most changes are based on the
 plurality of choices found in the REST survey.  

The baseline format assumes a UTF-8 file encoded messages and requires Punycode representations of `name` and `rdata`
 values.

Records
-------

    {
      "name": "example.com.",
      "type": "AAAA", //always represented as a string
      "rdata": "2001:db8:85a3::8a2e:370:7334",
      "ttl": int|null,
    }
    
Lower-casing the TTL field is a deliberate choice, as all but two of the REST APIs used uppercase.  In plaintext zone
 files, default TTL is represented by omission.  However, service providers tend to overload 0 or 1.  Neither of these
 approaches are satisfactory and a `null` sentinel value *must* be used to indicate the use of a default TTL.

In the baseline format, TXT records with long or multiple `rdata` values *must* be concatenated into a single string,
 without any delimiters to mark the boundaries between records:

    {
      "name": "example.com.",
      "type": "TXT",
      "rdata": "Lorem ipsum dolor sit amet, consectetur adipiscing elit ...",
      "ttl": 600,
    }

In the extended format, `rdata` can be either a string or an array of 255 byte strings.

Message
-------

    {
      "[id]": int,
      "[error]": false|"NXDOMAIN", //Replaces `rcode` field
      "[qr]": false,
      "[td]": false,     //TD - Whether reponse was truncated.
      "[ra]": true,      //RA - Whether recursion was available.
      "[rd]": true,      //RD - Whether recursion is requested.
      "[ad]": false,     //AD - Whether all responses requested validated using DNSSEC
      "[cd]": false,     //CD - Whether the client asked to disable DNSSEC
      "[*]": null,       //etc
      "question": [
        {"qname":"string", "qtype":"AAAA", "[qclass]": "IN"}
      ]
      "[answer]": Array<records>,
      "[additional]": Array<records>,
      "[authority]": Array<records>
    }
    
All fields have been lower-cased, the rcode field has been replaced with keyword errors, the q-prefix in the question
 section dropped, and various values given defaults.

Extended
========
The "Extended" format is designed to accommodate more traditional representations of DNS data as well as common vendor
 and application specific representations used internally.  It should be trivial for most servers to convert their
 existing internal formats to a **subset** of the extended format. Full support of the extended format is usually not a
 desirable goal, as it is much simpler to export to the baseline format for data exchange.
 
Hex representations of any field can be indicated by appending `Hex` to the key.  Conversion to the baseline format will
 fallback to the Hex field and invalid Hex values will simply be passed as a string.
 
Records
-------
The goal of the extended record format is to allow very simple processing of any record type using the `(name, type, 
 rdata, ttl)` tuple.  However, the `type` field can be represented as either an integer or a string.  The type string
 **should** be uppercase but a fully compliant parser **must** perform case insensitive checking.
 
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
 The extended checks `rdata`, `prio`, and `priority` before `pref` and `preference`.

Implementations **must** represent TXT `rdata` as a single string without any delimiters or as an array in `strings`:

    {
      "name": "example.com.",
      "type": "TXT",
      "rdata": "Lorem ipsum dolor sit amet, consectetur adipiscing elit ...",
      "ttl": 600,
      "[strings]": [ //each string is a maximum of 255 bytes
        "Lorem ipsum dolor sit amet, consectetur adipiscing elit ...",
        "Suspendisse sit amet dapibus ligula, id convallis dolor ...",
      ]
    }

Message
-------
Most implementations can support the Message format by making their keys lowercase:

    {
      "[rcode]": int,
      "[error]": int|bool|null|string
      "[id]": int,
      "[qr]": int|bool,
      "[qname|name]": string,
      "[qtype|type]": string,
      "[qclass|class]": int|string
      "[question]":
        [
          {"qname|name": "example.com.", "qtype|name": int|string, "[qclass|class]":"IN"        
        ]
      "[answer]": Array<records>,
      "[additional]": Array<records>,
      "[authority]": Array<records>
    }
 
[edns-max]: https://tools.ietf.org/html/rfc2671#section-4.5.5