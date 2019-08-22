# Table of contents

# Introduction
This how-to is created by the Dutch Internet Standards Platform (the organization behind [internet.nl](https://internet.nl)) and is meant to provide practical information and guidance on implementing DMARC.

# What is DMARC?
DMARC is short for **D**omain based **M**essage **A**uthentication, **R**eporting and **C**onformance and is described in [RFC 7489](https://tools.ietf.org/html/rfc7489). With DMARC the owner of a domain can, by means of a DNS record, publish a  policy that states how to handle e-mail (deliver, quarantine, reject) which is not properly authenticated using SPF and/or DKIM. 

At the same time DMARC also provides the means for receiving reports which allows a domain's administrator to detect whether their domainname is used for phishing or spam. 

 
# Why use DMARC?
Before DMARC, organizations already took several measures to determine the authenticity of an e-mail (like SPF and DKIM) to reduce the received amount of SPAM to a minimum. This is basically a good thing, but if these measures fail to choose whether or not an email is SPAM with a high level of certainty, the choice is redirected to the addressee (receiving party). This methodology is prone to abuse, since users are generally not equiped with the knowledge and/or means to classify incoming emails. 
 
DMARC addresses this problem and enables the owner of a domain to take explicit responsiblity with regard to the actions taken by the sending party when the validity of an incoming email cannot be determined.  

# Tips, tricks and notices for implementation
* Interoperabily issues: https://tools.ietf.org/html/rfc7960
* DMARC does not require both DKIM or SPF. 
* Parked domain: “DMARC p=reject”. Make sure to include rua and ruf addresses, since this allows monitoring of possible abuse attempts.
* RFC 7489 [states](https://tools.ietf.org/html/rfc7489#section-6.4) that the tags dmarc-version ("v=") and dmarc-request ("p=") should be on the first and second position of the DMARC record. The order of the other tags does not matter: "components other than dmarc-version and dmarc-request may appear in any order".
* [Errata 5440 of RFC 7489](https://www.rfc-editor.org/errata_search.php?rfc=7489) states that a semicolon should be included in the DMARC version tag. Correct: "v=DMARC1;". Incorrect: "v=DMARC1". 

# Creating a DMARC record
The DMARC policy is published by means of DNS TXT record.
Overview 

rua: aggregate reports
ruf: forensic reports

| DMARC configuration tag | Required? | Value(s) | Explanation |
| ---  | --- |  --- | --- |
| v | mandatory | DMARC1; | |
| p | mandatory | none<br>quaritine<br>reject | |
| rua | optional | rua@example.nl | This field contains an e-mail address |
| ruf | optional |ruf@example.nl | This field contains an e-mail address |
| fo | mandatory | 0<br>1<br>s<br>d | reporting oprions for |
| adkim | optional | s<br>r | |
| aspf | optional | s<br>r | |
| pct | optional | | |
| rf | optional | | |
| ri | optional | | |
| sp | optional | | |
