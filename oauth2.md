<a name="#top">Back to Top</a>

# Oauth2 Features

Stormpath provides three Oauth2 grant types, `client_credentials`, `password`, and `refresh_token`.  In
this document we discuss how these should be exposed in the developer's web
application.

## OAuth2 Configuration Options

The OAuth2 endpoint can be configured, or disabled entirely:

```yaml
web:
  oauth2:
    enabled: true
    uri: "/oauth/token"
```

#### stormpath.web.oauth2.enabled

By default we accept POSTS to this token uri and respond according to the
grant type.  If the developer sets this value to false we should not attach any
handler to this uri, allowing the framework to return it's default 404 response.

## Errors

If the grant type requested is not enabled, then we
should return the OAuth-compliant error code:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "error": "unsupported_grant_type"
    }

If no grant type is specified, the error should be `invalid_request`.

A GET to the endpoint URL should always return `405 Method Not Allowed`.

#### Error Transformations

Error responses from the application's `/oauth/token` endpoint should be
transformed to only include the message and error properties (see
[Error Handling][]).  This is an OAuth nuance that is only available on error
responses from this endpoint.  As such, an error response for this flow would
look like this:

```
{
  "message": "grant_type passwordx is an unsupported value.",
  "error": "invalid_request"
}
```

## Client Credentials Grant Flow

The product guide for this feature can be found here:

https://docs.stormpath.com/guides/api-key-management/

In this workflow, an api key and secret is provisioned for a stormpath account.
These credentials can be exchanged for an access token by making a POST request
to `/oauth/token` on the web application.  The request must look like this:

````
POST /oauth/token
Authorization: Basic <base64UrlEncoded(apiKeyId:apiKeySecret)>

grant_type=client_credentials
````

For this flow the underlying Stormpath SDK is used for generating the access
token and verifying that the API key & secret is valid, and that the account is
not disabled.  The response is an access token response, but this flow does not
return a refresh token (only an access token):

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA...",
  "expires_in":3600,
  "token_type":"Bearer"
}
```

### Client Credentials Options

```yaml
stormpath:
  web:
    oauth2:
      client_credentials:
        enabled: true
```

#### stormpath.web.oauth2.client_credentials.enabled

If set to false, the endpoint should return the `unsupported_grant_type` error
(see above).

## Password Grant Flow

The product guide for this feature can be found here:

http://docs.stormpath.com/guides/token-management/

In this workflow, an account can post their login (username or email) and
password to the `/oauth/token` endpoint, with the following body data:

````
POST /oauth/token

grant_type=password
&username=<username>
&password=<password>
````

Although the Oauth2 spec requires the parameter to be named `username`, this
value can also be the email address of a Stormpath account.

The framework should use the underlying SDK to exchange these credentials
for an access token response, using the `/oauth/token` endpoint of the
Stormpath Application that is specified in the configuration.

If the login is successful, we respond with an access token and refresh token.
This means that you need to remove the `stormpath_access_token_href` from the
API response.  Thus, the format of the response should be an OAuth2 access token
response, which looks like this:

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "token_type":"Bearer"
}
```

### Password Grant Options

```yaml
stormpath:
  web:
    oauth2:
      password:
        enabled: true
        validationStrategy: stormpath # or 'local' to do local JWT signature validation
```


#### stormpath.web.oauth2.password.enabled

If set to false, the endpoint should return the `unsupported_grant_type` error
(see above).

#### stormpath.web.oauth2.password.validationStrategies

Determines how we validate an access token.  There are two strategies:

* **local** - only verify the signature, expiration, and issuer

* **stormpath** - always validate against Stormpath, for additional checks
  such as Account and Application status.  This is the default, but the most
  consistent (will always be aware of Stormpath resource statuses).

The underlying SDK must have authenticators which can do either type of
validation.  The purpose of this configuration option is to inform the framework
intergration as to which type of authenticator it should build and use for
authentication attempts.

## Refresh Grant Flow

The product guide for this feature is found at: http://docs.stormpath.com/guides/token-management/#refreshing-access-tokens

The refresh grant type is required for clients using the password grant type to refresh their access_token. Thus, it's automatically enabled alongside the password grant type.

An account can post their refresh_token with the following body data:

```
POST /oauth/token
grant_type=refresh_token&
refresh_token=<refresh token>
```

The response is the same format as the password grant type. 

[Error Handling]: error-handling.md
