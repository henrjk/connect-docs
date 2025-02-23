# Clients


After setting up your Anvil Connect server, the next step is to register apps and services, called clients, to obtain credentials and configure relevant [properties](#client-properties), depending on the type of client. Once you've done this, you can add a library dependency to your client code and configure it to use your auth server and credentials.


### Trusted Clients

As an identity provider, Anvil Connect can optionally authenticate users of third party apps against your auth server. When users come from another developer's app, the server prompts users to authorize access to their account. This is important when sharing data outside of your own realm, but unnecessary for your own apps. To streamline the user experience, you can register "trusted" apps.


### Application Type

The application type, or type of client, is determined by the `application_type` property, which defaults to `web`. Anvil Connect supports the `application_type` values [defined in the OpenID Connect standard][oidc-client-reg-metadata]:

 - **`web`** - server-side apps like you might build with Rails or HTML5 apps that run entirely in the browser
 - **`native`** - iOS, Android, desktop, and CLI apps

In addition, Anvil Connect supports the **`service`** application type for use with RESTful and realtime APIs. Users typically don't interact directly with services; use this type if your client will acquire client-only credentials without user intervention.

[oidc-client-reg-metadata]: https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata


### Grant Types

Grant types determine what kind of _authorization grants_ the client is allowed to request. Anvil Connect supports three grant types derived from the OpenID Connect specification:

 - **`authorization_code`** - for server-side apps that can obtain tokens from the auth server behind the scenes
 - **`implicit`** - for apps that run on the client and need to get tokens directly from the auth server
 - **`refresh_token`** - for apps that need to "refresh" tokens when the user needs to stay signed in for prolonged periods

If left unspecified, the allowed grant types for a client default to only `authorization_code`.

### Response Types

Response types are related to grant types, but instead determine the exact types of tokens/data the client expects to handle. The supported response types are:

 - **`code`** - an authorization code used to obtain tokens
 - **`token`** - an access token, used to access resources such as the `userinfo` endpoint
 - **`id_token`** - an ID token, which identifies the signed-in user
 - **`none`** - nothing - the client just wants to check if authentication worked without touching any further data; cannot be used with other response types

If left unspecified, the allowed response types for a client default to only `code`.

Response types can be combined into sets to indicate that a client intends to use certain combinations together. For example, `code token` indicates that the client would like to obtain an authorization code together with an access token.

Each combination of response types must be explicitly defined for those response types to be used together in a request. Each possible permutation of that combination does not need to specified, however.

For example, assuming that a client needs to obtain an access token, an ID token, and an authorization code, the client will need to have `code id_token token` in its registered list of response types. `code token id_token`, `token code id_token`, etc. would all have the same effect.

Response types require that their related grant types are set on the client:

Response type | Required grant type
------------- | -------------------
`code` | `authorization_code`
`token` | `implicit`
`id_token` | `implicit`
`none` | N/A

## Registration

Registering a client creates a set of credentials that are required for interacting with Anvil Connect. You can set [properties](#client-properties) on a client that contain helpful information and also determine the way that client works depending on the context.


Although not strictly required, we recommend setting (at a minimum) `client_name` and `client_uri` for all clients. Web and native applications require one or more redirect_uris to be configured and should also define at least one `post_logout_redirect_uri` as well. When registering your own apps as clients, we recommended setting `trusted` to `true`.

There are three ways to register clients with Anvil Connect. You can use the Anvil Connect CLI, the Dynamic Registration endpoint, or the REST API.

### CLI command

The quickest way to register a client is with the `nvl` CLI tool. You will need to [have `nvl` configured for your server](cli.md#setting-up-a-server-for-administration).

First, log in to the server.

<pre>
$ nvl login

? <b>Select an Anvil Connect instance</b> (Use arrow keys)
❯ connect.example.com (connect-example.com)
  localhost:3000 (localhost-3000)
Selected issuer connect.example.com (https://connect.example.com)
? <b>Enter your email</b> admin@example.com
? <b>Enter your password</b> **********
You have been successfully logged in to connect.example.com
</pre>

Then, run the `client:register` command. Run without arguments, it will prompt for the client details. The supported arguments are [available in the CLI reference](cli.md#clientregister).

<pre>
$ nvl client:register

? <b>Select an Anvil Connect instance</b> (Use arrow keys)
❯ connect.example.com (connect-example.com)
  localhost:3000 (localhost-3000)
Selected issuer connect.example.com (https://connect.example.com)
? <b>Will this be a trusted client?</b> (y/N) N
? <b>Enter a name</b> Test App
? <b>Enter a URI</b> https://app.example.com
? <b>Enter a logo URI</b> https://app.example.com/logo.svg
? <b>Choose an application type</b> (Use arrow keys)
❯ web 
  native 
  service
? <b>Select response types</b> (Press &lt;space&gt; to select)
❯◯ code
 ◯ id_token token
 ◯ code id_token token
 ◯ token
 ◯ none
? <b>Select grant types</b> (Press &lt;space&gt; to select)
❯◯ authorization_code 
 ◯ implicit
 ◯ refresh_token
 ◯ client_credentials
? <b>Set the default max age</b> (in seconds) (3600)
? <b>Define redirect URIs</b> https://app.example.com/callback
? <b>Define redirect URIs</b> https://app.example.com/rp
? <b>Define redirect URIs</b>
? <b>Define post logout redirect URIs</b> https://app.example.com/
? <b>Define post logout redirect URIs</b>

Name					<b>Test App</b>
Client ID				95c9d5ae-63a9-4278-919b-b551c6a3f6ba
Client Secret			8eb670c2d09ba6978d97
Client URI				https://app.example.com
Application type		web
Response types			code
Grant types				authorization_code
Token auth method		client_secret_basic
Default max age			3600
Redirect URIs			https://app.example.com/callback
						https://app.example.com/rp
Logout Redirect URIs	https://app.example.com/
Trusted					false
Created					Fri Sep 25 2015 10:11:41 GMT-0700 (PDT)
Modified				Fri Sep 25 2015 10:11:41 GMT-0700 (PDT)
</pre>

If you are not yet running or unable to run `nvl`, you may use `nv`. Run it from the root of your project. **Note that this method of registering clients is deprecated and will be removed in a future release.**

```bash
$ nv add client '{
  "client_name": "Example App",
  "default_max_age": 36000,
  "response_types": ["code"],
  "grant_types": ["authorization_code"],
  "redirect_uris": ["http://localhost:9000/callback.html"],
  "post_logout_redirect_uris": ["http://localhost:9000"],
  "trusted": true
}'
```

### Dynamic Registration

The `/register` endpoint is configured by default to require an access token issued with `realm` scope. To use it you can either obtain a token with that scope or configure the server's [client_registration](http://localhost:1313/docs/connect-docs/server/#client-registration) property to be less restrictive.

The body of a client registration request may contain any of the metadata values specified in [OpenID Connect Dynamic Client Registration 1.0](http://openid.net/specs/openid-connect-registration-1_0.html), [Section 2. Client Metadata](http://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata). Values not understood by the server are ignored. Only a valid `redirect_uris` value is required.

#### Anonymous Request

```bash
$ curl -X POST https://connect.example.com/register
       -H 'Content-Type: application/json'
       -d '{
            "client_name": "Triangular Pretzel",
            "redirect_uris": ["https://app.example.com/callback"]
          }'
```



#### Authorized Request

```bash
$ curl -X POST https://connect.example.com/register
       -H 'Content-Type: application/json'
       -H 'Authorization: Bearer <TOKEN>'
       -d '{
            "client_name": "Triangular Pretzel",
            "redirect_uris": ["https://app.example.com/callback"]
          }'
```

#### Response

In addition to client metadata, the registration response contains a `registration_client_uri` specifying the configuration endpoint for this client, and a `registration_access_token` which must be used as a bearer token to read and update the client configuration at that endpoint.

```bash
HTTP/1.1 201 Created
cache-control: no-store
pragma: no-cache
content-type: application/json

{
  "client_id":"e090a523-2ea8-43c8-b012b58ae60ce25d",
  "client_name":"Example",
  "application_type":"web",
  "redirect_uris":["https://app.example.com/callback"],
  "token_endpoint_auth_method": "client_secret_basic",
  "client_secret": "3ba1dbffa4402b7a6a76"
  "client_id_issued_at":1433960696883,
  "registration_client_uri":"https://connect.example.com/register/e090a523-2ea8-43c8-b012-b58ae60ce25d",
  "registration_access_token":"eyJhbGciOiJSUzI1NiJ9.eyJ***2fQ.UmN***kE0"
}
```

### REST API

### Registration permissions

Anvil Connect can be configured for three types of client registration: `dynamic`, `token`, or `scoped`, each being more restrictive than the previous option. The default `client_registration` type is `scoped`. Trusted clients require additional scope to register. This can be configured with the `trusted_registration_scope` setting, which defaults to `realm`.

#### Dynamic Client Registration

With `client_registration` set to `dynamic`, any party can register a client with the authorization server. Optionally, a bearer token may be provided in the authorization header per RFC6750. If a valid access token is presented with a registration request, the client will be associated with the user represented by that token.

```json
{
  // ...
  "client_registration": "dynamic",
  // ...
}
```

The following table indicates expected responses to *Dynamic Client Registration* requests.

| trusted | w/token | w/scope | response  |
|:-------:|:-------:|:-------:|----------:|
|         |         |         | 201       |
| x       |         |         | 403       |
|         | x       |         | 201       |
| x       | x       |         | 403       |
| x       | x       | x       | 201       |
|         | x       | x       | 201       |


#### Token-restricted Registration

Client registration can be restricted so that a valid user access token is required by setting `client_registration` to `token`. In this case, any request without a token will fail. As with *Dynamic Client Registration*, in order to register a trusted client, the access token must have sufficient scope.

```json
{
  // ...
  "client_registration": "token",
  // ...
}
```

| trusted | w/token | w/scope | response  |
|:-------:|:-------:|:-------:|----------:|
|         |         |         | 403       |
| x       |         |         | 403       |
|         | x       |         | 201       |
| x       | x       |         | 403       |
| x       | x       | x       | 201       |
|         | x       | x       | 201       |

#### Scoped Registration

Third party registration can be restricted altogether with the `scoped` `client_registration` setting. In this case, all registration requires a prescribed `registration_scope`.

```json
{
  // ...
  "client_registration": "scoped",
  // ...
}
```

| trusted | w/token | w/scope | response  |
|:-------:|:-------:|:-------:|----------:|
|         |         |         | 403       |
| x       |         |         | 403       |
|         | x       |         | 403       |
| x       | x       |         | 403       |
| x       | x       | x       | 201       |
|         | x       | x       | 201       |

## Client Properties

#### redirect_uris

<span class="label label-danger">Required</span>

Array of URIs that can be used by the server for redirecting users back to the client app after authentication. The `redirect_uri` parameter in an auth request must match one of these values exactly.

```
"redirect_uris": [
  "https://app.example.com/callback",
  "https://app.example.com/callback.html"
]
```

#### application_type

Three kinds of clients can be registered with Anvil Connect. The client type is determined by setting the `application_type` property of a client. Values can include `web`, `native`, and `service`.

```
"application_type": "web"
```

#### client_name

Name of the client that may be presented to the user during authentication.

```
"client_name": "Shiny App"
```

#### logo_uri

URI of an image that may be presented to the user during authentication.

```
"logo_uri": "http://example.com/image.jpg"
```

#### client_uri

URI of the application or information about the application.

```
"client_uri": "http://app.example.com"
```

#### response_types

Array of allowed response types. Valid values are `code`, `token`, `id_token`, and combinations of the three. `none` is also a valid option, but cannot be in a set with other response types, i.e. `[ "none", "code" ]` is valid but `[ "none code" ]` is not. Defaults to `[ "code" ]`. _[See response types for more details.](#response-types)_

```
"response_types": [ "code" ]
```

#### grant_types

Array of allowed grant types. Valid values are `authorization_code`, `implicit`, and `refresh_token`. Defaults to `[ "authorization_code" ]`. _[See grant types for more details.](#grant-types)_

```
"grant_types": [ "authorization_code" ]
```

#### default_max_age

Integer representing time in seconds before tokens will expire by default. This can be overridden by the `max_age` authorization parameter.

```
"default_max_age": 36000
```

#### post_logout_redirect_uris

Array of URIs that can be used by the server for redirecting users back to the client app after logout. The `post_logout_redirect_uri` parameter in an logout request must match one of these values exactly.

```
"post_logout_redirect_uris": [
  "https://app.example.com"
]
```

#### trusted

The string `true` indicates a client is part of your security realm. Any other value (including no value) indicates the client should be treated as a third party.

```
"trusted": true
```

#### default_client_scope

Array of strings used to specify scope requested for client access tokens issued by a `client_credentials` grant.

```
"default_client_scope": [
  "profile",
  "realm"
]
```

#### scopes

Array of strings. The `scopes` setting determines which scopes are required to access an app.

```
"scopes": [
  "photos",
  "activity"
]
```



## Permissions

### Scopes

When you want to require a user to have permission to access an app, you can define a `scopes` setting for the client. If this property contains any scopes, the user must be authorized to access those scopes via roles.

### RBAC

Clients can be assigned roles which grant the client permission to access scopes associated with the role. This is useful for API keys (two-legged OAuth).

You can assign a role to a client using the CLI.

```bash
# -c indicates the assignment should be made to a client
$ nv assign -c <CLIENT_ID> <ROLE>
```

## Client Libraries

#### JavaScript

* [JavaScript](https://github.com/anvilresearch/connect-js)
* [AngularJS](https://github.com/anvilresearch/connect-js)

#### Node.js

* [Node.js](https://github.com/anvilresearch/connect-nodejs)
* [Passport OpenID Connect Strategy](https://github.com/jaredhanson/passport-openidconnect)

#### PHP

* [OpenID Connect PHP](https://github.com/jumbojett/OpenID-Connect-PHP)

<!--
#### Java

#### Ruby

#### Python

#### ...
-->

## Supported Software

* [Drupal](https://www.drupal.org/project/openid_connect)
* [Wordpress](https://github.com/jumbojett/Wordpress-OpenID-Connect-Login)

<!--
#### Ghost
#### Nodebb
-->
