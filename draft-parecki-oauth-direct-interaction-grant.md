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
  - fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
  - fullname: Pieter Kasselman
    organization: Microsoft
    email: pieter.kasselman@microsoft.com

normative:
  RFC6749:
  RFC6755:
  RFC7636:
  RFC8259:

informative:
  RFC8628:


--- abstract

This document extends the OAuth 2.0 Authorization Framework {{RFC6749}} with
new grant types to support multi-factor authentication as well as alternatives to password authentication.
These can be used by applications that want to control the user experience of the process of obtaining authorization from the user.

In many cases, this can provide an entirely browserless experience suited for native
applications, delegating to the browser in unexpected, high risk, or error conditions.

While a fully-delegated approach using the Authorization Code Grant is generally
preferred, this draft provides a mechanism for the client to directly interact
with the user. This requires a high degree of trust between the authorization server
and the client. It should only be considered when there are usability
concerns with a redirect-based approach, such as for native mobile or desktop applications.


--- middle

# Introduction

TODO: Key points to address include problem description, the relationship to the step-up authentication spec (use of acr etc.), properties of the protocol (extensibility etc).

## Usage and Applicability

TODO: Mention the trust prerequisites for this to be useful. Absolutely not allowed for third-party apps. Designed for native apps, specifically first-party, when the AS and app are operated by the same entity, and the user understands them both as the same entity...




# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the terms "Access Token", "Authorization Code",
"Authorization Endpoint", "Authorization Server" (AS), "Client", "Client Authentication",
"Client Identifier", "Client Secret", "Grant Type", "Protected Resource",
"Redirection URI", "Refresh Token", "Resource Owner", "Resource Server" (RS)
and "Token Endpoint" defined by {{RFC6749}}.

TODO: Replace RFC6749 references with OAuth 2.1


# Protocol Overview


1. The client prompts the user and collects their user identifier (e.g. email address)
2. The client sends the user identifier to the AS, along with any hint it may have received about the authentication level required
3. The AS replies with the authentication mechanism required based on the user ID provided (should these be amr values?)
4. The client requests an authentication challenge from the AS (this is optional - only if the authentication method requires it)
5. The AS delivers a challenge to the user (e.g. one-time code via email, SMS, or a push notification in an app) (this is optional - only if it is a challenge response type authentiation method).
6. The client collects the necessary authentication details from the user and sends them back to the AS
7. The AS decides if additional requirements need to be met, repeating steps 3 through 6 as needed (the AS knows this due to the acr_value sent by the client initially, or from its own requirements)
8. The AS replies with the token response



## Authorization Grants

### One-Time Passwords (OTP)

The one-time password (OTP) credentials can be used as an
authorization grant to obtain an access token.  An OTP is generated
by a hardware device or a software application installed on a device
such as a mobile phone.  These devices have an embedded secret which
used as a seed for generating OTPs.  An OTP is single-use and proves
control of the device.

When used in addition to another grant type (such as resource owner
password credentials), one-time password credentials add an
additional factor of authentication.

Even though this grant type requires direct client access to the
resource owner credentials, the resource owner credentials are used
for a single request and are exchanged for an access token.
Furthermore, the single-use nature of an OTP limits the impact of
exposing a long-term password alone to the client when the resulting
access token or refresh token are also short-lived.

OTP methods include:

* Code generators such as Google Authenticator

### Out-of-Band Authorization (OOB)

The out-of-band (OOB) authorization grant allows authorization to be
obtained via a secondary channel.  Once authorized, the grant can be
used to obtain an access token.  Use of a secondary channel supports
several use cases.

The out-of-band authorization grant can be used with an out-of-band
authenticator to add an additional factor of authentication.  An out-
of-band authenticator is a physical device that is uniquely
addressable and can communicate with the authorization server over a
secondary channel.  Common examples would be a mobile phone using SMS
as a secondary channel, or a software application installed on a
device using push notifications as a secondary channel.

The out-of-band authorization grant can also facilitate more complex
authorization processes.  For example, out-of-band authorization can
be used to perform multi-party authorization, in which two or more
resource owners are needed in order to grant access.

OOB methods include:

* One-time codes sent via email and SMS
* Display a code on the primary device, enter it on the secondary device (e.g. OAuth Device Flow)
* Acknowledge a push notification on a secondary device
* Compare the codes displayed on two devices


### Recovery Code

The recovery code is numeric or character string from a set of
secrets shared between the resource owner and the authorization
server.  These secrets are typically used by the resource owner in
the event another authenticator is lost or malfunctions.


# Protocol Endpoints

* Authorization initiation endpoint
  * A client initiates a login flow
  * With or without information collected from the user (e.g. password)
  * May contain a `device_session`
  * Returns an `mfa_token`
* Authorization challenge endpoint
  * A client requests an authorization challenge (e.g. send a user an SMS code), which it can later use as an authorization grant
* Token endpoint
  * This specification defines new grant types and new error responses
  * Adds `device_session` in token response
  * Note: Passwords are never sent to the token endpoint

Not every authorization grant type utilizes all endpoints.
Extension grant types MAY define additional endpoints as needed.

The token endpoint is used by the client to obtain an access token by
presenting its authorization grant or refresh token, as described in
Section 3.2 of OAuth 2.0 {{RFC6749}}.

This specification adds additional grant types used at the token endpoint,
as well as extends the token endpoint response to allow the authorization
server to indicate that further authentication of the user is required.

Extension Grant Types

* grant_type=urn:ietf:params:oauth:grant-type:mfa-otp
* grant_type=urn:ietf:params:oauth:grant-type:mfa-oob
* grant_type=urn:ietf:params:oauth:grant-type:mfa-recovery-code
* grant_type=urn:ietf:params:oauth:grant-type:otp
* grant_type=urn:ietf:params:oauth:grant-type:oob




# Authorization Initiation Request {#authorization-initiation}

A client may wish to initiate an authorization flow by first prompting the user for their user identifier or other account information. The authorization initiation endpoint is a new endpoint to collect this login hint and direct the client with the next steps, whether that is to do an MFA flow, or perform an OAuth redirect-based flow.

The authorization initiation endpoint is an optional feature that can be used to drive passwordless flows.


## Authorization Request

The client makes a request to the authorization initiation endpoint by adding the
following parameters using the "application/x-www-form-urlencoded"
format with a character encoding of UTF-8 in the HTTP request body:

"login_hint":
: OPTIONAL. If the client has collected the user's username, email,
  phone number or other identifier, it can provide this in the request.

"password":
: OPTIONAL. If the client has collected the user's password, it can provide
  it at this stage.

"scope":
: OPTIONAL. The OAuth scope defined in {{RFC6749}}.

"acr_values":
: OPTIONAL. The acr_values requested by the client.

"device_session":
: OPTIONAL. If the client has previously obtained a device session, described in {{device-session}}.

For example:

    POST /initiate HTTP/1.1
    Host: server.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded

    login_hint=%2B1%20%28310%29%20123-4567&scope=profile


## Authorization Response

The authorization server responds with an MFA token
defined in {{mfa-token-response}}, or an authorization challenge response
defined in {{authorization-challenge-response}}.

For example, the authorization server can reply with an MFA token,
as described {{mfa-token-response}}:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "mfa_token": "uY29tL2F1dGhlbnRpY"
    }

Or, if the authorization server requires that this particular user
go through a typical redirect flow, can respond with the redirect challenge,
as described in {{redirect-challenge}}:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "challenge_type": "redirect"
    }



# Authorization Challenge Response {#mfa-token-response}

Upon any request to the token endpoint, the authorization server can
respond with an authorization challenge instead of a successful access token response.

An authorization challenge error response is a particular type of
error response as defined in Section 5.2 of OAuth 2.0 {{RFC6749}} where
the error code is set to the following value:

"mfa_required":
: The authorization grant is insufficiently authorized, but another
  access token request MAY succeed if an additional authorization
  grant is presented.

In addition to the response parameters defined in Section 5.2 of
OAuth 2.0 {{RFC6749}}, the following parameters MUST be included in the
response when the error code is set to `mfa_required`:

"mfa_token":
: MFA token value associated with the ongoing authorization session.

For example:

    HTTP/1.1 403 Forbidden
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "error": "mfa_required",
      "mfa_token": "uY29tL2F1dGhlbnRpY"
    }


# Authorization Challenge Endpoint

The authorization challenge endpoint is used by the client to obtain
an authorization challenge by presenting an MFA token.

Use of the authorization challenge endpoint is OPTIONAL; if a client
knows of a suitable authenticator through an out-of-band mechanism,
it can obtain a strong authorization grant directly.


## Authorization Challenge Request {#authorization-challenge-request}

The client makes a request to the authorization challenge endpoint by
adding the following parameters using the "application/x-www-form-
urlencoded" format with a character encoding of UTF-8 in the HTTP
request body:

"mfa_token":
: REQUIRED.  The MFA token received from the authorization server in
  the authorization challenge error response.

"challenge_type":
: OPTIONAL.  List of authorization challenge type strings that the
  client supports, expressed as a list of space-delimited, case-
  insensitive strings.

"authenticator_id":
: OPTIONAL.  The identifier of the authenticator to challenge.  The
  authorization server MUST ensure that the authenticator is
  associated with the resource owner.

"device_session":
: OPTIONAL. The device session, described in {{device-session}}.

"client_id":
: REQUIRED, if the client is not authenticating with the
  authorization server as described in Section 3.2.1 of OAuth 2.0
  {{RFC6749}}.


For example, the client makes the following HTTP request:

    POST /challenge HTTP/1.1
    Host: server.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded

    mfa_token=uY29tL2F1dGhlbnRpY&challenge_type=otp%20oob

The authorization server MUST:

* require client authentication for confidential clients or for any
  client that was issued client credentials (or with other
  authentication requirements),

* authenticate the client if client authentication is included,

* ensure that the MFA token was issued to the authenticated
  confidential client, or if the client is public, ensure that the
  token was issued to the `client_id` in the request,

* verify that the MFA token is valid, and

* restore the authorization session based on the state referenced by
  or encoded into the MFA token.

## Authorization Challenge Response {#authorization-challenge-response}

If the authorization challenge request is valid and authorized, the
authorization server selects an authorization challenge, the response
to which would satisfy the authorization session, and constructs the
response by adding the following parameters to the entity-body of the
HTTP response using the `application/json` format {{RFC8259}} with a
200 (OK) status code:

"challenge_type":
: REQUIRED.  The type of the authorization challenge issued.  Value
  is case insensitive.

All additional parameters are specified by the authorization
challenge type. This document defines the `otp` type, the `oob` type,
the `recovery-code`, and the `redirect` type.

For example:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "challenge_type": "otp"
    }

If the authorization challenge request failed, the authorization
server responds with an error response as described in Section 5.2 of
OAuth 2.0 {{RFC6749}}.

In addition to the error codes defined in Section 5.2 of OAuth 2.0
{{RFC6749}}, the following error codes are specified for use in
authorization challenge endpoint responses:

"invalid_authenticator":
: The requested authenticator does not exist or is not associated
  with the resource owner.

"expired_token":
: The provided MFA token is invalid, expired, revoked, or was issued
  to another client.  The client MAY initiate a new authorization
  session.

"unsupported_challenge_type":
: The challenge types supported by the client are not supported by
  the authorization server or not available to the resource owner.

"association_required":
: The resource owner is not associated with any authenticator.  The
  authorization session MAY be continued by completing authenticator
  association.

"server_error":
: The authorization server encountered an unexpected condition that
  prevented it from fulfilling the request.

"bad_gateway":
: The authorization server received an invalid response from an
  upstream server it accessed in attempting to fulfull the request.
  This typically occurs when challenging an OOB authenticator and
  the gateway is down, for example SMS.


### OTP Challenge

If the authorization server requires an OTP credential as an
additional authorization grant, it responds with an OTP authorization
challenge type containing the following parameters:

"challenge_type":
: REQUIRED.  Value MUST be set to `otp`.

No additional parameters are specified for the OTP authorization
challenge type.

For example:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
        "challenge_type": "otp"
    }


### OOB Challenge {#oob-authorization-challenge}

If the authorization server requires out-of-band authorization as an
additional authorization grant, it generates a unique out-of-band
transaction code that is valid for a limited time.  It then initiates
an out-of-band authorization operation, the details of which are out
of scope for this specification.  It then responds with an OOB
authorization challenge type containing the following parameters:

"challenge_type":
: REQUIRED.  Value MUST be set to "oob".

"oob_code":
: REQUIRED.  The out-of-band transaction code.  The out-of-band
  transaction code MUST expire shortly after it is issued to
  mitigate the risk of leaks.  A maximum out-of-band transaction
  code lifetime of 10 minutes is RECOMMENDED.
  TODO: Determine whether this is required or what benefit it provides if any.

"binding_code":
: OPTIONAL.  The end-user verification code used to bind the
  authorization operation on the secondary channel with the primary
  channel.  REQUIRED, if the value of "binding_method" is set to
  "transfer" or "compare".

"binding_method":
: OPTIONAL.  The method used to bind the authorization operation on
  the secondary channel with the primary channel.  If no value is
  provided, clients MUST use "none" as the default.  Values defined
  by this specification are:

     "prompt":
     :  The end user should be prompted to enter a code received during
        out-of-band authorization via the secondary channel into the
        client.  For example, the end user receives a code on their
        mobile phone (typically a 6-digit code) and types it into the
        client.

     "transfer":
     :  The client displays or otherwise communicates the
        "binding_code" to the end user and instructs them to enter it
        into or otherwise transfer it to the secondary channel.  For
        example, the end user may view the "binding_code" on the client
        and either type it into an app on their mobile phone or use a
        QR code to effect the transfer.

     "compare":
     :  The client displays the "binding_code" to the end user and
        instructs them to compare it to the code received during out-
        of-band authorization before confirming authorization via the
        secondary channel.

     "none":
     :  No binding is performed between the client on the primary
        channel and the out-of-band authorization operation via the
        secondary channel.

"expires_in":
: OPTIONAL.  The lifetime in seconds of the `oob_code`.

"interval":
: OPTIONAL.  The minimum amount of time in seconds that the client
  SHOULD wait between polling requests to the token endpoint.  If no
  value is provided, clients MUST use 5 as the default.

For example:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
       "challenge_type": "oob",
       "oob_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
       "binding_method": "prompt",
       "expires_in": 300,
       "interval": 5
    }


### Recovery Code Challenge

If the authorization server requires a recovery code as an
authorization grant, it responds with a recovery code authorization
challenge containing the following parameters:

"challenge_type":
: REQUIRED.  Value MUST be set to `recovery-code`.

No additional parameters are specified for the recovery code
authorization challenge type.

For example:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "challenge_type": "recovery-code"
    }

While this is likely to be an uncommon challenge response,
the AS may decide a recovery code is required, for example if
it is known that the user has lost their other MFA options.


### Redirect Challenge {#redirect-challenge}

In the case where the authorization server wishes to interact with the user itself, limiting the client's interaction with the user, it can return the `redirect` challenge type.

"challenge_type":
: REQUIRED. Value MUST be set to `redirect`.

The client is expected to initiate a traditional OAuth Authorization Code flow with PKCE according to {{RFC6749}} and {{RFC7636}}.

For example:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store

    {
      "challenge_type": "redirect"
    }

TODO: Should there be some connection to the authorization code flow that the client initiates next?

This can be used to enable primary or secondary authentication with social providers or third party IdPs which require a browser redirect flow.



## User Interaction

### OTP Challenge Interaction

After receiving an OTP challenge, the client prompts or otherwise
interacts with the resource owner to obtain an OTP generated by a
device in the possession of the resource owner.

### OOB Challenge Interaction

After receiving an out-of-band challenge, the client prompts or
otherwise interacts with the resource owner to inform them of the
ongoing authorization operation.  Any necessary actions by the
resource owner or other party on the secondary channel are out of
scope of this specification.

For example, the client informs the user to expect to receive a
verification code via SMS, and provides a form to enter the code.

### Recovery Code Challenge Interaction

After receiving a recovery code challenge, the client prompts or
otherwise interacts with the resource owner to obtain a recovery
code.  Such codes are typically already in the resource owner's
posession, having been previously distributed to the resource owner.

Recovery codes are typically necessary upon the resource owner
realizing that other authenticators have been lost or are
malfunctioning (for instance, when attempting to satisfy a non-
recovery code authorization challenge).  In such an event, the client
SHOULD provide the resource owner a means of directly entering a
recovery flow.

### Redirect Challenge Interaction

After receiving a redirect challenge, the client initiates an
OAuth authorization code flow with the authorization server. The
tokens are obtained with the traditional authorization code grant.


# Token Request Grant Types

This specification defines new grant types at the token endpoint,
providing a way for a client to obtain an access token given any of
the MFA challenges previously described.

## MFA OTP Grant

The client makes a request to the token endpoint by adding the
following parameters using the "application/x-www-form-urlencoded"
format with a character encoding of UTF-8 in the HTTP request body:

"grant_type":
: REQUIRED. `urn:ietf:params:oauth:grant-type:mfa-otp`

"otp":
: REQUIRED. The one-time password generated by a device. e.g. 123456

"mfa_token":
: REQUIRED. The MFA token, "mfa_token" from the prior authorization
  challenge error response

"client_id":
: REQUIRED if the client is not authenticating with the
  authorization server.


For example:

    POST /token HTTP/1.1
    Host: server.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:mfa-otp
    &otp=123456&mfa_token=uY29tL2F1dGhlbnRpY

The authorization server MUST:

   *  require client authentication for confidential clients or for any
      client that was issued client credentials (or with other
      authentication requirements),

   *  authenticate the client if client authentication is included,

   *  ensure that the MFA token was issued to the authenticated
      confidential client, or if the client is public, ensure that the
      token was issued to `client_id` in the request,

   *  verify that the MFA token is valid,

   *  restore the authorization session based on the state referenced by
      or encoded into the MFA token, and

   *  validate the one-time password credentials using its existing OTP
      validation algorithm.


## MFA OOB Grant

The client makes a request to the token endpoint by adding the
following parameters using the "application/x-www-form-urlencoded"
format with a character encoding of UTF-8 in the HTTP request body:

"grant_type":
: REQUIRED. `urn:ietf:params:oauth:grant-type:mfa-oob`

"oob_code":
: REQUIRED.  The out-of-band transaction code, `oob_code` from the
  authorization challenge, defined in {{oob-authorization-challenge}}

"binding_code":
: REQUIRED, if the binding method of the authorization challenge
 is set to "prompt".

"mfa_token":
: REQUIRED.  The MFA token, "mfa_token" from the prior authorization
  challenge error response, defined in {{mfa-token-response}}.

"client_id":
: REQUIRED, if the client is not authenticating with the
  authorization server as described in Section 3.2.1 of OAuth 2.0
  {{RFC6749}}.

If the client type is confidential or the client was issued client
credentials (or assigned other authentication requirements), the
client MUST authenticate with the authorization server as described
in Section 3.2.1 of OAuth 2.0 {{RFC6749}}.

For example, the client makes the following HTTP request using
transport-layer security (with extra line breaks for display purposes only):

    POST /token HTTP/1.1
    Host: server.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:mfa-oob
    &oob_code=GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS
    &binding_code=123456&mfa_token=uY29tL2F1dGhlbnRpY

The authorization server MUST:

* require client authentication for confidential clients or for any
  client that was issued client credentials (or with other
  authentication requirements),

* authenticate the client if client authentication is included,

* ensure that the MFA token was issued to the authenticated
  confidential client, or if the client is public, ensure that the
  token was issued to `client_id` in the request,

* verify that the MFA token is valid,

* restore the authorization session based on the state referenced by
  or encoded into the MFA token,

* validate the out-of-band transaction code using its existing OOB
  validation algorithm, and

* if required, validate the binding code using its existing OOB
  validation algorithm.


### Token Error Response

In addition to the error codes defined in Section 5.2 of OAuth 2.0
{{RFC6749}}, the following error codes are specific for the out-of-band
authorization grant in token endpoint responses:

"authorization_pending":
: The authorization request is still pending as the out-of-band
  authorization operation has not yet completed.  The client SHOULD
  repeat the access token request to the token endpoint (a process
  known as polling).  Before each new request, the client MUST wait
  at least the number of seconds specified by the `interval`
  parameter of the authorization challenge (see Section X), or 5
  seconds if none was provided, and respect any increase in the
  polling interval required by the `slow_down` error.

"slow_down":
: A variant of "authorization_pending", the authorization request is
  still pending and polling should continue, but the interval MUST
  be increased by 5 seconds for this and all subsequent requests.

"access_denied":
: The authorization request was denied.

"expired_token":
: The "oob_code" or "mfa_token" has expired, and the authorization
  session has concluded.  The client MAY commence a new
  authorization session but SHOULD wait for user interaction before
  restarting to avoid unnecessary polling.

The `authorization_pending` and `slow_down` error codes define
particularly unique behavior, as they indicate that the OAuth client
should continue to poll the token endpoint by repeating the token
request (implementing the precise behavior defined above).  If the
client receives an error response with any other error code, it MUST
stop polling and SHOULD react accordingly, for example, by displaying
an error to the user.

On encountering a connection timeout, clients MUST unilaterally
reduce their polling frequency before retrying.  The use of an
exponential backoff algorithm to achieve this, such as doubling the
polling interval on each such connection timeout, is RECOMMENDED.

The error codes and client behavior specified in this section are
intentionally identical to those defined by OAuth 2.0 Device
Authorization Grant {{RFC8628}}.


## Recovery Code Grant

The client makes a request to the token endpoint by adding the
following parameters using the "application/x-www-form-urlencoded"
format with a character encoding of UTF-8 in the HTTP request body:

"grant_type":
: REQUIRED.  `urn:ietf:params:oauth:grant-type:mfa-recovery-code`

"recovery_code":
: REQUIRED.  The recovery code.

"mfa_token":
: REQUIRED.  The MFA token, "mfa_token" from the prior authorization
  challenge error response, defined in {{mfa-token-response}}.

"client_id":
: REQUIRED, if the client is not authenticating with the
  authorization server as described in Section 3.2.1 of OAuth 2.0
  {{RFC6749}}.

If the client type is confidential or the client was issued client
credentials (or assigned other authentication requirements), the
client MUST authenticate with the authorization server as described
in Section 3.2.1 of OAuth 2.0 [RFC6749].

For example, the client makes the following HTTP request using
transport-layer security (with extra line breaks for display purposes
only):

    POST /token HTTP/1.1
    Host: server.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:mfa-recovery-code
    &recovery_code=WDJB-MJHT&mfa_token=uY29tL2F1dGhlbnRpY

The authorization server MUST:

* require client authentication for confidential clients or for any
  client that was issued client credentials (or with other
  authentication requirements),

* authenticate the client if client authentication is included,

* ensure that the MFA token was issued to the authenticated
  confidential client, or if the client is public, ensure that the
  token was issued to `client_id` in the request,

* verify that the MFA token is valid,

* restore the authorization session based on the state referenced by
  or encoded into the MFA token, and

* validate the recovery code using its existing recovery code
  validation algorithm.


### Access Token Response

In addition to the parameters defined in Section 5.1 of OAuth 2.0
{{RFC6749}}, the following additional parameters are specified for the
recovery code grant:

"recovery_code":
: OPTIONAL.  A newly issued recovery code, in which case the client
  MUST discard the old recovery code and replace it with the new
  recovery code.

TODO: Do most systems currently give the user new recovery codes during this flow?


## Redirect Grant

No changes are made to the authorization code flow after the client
receives a redirect challenge.


## OTP Grant

TODO: OTP only, not in response to an MFA challenge.

The client makes a request to the token endpoint by adding the
following parameters using the "application/x-www-form-urlencoded"
format with a character encoding of UTF-8 in the HTTP request body:

"grant_type":
: REQUIRED.  `urn:ietf:params:oauth:grant-type:otp`

"username":
: REQUIRED. The resource owner username.

"otp":
: REQUIRED. The one-time password generated by a device.

"client_id":
: REQUIRED, if the client is not authenticating with the
  authorization server as described in Section 3.2.1 of OAuth 2.0
  {{RFC6749}}.

"device_session":
: OPTIONAL. The `device_session` if the client has one.



## OOB Grant

TODO: OOB only, not in response to an MFA challenge. Looks almost just like the MFA+OOB grant, just without the mfa_token.






## Device Session {#device-session}

In addition to the parameters defined in Section 5.1 of OAuth 2.0
{{RFC6749}}, the following additional parameters are specified for
the token response for any grant type defined by or extended from this specification.

"device_session":
: OPTIONAL.  The device session contains relevant data to the device and the current user authenticated with the device.

The device session is completely opaque to the client, and as such the AS MUST adequately protect the value such as using a JWE if the AS is not maintaining state on the backend.

The device session can be used by the client on a subsequent authorization initiation request, described in {{authorization-initiation}}, or in an authorization challenge request, described in {{authorization-challenge-request}}.




# Refresh Token Grant

TODO: Describe how the AS could return a challenge to the client on the normal refresh token request
that tells the client they need to get the user to re-authenticate or provide an MFA token.

(No normative changes are required beyond what has already been described in the draft at this point.)



# Security Considerations

TODO Security

## Native client and Authorisation Server trust relationship

TODO: Emphasise the first-party 'same owner' relationship between the native client and the authorisation server


## Phishing

TODO: Describe the phishing risk this opens up since the client is collecting and providing all the user's information to the authorization server. The AS MAY decide to require the user go through a redirect flow at any stage of the process based on its own risk assessment.



## Client Authentication

TODO: Describe the likely lack of client authentication because this is expected to be deployed by native apps. Maybe mention alternatives to client authentication such as App Attestation, or using a risk engine to analyze other aspects of the request from the client.


## Leaking Information

TODO: Attackers may be able to query the AS with user IDs to learn whether an identifier corresponds to an active account and which types of authentication a particular account has configured. What mitigations needed here? Possibly avoiding returning an error for unrecognized identifiers.


## Notification fatigue

TODO: A client may be able to cause repeated notifications to any user given their user ID.

Mitigations:

* rate limiting
* use notification-based methods only as a secondary factor or when using a previously issued device_session






# IANA Considerations

## OAuth Parameter Registration

TODO

## OAuth URI Registration

This specification registers the following values in the IANA "OAuth URI"
registry (IANA.OAuth.Parameters) established by {{RFC6755}}.

    URN: `urn:ietf:params:oauth:grant-type:...`
    Common Name: ...
    Change Controller: IESG
    Specification Document:

    URN: `urn:ietf:params:oauth:grant-type:...`
    Common Name: ...
    Change Controller: IESG
    Specification Document:

    URN: `urn:ietf:params:oauth:grant-type:...`
    Common Name: ...
    Change Controller: IESG
    Specification Document:

    URN: `urn:ietf:params:oauth:grant-type:...`
    Common Name: ...
    Change Controller: IESG
    Specification Document:


## OAuth Extensions Error Registration

This specification registers the following values in the IANA "OAuth
Extensions Error Registry" registry (IANA.OAuth.Parameters)
established by {{RFC6749}}.

    Name: ...

## Authorization Server Metadata

This specification defines two new endpoints at the authorization server.
TODO: Need to register these in the Authorization Server Metadata document too.

* authorization initiation endpoint
* authorization challenge endpoint


--- back


# Use Cases

TODO: Outlining these in the document will also help guide developers to understand what they can (and cannot) expect to be able to do. For each use case, briefly describe the expected outcome, and outline each step and which part of the spec accomplishes each step.

### Password + MFA

* User enters username and password (/initiate login_hint= password=)
* AS responds with `mfa_required`
* Client initiates MFA challenge (authorization challenge endpoint)
* AS responds with details of MFA challenge
* Client collects MFA from user (OTP, SMS code, etc)
* Client makes token request (new mfa-otp/mfa-oob grant type)

### Passwordless OTP

* Client collects username and OTP from user
* Client makes token request (new otp grant type)

### Re-authenticating to an app a week later

* You log in to an app (in any way, redirect flow or direct)
* App gets short lived access token and long lived refresh token
* A week later, the user launches the app, access token is expired
* App uses the refresh token, gets back an `mfa_required` error
* App prompts and collects MFA from user (OTP or OOB)
* App submits token request with grant_type=mfa_otp and mfa_token (which was already associated with the refresh token)

### Registration

* Client asks for user's email, starts a flow (/initiate login_hint=email@example.com)
* AS responds with `{"challenge_type":"oob","oob_code":"...",etc}`
* Client collects OOB code from user
* Client makes token request
* Client receives an access token and device_session
* Client collects the user's name, updates their profile using the access token
* Client asks for the user's phone number
* Client starts a new flow (/initiate login_hint=+1phone device_session=X)
* AS recognizes existing device_session, responds with `{"challenge_type":"oob","oob_code":"...",etc}`
* Client collects OOB code from user
* Client makes token request
* Client receives an access token, now associated with a new account with both a verified email and phone number


### Sign-in with only email verification
e.g. user provides e-mail, is sent a verification code, and then uses the verification code to prove they control the e-mail. Can also be used for registration.

### Discover supported authentication methods
e.g. developer can query the authorisation server to determine what authentication methods are supported.

### Discover supported account recover authentication methods
e.g. developer can query the authorisation server to determine what authentication methods are used for account recovery if one of the methods are lost.

### Update an existing authentication method
e.g. the authorisation server may require the user to update a password or provide a new phone number, key or alternative e-mail address if it believes the existing mechanism is no longer trustworthy.

### Initiate browser-based interaction for certain scenarios
e.g. some scenarios don't benefit from a native implementation and may be more efficiently or securely implemented through the browser (e.g. error scenarios, password recovery, identity verification, social sign-in).

### Discovering custom user attributes
e.g. Ability to know mandatory and optional custom attributes configured on the authorisation server (can this be achieve through AS metadata instead of as part of the protocol)?

### Provide different UX depending on a user's enrolled authenticators
An AS may support multiple different authenticators, and a user may have set up only one. A client will not know which a user has set up ahead of time. The AS needs to drive the UI in the client login process depending on the configuration of the user account.

### Register new authentication methods
e.g. user selects password, provides phone for SMS codes, etc

### User registration
e.g. user enters a new email, completes the email challenge, the AS creates a new account for them, then wants to enroll additional factors such as SMS



# Old Notes

## Client receives trigger for authentication

TODO: The client may receive a trigger from the resource server to initiate an authentication, possibly with a hint from the resource server in the form of an acr value, indicating it needs to initiate and authentications.

TBD: May need to reference the existing step-up authentication draft instead


## Checking for additional requirements

If the initial set of information provided by the client is correct, the AS
MAY choose to either respond immediately with a successful token response,
or prompt the client for an additional challenge.

For example, the AS could first require the client prompt the user for a one-time-code they
received via email, and then in a second step, ask the client to prompt the user




# Acknowledgments

TODO acknowledge.
