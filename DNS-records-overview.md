This document lists the basic usage of commonly used DNS records. It can be used to track commonly made mistakes.

# A
* Points to an IPv4 address.
* Does not point to anything else.
* Record does not start (left side) with _ or -.

# AAAA
* Points to an IPv6 address.
* Does not point to anything else.
* Record does not start (left side) with _ or -.

# MX
* Used to specify a mailserver.
* Contains a hostname, which in turn points to one or multiple A and/or AAAA records.
* Preferably does not point to other record types, but the use of CNAME records is seen in practice. RFC's are inconsistent on this.
* Has a priority value.

# CNAME
* Redirects to other DNS name with its own records (A, AAAA, CNAME).
* Be careful with CNAME chaining; don't use too many CNAMEs in a row.

# CAA
* Used to allowlist Certification Authorities that may issue certificates.
* May contain several policies, like issue, issuewild (wildcard, i.e. certificates valid for all subdomains), issuemail (S/MIME certificates) and issuevmc (BIMI certificates).

# SRV
* Records starts (left side) with _.
* Points to an A and/or AAAA record. 
* Has a priority value.

# NS
* Used to specify a nameserver.
* Contains a hostname, which in turn points to one or multiple A and/or AAAA records.

# SOA
* Mandatory for every DNS zone.
* Contains the following information (separated by a single white space):
   * FQDN of the primary name server followed by a trailing dot.
   * e-mail address of the DNS administrator (followed by a trailing dot, the @ replaced with a dot and other dots escaped using a backslash).
   * an opening round bracket "(".
   * serial number that is changed (increased) on every zone change.
   * refresh time (in seconds) for a secondary name server to check the primary name server for changes in the zone.
   * retry time (in seconds) for a secondary name server for requesting the serial number when the primary name server did not respond on the previous request. 
   * expire time (in seconds) after which a secondary name server should stop answering requests if the primary name server is not responding. 
   * Negative caching time to live (in seconds) which is the time a caching name server should cache a negative result (indicating that a name does not exist) coming from the authorative name server before trying again.
   * a closing round bracket ")".

