---
title: "Client-Server API"
weight: 10
type: docs
---

The client-server API allows clients to
send messages, control rooms and synchronise conversation history. It is
designed to support both lightweight clients which store no state and
lazy-load data from the server as required - as well as heavyweight
clients which maintain a full local persistent copy of server state.

## API Standards

The mandatory baseline for client-server communication in SDM is
exchanging JSON objects over HTTP APIs. More efficient transports may be
specified in future as optional extensions.

HTTPS is recommended for communication. The use of plain HTTP is not
recommended outside test environments.

Clients are authenticated using opaque `access_token` strings (see [Client
Authentication](#client-authentication) for details).

All `POST` and `PUT` endpoints, with the exception of
[`POST /_api/media/v3/upload`](#post_apimediav3upload), require the
client to supply a request body containing a (potentially empty) JSON object.
Clients should supply a `Content-Type` header of `application/json` for all requests with JSON bodies,
but this is not required.

Similarly, all endpoints require the server to return a JSON object,
with the exception of 200 responses to
[`GET /_api/media/v3/download/{serverName}/{mediaId}`](#get_apimediav3downloadservernamemediaid)
and [`GET /_api/media/v3/thumbnail/{serverName}/{mediaId}`](#get_apimediav3thumbnailservernamemediaid).
Servers must include a `Content-Type` header of `application/json` for all JSON responses.

All JSON data, in requests or responses, must be encoded using UTF-8.


### Standard error response

Any errors which occur at the SDM API level MUST return a "standard
error response". This is a JSON object which looks like:

```json
{
  "errcode": "<error code>",
  "error": "<error message>"
}
```

The `error` string will be a human-readable error message, usually a
sentence explaining what went wrong.

The `errcode` string will be a unique string which can be used to handle an
error message e.g.  `M_FORBIDDEN`. Error codes should have their namespace
first in ALL CAPS, followed by a single `_`. For example, if there was a custom
namespace `com.mydomain.here`, and a `FORBIDDEN` code, the error code should
look like `COM.MYDOMAIN.HERE_FORBIDDEN`. Error codes defined by this
specification should start `M_`.

Some `errcode`s define additional keys which should be present in the error
response object, but the keys `error` and `errcode` MUST always be present.

Errors are generally best expressed by their error code rather than the
HTTP status code returned. When encountering the error code `M_UNKNOWN`,
clients should prefer the HTTP status code as a more reliable reference
for what the issue was. For example, if the client receives an error
code of `M_NOT_FOUND` but the request gave a 400 Bad Request status
code, the client should treat the error as if the resource was not
found. However, if the client were to receive an error code of
`M_UNKNOWN` with a 400 Bad Request, the client should assume that the
request being made was invalid.

#### Common error codes

These error codes can be returned by any API endpoint:

`M_FORBIDDEN`
Forbidden access, e.g. joining a room without permission, failed login.

`M_UNKNOWN_TOKEN`
The access or refresh token specified was not recognised.

An additional response parameter, `soft_logout`, might be present on the
response for 401 HTTP status codes. See [the soft logout
section](#soft-logout) for more information.

`M_MISSING_TOKEN`
No access token was specified for the request.

`M_BAD_JSON`
Request contained valid JSON, but it was malformed in some way, e.g.
missing required keys, invalid values for keys.

`M_NOT_JSON`
Request did not contain valid JSON.

`M_NOT_FOUND`
No resource was found for this request.

`M_LIMIT_EXCEEDED`
Too many requests have been sent in a short period of time. Wait a while
then try again.

`M_UNRECOGNIZED`
The server did not understand the request. This is expected to be returned with
a 404 HTTP status code if the endpoint is not implemented or a 405 HTTP status
code if the endpoint is implemented, but the incorrect HTTP method is used.

`M_UNKNOWN`
An unknown error has occurred.

#### Other error codes

The following error codes are specific to certain endpoints.

<!-- TODO: move them to the endpoints that return them -->

`M_UNAUTHORIZED`
The request was not correctly authorized. Usually due to login failures.

`M_USER_DEACTIVATED`
The user ID associated with the request has been deactivated. Typically
for endpoints that prove authentication, such as `/login`.

`M_USER_IN_USE`
Encountered when trying to register a user ID which has been taken.

`M_INVALID_USERNAME`
Encountered when trying to register a user ID which is not valid.

`M_ROOM_IN_USE`
Sent when the room alias given to the `createRoom` API is already in
use.

`M_INVALID_ROOM_STATE`
Sent when the initial state given to the `createRoom` API is invalid.

`M_THREEPID_IN_USE`
Sent when a threepid given to an API cannot be used because the same
threepid is already in use.

`M_THREEPID_NOT_FOUND`
Sent when a threepid given to an API cannot be used because no record
matching the threepid was found.

`M_THREEPID_AUTH_FAILED`
Authentication could not be performed on the third party identifier.

`M_THREEPID_DENIED`
The server does not permit this third party identifier. This may happen
if the server only permits, for example, email addresses from a
particular domain.

`M_SERVER_NOT_TRUSTED`
The client's request used a third party server, e.g. identity server,
that this server does not trust.

`M_UNSUPPORTED_ROOM_VERSION`
The client's request to create a room used a room version that the
server does not support.

`M_INCOMPATIBLE_ROOM_VERSION`
The client attempted to join a room that has a version the server does
not support. Inspect the `room_version` property of the error response
for the room's version.

`M_BAD_STATE`
The state change requested cannot be performed, such as attempting to
unban a user who is not banned.

`M_GUEST_ACCESS_FORBIDDEN`
The room or resource does not permit guests to access it.

`M_CAPTCHA_NEEDED`
A Captcha is required to complete the request.

`M_CAPTCHA_INVALID`
The Captcha provided did not match what was expected.

`M_MISSING_PARAM`
A required parameter was missing from the request.

`M_INVALID_PARAM`
A parameter that was specified has the wrong value. For example, the
server expected an integer and instead received a string.

`M_TOO_LARGE`
The request or entity was too large.

`M_EXCLUSIVE`
The resource being requested is reserved by an application service, or
the application service making the request has not created the resource.

`M_RESOURCE_LIMIT_EXCEEDED`
The request cannot be completed because the homeserver has reached a
resource limit imposed on it. For example, a homeserver held in a shared
hosting environment may reach a resource limit if it starts using too
much memory or disk space. The error MUST have an `admin_contact` field
to provide the user receiving the error a place to reach out to.
Typically, this error will appear on routes which attempt to modify
state (e.g.: sending messages, account data, etc) and not routes which
only read state (e.g.: `/sync`, get account data, etc).

`M_CANNOT_LEAVE_SERVER_NOTICE_ROOM`
The user is unable to reject an invite to join the server notices room.
See the [Server Notices](#server-notices) module for more information.

## Client Authentication

Most API endpoints require the user to identify themselves by presenting
previously obtained credentials in the form of an `access_token` query
parameter or through an Authorization Header of `Bearer $access_token`.
An access token is typically obtained via the [Login](#login) or
[Registration](#account-registration-and-management) processes. Access tokens
can expire; a new access token can be generated by using a refresh token.

{{% boxes/note %}}
This specification does not mandate a particular format for the access
token. Clients should treat it as an opaque byte sequence. Servers are
free to choose an appropriate format. Server implementors may like to
investigate [macaroons](http://research.google.com/pubs/pub41892.html).
{{% /boxes/note %}}

### Using access tokens

Access tokens may be provided in two ways, both of which the homeserver
MUST support:

1.  Via a query string parameter, `access_token=TheTokenHere`.
2.  Via a request header, `Authorization: Bearer TheTokenHere`.

Clients are encouraged to use the `Authorization` header where possible
to prevent the access token being leaked in access/HTTP logs. The query
string should only be used in cases where the `Authorization` header is
inaccessible for the client.

When credentials are required but missing or invalid, the HTTP call will
return with a status of 401 and the error code, `M_MISSING_TOKEN` or
`M_UNKNOWN_TOKEN` respectively.  Note that an error code of `M_UNKNOWN_TOKEN`
could mean one of four things:

1. the access token was never valid.
2. the access token has been logged out.
3. the access token has been [soft logged out](#soft-logout).
4. {{< added-in v="1.3" >}} the access token [needs to be refreshed](#refreshing-access-tokens).

When a client receives an error code of `M_UNKNOWN_TOKEN`, it should:

- attempt to [refresh the token](#refreshing-access-tokens), if it has a refresh
  token;
- if [`soft_logout`](#soft-logout) is set to `true`, it can offer to
  re-log in the user, retaining any of the client's persisted
  information;
- otherwise, consider the user as having been logged out.

### Relationship between access tokens and devices

Client [devices](../index.html#devices) are closely related to access
tokens and refresh tokens. SDM servers should record which device
each access token and refresh token are assigned to, so that
subsequent requests can be handled correctly. When a refresh token is
used to generate a new access token and refresh token, the new access
and refresh tokens are now bound to the device associated with the
initial refresh token.

By default, the [Login](#login) and [Registration](#account-registration-and-management)
processes auto-generate a new `device_id`. A client is also free to
generate its own `device_id` or, provided the user remains the same,
reuse a device: in either case the client should pass the `device_id` in
the request body. If the client sets the `device_id`, the server will
invalidate any access and refresh tokens previously assigned to that device.

### Refreshing access tokens

{{% added-in v="1.3" %}}

Access tokens can expire after a certain amount of time. Any HTTP calls that
use an expired access token will return with an error code `M_UNKNOWN_TOKEN`,
preferably with `soft_logout: true`. When a client receives this error and it
has a refresh token, it should attempt to refresh the access token by calling
[`/refresh`](#post_apiclientv3refresh). Clients can also refresh their
access token at any time, even if it has not yet expired. If the token refresh
succeeds, the client should use the new token for future requests, and can
re-try previously-failed requests with the new token. When an access token is
refreshed, a new refresh token may be returned; if a new refresh token is
given, the old refresh token will be invalidated, and the new refresh token
should be used when the access token needs to be refreshed.

The old refresh token remains valid until the new access token or refresh token
is used, at which point the old refresh token is revoked. This ensures that if
a client fails to receive or persist the new tokens, it will be able to repeat
the refresh operation.

If the token refresh fails and the error response included a `soft_logout:
true` property, then the client can treat it as a [soft logout](#soft-logout)
and attempt to obtain a new access token by re-logging in. If the error
response does not include a `soft_logout: true` property, the client should
consider the user as being logged out.

Handling of clients that do not support refresh tokens is up to the homeserver;
clients indicate their support for refresh tokens by including a
`refresh_token: true` property in the request body of the
[`/login`](#post_apiclientv3login) and
[`/register`](#post_apiclientv3register) endpoints. For example, homeservers
may allow the use of non-expiring access tokens, or may expire access tokens
anyways and rely on soft logout behaviour on clients that don't support
refreshing.