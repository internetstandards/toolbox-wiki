# Under constrution!!
This document lists the basic usage of commonly used DNS records. It can be used to track commonly made mistakes.

# A
* Points to an IPv4 address.
* Does not point to anyhting else.
* Record does not start (left side) with _ or -.

# AAAA
* Points to an IPv6 address.
* Does not point to anyhting else.
* Record does not start (left side) with _ or -.

# MX
* Points to an A and/or AAAA record.
* Preferrably does not point to other record types, but the use of CNAME records is seen in practice. RFC's are inconsistent on this.
* Has a priority value. 

# CNAME
* Points to other records (A, AAAA, CNAME).
* Be carefull with CNAME chaining; don't use too many CNAMEs in a row.
* The end of a CNAME chain is always an A an/or AAAA record.
* **Can only be combined with NS / SOA records if left side is equal.**

# SRV
* Records starts (left side) with _.
* Points to an A and/or AAAA record. 
* Has a priority value. 

# NS
* Points to an A and/or AAAA record.
* **Used to point to subzones.** 
* **Used to indicate what is inside the parent zone.** 
* **Each zone needs this to indicate what is inside the parent zone as a reference to this zone.**

# SOA
* Mandatory for every DNS zone.
* Contains the following information (seperated by a single white space):
   * FQDN of the primairy name server followed by a trailing dot.
   * e-mail address of the DNS administrator (followed by a trailing dot, the @ replaced with a dot).
   * an opening round bracket "(".
   * serial number that is changed (increased) on every zone change.
   * refresh time (in seconds) for a secondary name server to check the primairy name server for changes in the zone.
   * retry time (in seconds) for a secondary name server for requesting the serial number when the primary name server did not respond on the previous request. 
   * expire time (in seconds) after which a secondary name server should stop answering requests if the primary name server is not responding. 
   * Negative caching time to live (in seconds) which is the time a caching name server should cache a negative result (indicating that a name does not exist) coming from the authorative name server before trying again.
   * a closing round bracket ")".

