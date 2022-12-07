---
title: "OAuth 2.0 Direct Interaction Grant"
category: std

docname: draft-parecki-oauth-direct-interaction-grant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: OAuth Working Group
keyword:
 - oauth
venue:
  group: OAuth
  type: Working Group
  mail: oauth@ietf.org
  arch: https://oauth.net
  github: aaronpk/oauth-direct-interaction-grant
  latest: https://aaronpk.github.io/oauth-direct-interaction-grant/draft-parecki-oauth-direct-interaction-grant.html

author:
 -  fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  RFC6749:


informative:


--- abstract

This document extends the OAuth 2.0 Authorization Framework {{RFC6749}} with a
new grant type, the "Direct Interaction Grant", which can be used by applications
that want to control the user experience of the process of obtaining authorization
from the user.

In many cases, this can provide an entirely browserless experience suited for native
applications, only delegating to the browser in unexpected or error conditions.

While a fully-delegated approach using the Authorization Code Grant is generally
preferred, this draft provides a mechanism for the client to directly interact
with the user. This requires a high degree of trust between the authorization server
and the client, as well as should only be considered when there are usability
concerns with a browser-based approach, such as for native mobile or desktop applications.


--- middle

# Introduction

TODO Introduction

## Usage and Applicability

TODO


# Conventions and Definitions

{::boilerplate bcp14-tagged}






# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
