# Apostle API standard

## Guidelines

This document provides guidelines and examples for the Apostle APIs,encouraging consistency, maintainability, and best practices.

### Sources

 * [GoCardless API Specification](https://github.com/gocardless/http-api-design/blob/master/README.md)

## JSON API

All endpoints MUST follow the [JSON API specification](http://jsonapi.org/format/).

## JSON Only

The Apostle API MUST accept only JSON.

Reference: http://www.mnot.net/blog/2012/04/13/json_or_xml_just_decide

## General guidelines

 * An URL identifies a resource.
 * URLs SHOULD use nouns only, no verbs.
 * For consistency, use plural nouns only.
 * Use HTTP verbs (GET, POST, PUT, DELETE) to operate on resources.
 * Do not nest resources - it enforces relationships that could change and makes clients harder to write.
     - Use filtering `/messages?connection=xyz` rather than `/connections/xyz/messages`
 * API versions SHOULD be represented as dates documented in a changelog Version number SHOULD NOT be in the url.
 * APIs SHOULD be behind a subdomain: `api.apostle.nl`

## RESTful URLs

### Good URL examples

 * List of connections
     - GET `https://api.apostle.nl/connections`
 * Filtering is a query
     - GET `https://api.apostle.nl/connections?state=error&sort=name`
     - GET `https://api.apostle.nl/connections?sort=created`
 * A single connection
     - GET `https://api.apostle.nl/connections/1234`
 * All messages received or sent with this connection
     - GET `https://api.apostle.nl/messages?connection=1234`
 * Get multiple resources
     - GET `https://api.apostle.nl/messages/1234,5678,9012`
 * Actions on resource
     - POST `https://api.apostle.nl/messages/1234/actions/approve`

### Bad URL examples

 * Non-plural noun
     - `https://api.apostle.nl/connection`
     - `https://api.apostle.nl/connection/1234`
     - `https://api.apostle.nl/connection/1234/action`
 * Verb in URL
     - `https://api.apostle.nl/connection/create`
 * Nested resource
     - `https://api.apostle.nl/connections/1234/messages`
 * Filter outside of query string
     - `https://api.apostle.nl/connections/1234/name`
 * Filtering to get multiple resources
     - `https://api.apostle.nl/connections?id[]=12&id[]=34`

## Actions

Avoid resource actions. Create seperate resources where possible:

### Good
```
POST /messages?connection=1234&body=Lorem%20ipsum HTTP/1.1
```

### Bad
```
POST /connections/1234/messages HTTP/1.1
```

Where special actions are REQUIRED, place them under an `actions` prefix.

```
POST /messages/ID/actions/approve HTTP/1.1
```

## Responses

Don't set values in keys.

### Good

No values in keys:

```
"data": [
    { "id": "1234", "name": "Foo" },
    { "id": "5678", "name": "Bar" }
]
```

### Bad

Values in keys:

```
"data": {
    "1234": "Foo",
    "5678": "Bar"
}
```

## String IDs

Always return string IDs, some languages like JavaScript don't support big integers. Serialize/deserialize integers to strings if storing IDs as integers.

## Error handling

Error responses SHOULD include a message for the user, internal error type (corresponding to some specific internally determined constant, which MUST be represented as a string), links where developers can find more info.

There MUST only be one top level error. Errors SHOULD be returned in turn. This makes internal error logic simpler. It also makes it easier for API consumers to deal with errors.

Validation and resource errors are nested in the top level error under `errors`.

The error is nested in `error` to make it possible to add, for example, deprecation errors on successful requests.

The HTTP status `code` is used as a top level error, `type` is used a sub error. Nested `errors` MAY have more specific `type` errors like `invalid_field`.

### Separate formatting errors from errors the integration SHOULD handle

Formatting errors include field presence, length etc. Returned when the data you have passed is wrong.

An error that SHOULD be handled in the integration could be when attempting to create a payment against a mandate and the mandate has expired. This is a edge case that needs handling in the API integration. Do not mask these errors as validation errors. Always return these types of errors as a top level error.

### Top level errors

 * Top level errors MUST implement `type`, `reason`, `code`,`message`.
 * `type` MUST relate to the `reason`, use it to categorise the error, e.g. `api_error`.
 * `reason` MUST be specific to the error.
 * `message` MUST be specific.
 * Top level errors MAY implement `documentation_url`, `request_url `, `id `.
 * Only return `id` for server errors (5xx). The `id` SHOULD point to the exception you track internally.

```
{
  "error": {
    "documentation_url": "https://api.apostle.nl/docs/beta/errors#access_forbidden",
    "request_url": "https://api.apostle.nl/requests/REQUEST_ID",
    "request_id": "REQUEST_ID",
    "id": "ERROR_ID",
    "type": "access_forbidden",
    "code": 403,
    "message": "You don't have the right permissions to access this resource"
  }
}
```

### Nested errors

 * Nested errors MUST implement `reason`, `message`.
 * `reason` MUST be specific to the error.
 * Nested errors MAY implement `field`.

```
{
  "error": {
    "top level errors": "...",

    "errors": [{
      "field": "account_number",
      "reason": "missing_field",
      "message": "Account number is REQUIRED"
    }]
  }
}
```

### HTTP Status Code summary

##### `200 OK` - Everything worked as expected.
##### `400 Bad Request` - Eg. invalid JSON.
##### `401 Unauthorized` - No valid API key provided.
##### `402 Request Failed` - Parameters were valid but request failed.
##### `403 Forbidden` - Missing or invalid permissions.
##### `422 Unprocessable Entity` - Parameters were invalid/validation failed.
##### `404 Not Found` - The requested item doesn't exist.
##### `500`, `502`, `503`, `504 Server errors` - something went wrong on our end.

##### `400 Bad Request`

 * When the request body contains malformed JSON.
 * When the JSON is valid, but the document structure is invalid (e.g. when passing an array when you SHOULD be passing an object).

##### `422 Unprocessable Entity`

 * When model validations fail for fields (e.g. name to long).
 * When creating a resource with a related resource being in a bad state.

## What changes are considered "backwards compatible"?

 * Adding new API resources.
 * Adding new OPTIONAL request parameters to existing API methods.
 * Adding new properties to existing API responses.
 * Changing the order of properties in existing API responses.
 * Changing the length or format of object IDs or other opaque strings.
 * You can safely assume object IDs we generate will never exceed 128 characters, but you SHOULD be able to handle IDs of up to that length.
 * Adding new event types. Your webhook listener SHOULD gracefully handle unfamiliar events types.

## Versioning changes

The versioning scheme is designed to promote incremental improvement to the API and discourage rewrites.

WebHooks/Server initiated events SHOULD NOT contain serialised versions of resources. Instead provide an id to the resource that changed and let the client request it using a version.

### Version maintenance

Maintain versions for at least 3 months.

### Implementation guidelines

The API version MUST be set using a custom HTTP header. API version MUST NOT be defined in the URL structure (e.g. `/v1`) this makes incremental change impossible.

#### HTTP Header
`Version: 2014-12-01`

Enforce the header on all requests.

Validate the version against available versions. Do not allow dates up to a version.

The API changelog MUST only contain backwards-incompatible changes. All non-breaking changes are automatically available to old versions.

Reference: https://stripe.com/docs/upgrades

## X-Header

The use of `X-Custom-Header` has been deprecated, see: http://tools.ietf.org/html/rfc6648

## Resource filtering

Resource filters SHOULD always be in singular form.

Multiple IDs supplied to one filter, as a comma separated list, SHOULD be translated into an `OR` query. Chaining multiple filters with `&` SHOULD be translated into an `AND` query.

### Good

```
GET /messages?connection=1234,5678&type=planned HTTP/1.1
```

### Bad

```
GET /messages/connections=1234,5678&type=planned HTTP/1.1
```

## Pagination

All list/index endpoints MUST be paginated by default.
Pagination MUST be reverse chronological. To enable cursor based pagination, IDs SHOULD be increasing.

Only support cursor or time based pagination.

Defaults:

 * `limit`: 50
 * `after`: Newest resource
 * `before`: `null`

Limits:

 * `limit`: 500

Parameters:

| Name          | Type          | Description           |
| ------------- |:-------------:| -------------------- :|
| `after`       | string        | id to start after     |
| `before`      | string        | id to start before    |
| `limit`       | string        | number of records     |

## Response

Paginated results are always enveloped:

```
{
  "meta": {
    "cursors": {
      "after": "abcd1234",
      "before": "wxyz0987"
    },
    "limit": 50
  },
  "data": [{
    ...
  }]
}
```

## Updates

Full/partial updates using `PUT` SHOULD replace any parameters passed and ignore fields not submitted.

## JSON encode POST & PUT bodies

POST, PUT & PATCH expect JSON bodies in request. `Content-Type` header is REQUIRED to be set to `application/json`. For unsupported media types a `415 Unsupported Media Type` response code is returned.

## Pretty printed responses

JSON responses SHOULD be pretty printed.

## Time zone/dates

Explicitly provide an ISO8601 timestamp with timezone information (date time in UTC). For API calls that allow for a timestamp to be specified, we use that exact timestamp. These timestamps look something like `2014-02-27T15:05:06+01:00`. ISO 8601 UTC format: `YYYY-MM-DDTHH:MM:SSZ`.

## HTTP Rate Limiting

All endpoints MUST be rate limited.

You can check the returned HTTP headers of any API request to see your current rate limit status:

```
Rate-Limit-Limit: 5000
Rate-Limit-Remaining: 4994
Rate-Limit-Reset: Thu, 01 Dec 1994 16:00:00 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Retry-After: Thu, 01 MAY 2014 16:00:00 GMT

RateLimit-Reset uses the HTTP header date format: RFC 1123 (Thu, 01 Dec 1994 16:00:00 GMT)
```

Exceeding rate limit:

```
// 429 Too Many Requests
{
    "message": "API rate limit exceeded.",
    "type": "rate_limit_exceeded",
    "documentation_url": "http://developer.apostle.nl/#rate_limit_exceeded"
}
```

## CORS

Support Cross Origin Resource Sharing (CORS) for AJAX requests. You can read the CORS W3C working draft, or this intro from the HTML 5 Security Guide.

Any domain that is registered against the requesting account is accepted.

```
$ curl -i https://api.apostle.nl -H "Origin: http://dvla.com"
```

```
HTTP/1.1 302 Found
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: ETag, Link, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset, OAuth-Scopes, Accepted-OAuth-Scopes
Access-Control-Allow-Credentials: false

// CORS Preflight request
// OPTIONS 200
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-Modified-Since, If-None-Match, If-Unmodified-Since, Requested-With
Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE
Access-Control-Expose-Headers: ETag, RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: false
```

## TLS/SSL

All API request MUST be made over SSL. Any non-secure requests will return ssl_REQUIRED. No redirects are made. Outgoing web hooks MUST be SSL.

```
HTTP/1.1 403 Forbidden
Content-Length: 35

{
  "message": "API requests MUST be made over HTTPS",
  "type": "ssl_REQUIRED",
  "docs": "https://developer.apostle.nl/errors#ssl_REQUIRED"
}
```
