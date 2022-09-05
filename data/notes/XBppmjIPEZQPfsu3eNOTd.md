
[X.690](https://en.wikipedia.org/wiki/X.690#BER_encoding) is an ITU-T standard specifying several ASN.1 encoding formats:

- Basic Encoding Rules (BER)
- Distinguished Encoding Rules (DER), derived from BER
- Canonical Encoding Rules (CER) (not concerned in this series articles)

## Basic Structure

Type–length–value (TLV) encodings:

Identifier octets | Length octets | Contents octets | End-of-contents octets
--                | --            | --              | --
Type	            | Length        | *optional* Value  | *optional*

### Identifier




2A 86  48 86 F7 0D 01 07 02