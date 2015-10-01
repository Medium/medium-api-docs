Developing a web integration for Medium
===========================================

This document describes how to use the Medium API to create posts on Medium on behalf of a Medium account.

1. Overview
-----------

Medium’s API is a JSON-based OAuth2 API. All requests are made to endpoints beginning:
`https://api.medium.com/v1`

All requests must be secure, i.e. `https`, not `http`.

2. Authentication
-----------------

In order to publish on behalf of a Medium account, you will need an access token. An access token grants limited access to a user’s account.

We will supply you with a `clientId` and and a `clientSecret` with which you may access Medium’s API. These identify your account and should not be used for other integrations. The `clientSecret` should be treated like a password and stored securely.

We have implemented a standard OAuth2 flow. The first step is to acquire a short term authorization code by sending the user to our authorization URL so they can grant access to your integration.

```
https://medium.com/m/oauth/authorize?client_id={{clientId}}
    &scope=basicProfile,publishPost
    &state={{state}}
    &response_type=code
    &redirect_uri={{redirectUri}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `client_id`     | string   | required   | The clientId we will supply you that identifies your integration. |
| `scope`         | string   | required   | The access that your application is requesting, comma separated. Currently, there are only two valid scope values, and you should request both, thus: ‘basicProfile,publishPost’ |
| `state`         | string   | required   | Arbitrary text of your choosing, which we will repeat back to you to help you prevent request forgery. |
| `response_type` | string   | required   | The field currently has only one valid value, and should be `code`.  |
| `redirect_uri`  | string   | required   | The URL where we will send the user after they have completed the login dialog. This field should be URL encoded.  |

If the user grants your request for access, we will send them back to the specified `redirect_uri` with a state and code parameter:

```
https://example.com/callback/medium?state={{state}}
    &code={{code}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `state`         | string   | required   | The state you specified in the request.         |
| `code`          | string   | required   | A short-lived authorization code that may be exchanged for an access token. |

If the user declines access, we will send them back to the specified `redirect_uri` with an error parameter:

```
https://example.com/callback/medium?error=access_denied
```

Once you have an authorization code, you may exchange it for a long-lived access token with which you can make authenticated requests on behalf of the user. To acquire an access token, make a form-encoded server-side POST request:

```
POST /v1/tokens HTTP/1.1
Host: api.medium.com
Content-Type: application/x-www-form-urlencoded
Accept: application/json
Accept-Charset: utf-8

code={{code}}&client_id={{client_id}}&client_secret={{client_secret}}&grant_type=authorization_code&redirect_uri={{redirect_uri}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `code`          | string   | required   | The authorization code you received in the previous step. |
| `client_id`     | string   | required   | Your integration's `clientId` |
| `client_secret` | string   | required   | Your integration's `clientSecret` |
| `grant_type`    | string   | required   | The literal string "authorization_code" |
| `redirect_uri`  | string   | required   | The same redirect_uri you specified when requesting an authorization code. |

If successful, you will receive back an access token response:

```
HTTP/1.1 201 OK
Content-Type: application/json; charset=utf-8
{
 "token_type": "Bearer",
 "access_token": {{access_token}},
 "refresh_token": {{refresh_token}}
}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `token_type`    | string   | required   | The literal string "Bearer"                     |
| `access_token`  | string   | required   | A token that is valid for 60 days and may be used to perform authenticated requests on behalf of the user. |
| `refresh_token` | string   | required   | A token that does not expire which may be used to acquire a new `access_token`.                            |

Each access token is valid for 60 days. When an access token expires, you may request a new token using the refresh token. Refresh tokens do not expire. Both access tokens and refresh tokens may be revoked by the user at any time. You should treat both access tokens and refresh tokens like passwords and store them securely.

Both access tokens and refresh tokens are consecutive strings of hex digits, like this:

```
181d415f34379af07b2c11d144dfbe35d
```

To acquire a new access token using a refresh token, make the following form-encoded request:

```
POST /v1/tokens HTTP/1.1
Host: api.medium.com
Content-Type: application/x-www-form-urlencoded
Accept: application/json
Accept-Charset: utf-8

refresh_token={{refresh_token}}&client_id={{client_id}}
&client_secret={{client_secret}}&grant_type=refresh_token
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `refresh_token` | string   | required   | A valid refresh token.                          |
| `client_id`     | string   | required   | Your integration's `clientId`                   |
| `client_secret` | string   | required   | Your integration's `clientSecret`               |
| `grant_type`    | string   | required   | The literal string "refresh_token"              |


3. User information
-------------------

The first request you make should be to acquire user details. This will confirm that your accessToken is valid, and give you a user id that you will need for subsequent requests. Make an authenticated GET request to:

```
https://api.medium.com/v1/me
```

To make an authenticated request, send your accessToken in the Authorization header.

Example request:

```
GET /v1/me HTTP/1.1
Host: api.medium.com
Authorization: Bearer 181d415f34379af07b2c11d144dfbe35d
Content-Type: application/json
Accept: application/json
Accept-Charset: utf-8
```

The response is a User object within a data envelope.

Example response:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
{
 "data": {
   "id": "5303d74c64f66366f00cb9b2a94f3251bf5",
   "username": "majelbstoat",
   "name": "Jamie Talbot",
   "url": "https://medium.com/@majelbstoat",
   "imageUrl": "https://images.medium.com/0*fkfQiTzT7TlUGGyI.png"
 }
}
```

Where a User object is:

| id         | string | A unique identifier for the user.               |
| username   | string | The user's username on Medium.                  |
| name       | string | The user's name on Medium.                      |
| url        | string | The URL to the user's profile on Medium         |
| imageUrl   | string | The URL to the user's avatar on Medium          |

Possible errors:

| 401 Unauthorized     | The `accessToken` is invalid or has been revoked. |


4. Creating a post
------------------

To create a post, make an authenticated POST request to:

```
https://api.medium.com/v1/users/{{authorId}}/posts
```

Where `authorId` is the user identifier you acquired in Step 2.

Example request:

```
POST /v1/users/5303d74c64f66366f00cb9b2a94f3251bf5/posts HTTP/1.1
Host: api.medium.com
Authorization: Bearer 181d415f34379af07b2c11d144dfbe35d
Content-Type: application/json
Accept: application/json
Accept-Charset: utf-8
{
 "title": "Liverpool FC",
 "content": "<p>The most successful team in history.</p>",
 "tags": ["football", "sport", "Liverpool"],
 "canonicalUrl": "http://jamietalbot.com/posts/liverpool-fc",
 "contentFormat": "html"
}
```

With the following fields:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| title           | string   | required   | The title of the post. Note that this title is used for SEO and when rendering the post as a listing, but will not appear in the actual post—for that it must be specified in the `content` field as well. |
| content         | string   | required   | The body of the post, in a valid, semantic, HTML fragment. Further markups may be supported in the future. For a full list of accepted tags, see [here](https://medium.com/@katie/a4367010924e).                |
| contentFormat   | string   | required   | The format of the "content" field. There is currently only one valid value, "html".                  |
| tags            | string[] | optional   | Tags to classify the post. Only the first three will be used.                                        |
| canonicalUrl    | string   | optional   | The original home of this content, if it was originally published elsewhere.                         |
| publishStatus   | enum     | optional   | The status of the post. Valid values are “public”, “draft”, or “unlisted”. The default is “public”.  |
| license         | enum     | optional   | The license of the post. Valid values are “all-rights-reserved”, “cc-40-by”, “cc-40-by-sa”, “cc-40-by-nd”, “cc-40-by-nc”, “cc-40-by-nc-nd”, “cc-40-by-nc-sa”, “cc-40-zero”, “public-domain”. The default is “all-rights-reserved”. |

The response is a Post object within a data envelope. Example response:

```
HTTP/1.1 201 OK
Content-Type: application/json; charset=utf-8
{
 "data": {
   "id": "e6f36a",
   "title": "Liverpool FC",
   "authorId": "5303d74c64f66366f00cb9b2a94f3251bf5",
   "tags": ["football", "sport", "Liverpool"],
   "url": "https://medium.com/@majelbstoat/liverpool-fc-e6f36a",
   "canonicalUrl": "http://jamietalbot.com/posts/liverpool-fc",
   "publishStatus": "public",
   "publishedAt": 1442286338435,
   "license": "all-rights-reserved",
   "licenseUrl": "https://medium.com/policy/9db0094a1e0f"
 }
}
```

Where a Post object is:
