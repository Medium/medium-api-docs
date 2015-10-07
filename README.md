# Medium’s API documentation

This repository contains the documentation for [Medium](https://medium.com)’s API.

#### Contents

- [Overview](#1-overview)
- [Authentication](#2-authentication)
  - [Browser-based authentication](#21-browser-based-authentication)
  - [Self-issued access tokens](#22-self-issued-access-tokens)
- [Resources](#3-resources)
  - [Users](#31-users)
  - [Posts](#32-posts)
  - [Images](#33-images)
- [Testing](#4-testing)
- [SDKs](#5-sdks)

## 1. Overview

Medium’s API is a JSON-based OAuth2 API. All requests are made to endpoints beginning:
`https://api.medium.com/v1`

All requests must be secure, i.e. `https`, not `http`.

#### Developer agreement

By using Medium’s API, you agree to our [terms of service](https://medium.com/@feerst/2b405a832a2f).

## 2. Authentication

In order to publish on behalf of a Medium account, you will need an access token. An access token grants limited access to a user’s account. We offer two ways to acquire an access token: browser-based OAuth authentication, and self-issued access tokens.

Unless there is a compelling reason, you should use browser-based authentication.

### 2.1 Browser-based authentication

We will supply you with a `clientId` and and a `clientSecret` with which you may access Medium’s API. Each integration should have its own `clientId` and `clientSecret`. The `clientSecret` should be treated like a password and stored securely.

The first step is to acquire a short term authorization code by sending the user to our authorization URL so they can grant access to your integration.

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
| `scope`         | string   | required   | The access that your integration is requesting, comma separated. Currently, there are three valid scope values, which are listed below. Most integrations should request `basicProfile` and `publishPost` |
| `state`         | string   | required   | Arbitrary text of your choosing, which we will repeat back to you to help you prevent request forgery. |
| `response_type` | string   | required   | The field currently has only one valid value, and should be `code`.  |
| `redirect_uri`  | string   | required   | The URL where we will send the user after they have completed the login dialog. This field should be URL encoded.  |

The following scope values are valid:

| Scope         | Description                                                             | Extended |
| ------------- | ----------------------------------------------------------------------- | -------- |
| basicProfile  | Grants basic access to a user’s profile (not including their email).    | No       |
| publishPost   | Grants the ability to publish a post to the user’s profile.             | No       |
| uploadImage   | Grants the ability to upload an image for use within a Medium post.     | Yes      |

Integrations are not permitted to request extended scope from users without explicit prior permission from Medium. Attempting to request these permissions through the standard user authentication flow will result in an error if extended scope has not been authorized for an integration.

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
| `client_id`     | string   | required   | Your integration’s `clientId` |
| `client_secret` | string   | required   | Your integration’s `clientSecret` |
| `grant_type`    | string   | required   | The literal string "authorization_code" |
| `redirect_uri`  | string   | required   | The same redirect_uri you specified when requesting an authorization code. |

If successful, you will receive back an access token response:

```
HTTP/1.1 201 OK
Content-Type: application/json; charset=utf-8
{
 "token_type": "Bearer",
 "access_token": {{access_token}},
 "refresh_token": {{refresh_token}},
 "scope": {{scope}},
 "expires_at": {{expires_at}}
}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `token_type`    | string   | required   | The literal string "Bearer"                     |
| `access_token`  | string   | required   | A token that is valid for 60 days and may be used to perform authenticated requests on behalf of the user. |
| `refresh_token` | string   | required   | A token that does not expire which may be used to acquire a new `access_token`.                            |
| `scope`         | string[] | required   | The scopes granted to your integration.         |
| `expires_at`    | int64    | required   | The timestamp in unix time when the access token will expire |

Each access token is valid for 60 days. When an access token expires, you may request a new token using the refresh token. Refresh tokens do not expire. Both access tokens and refresh tokens may be revoked by the user at any time. **You must treat both access tokens and refresh tokens like passwords and store them securely.**

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
| `client_id`     | string   | required   | Your integration’s `clientId`                   |
| `client_secret` | string   | required   | Your integration’s `clientSecret`               |
| `grant_type`    | string   | required   | The literal string "refresh_token"              |


###  2.2 Self-issued access tokens

Self-issued access tokens (described in user-facing copy as integration tokens) are explicitly designed for desktop integrations where implementing browser-based authentication is non-trivial, or software like plugins where it is impossible to secure a client secret. You should not request that a user give you an integration token if you don’t meet these criteria. Users will be cautioned within Medium to treat integration tokens like passwords, and dissuaded from making them generally available.

Users can generate an access token from the [Settings page](https://medium.com/me/settings) of their Medium account.

You should instruct your user to visit this URL and generate an integration token from the `Integration Tokens` section. You should suggest a description for this
token - typically the name of your product or feature - and use it consistently for all users.

Self-issued access tokens currently grant the `basicProfile` and `publishPost` scope. A future iteration of the API will require a user to select the scope they wish to grant access to.

Self-issued access tokens do not expire, though they may be revoked by the user at any time.

## 3. Resources

The API is RESTful and arranged around resources. All requests must be made with an integration token. All requests must be made using `https`.

Typically, the first request you make should be to acquire user details. This will confirm that your access token is valid, and give you a user id that you will need for subsequent requests.

### 3.1 Users

#### Getting the authenticated user’s details
Returns details of the user who has granted permission to the application.

```
GET https://api.medium.com/v1/me
```

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

| Field      | Type   | Description                                     |
| -----------|--------|-------------------------------------------------|
| id         | string | A unique identifier for the user.               |
| username   | string | The user’s username on Medium.                  |
| name       | string | The user’s name on Medium.                      |
| url        | string | The URL to the user’s profile on Medium         |
| imageUrl   | string | The URL to the user’s avatar on Medium          |

Possible errors:

| Error code           | Description                                     |
| ---------------------|-------------------------------------------------|
| 401 Unauthorized     | The `accessToken` is invalid or has been revoked. |


### 3.2 Posts

#### Creating a post
Creates a post on the authenticated user’s profile.

```
POST https://api.medium.com/v1/users/{{authorId}}/posts
```

Where authorId is the user id of the authenticated user.

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
 "contentFormat": "html",
 "content": "<h1>Liverpool FC</h1><p>You’ll never walk alone.</p>",
 "tags": ["football", "sport", "Liverpool"],
 "publishStatus": "public"
}
```

With the following fields:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| title           | string   | required   | The title of the post. Note that this title is used for SEO and when rendering the post as a listing, but will not appear in the actual post—for that it must be specified in the `content` field as well. |
| contentFormat   | string   | required   | The format of the "content" field. There are two valid values, "html", and "markdown" |
| content         | string   | required   | The body of the post, in a valid, semantic, HTML fragment, or Markdown. Further markups may be supported in the future. For a full list of accepted HTML tags, see [here](https://medium.com/@katie/a4367010924e).                |
| tags            | string[] | optional   | Tags to classify the post. Only the first three will be used. Tags longer than 25 characters will be ignored.                                        |
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

| Field         | Type        | Description                                     |
| --------------|-------------|-------------------------------------------------|
| id            | string      | A unique identifier for the post.               |
| title         | string      | The post’s title                                |
| authorId      | string      | The userId of the post’s author                 |
| tags          | string[]    | The post’s tags                                 |
| url           | string      | The URL of the post on Medium                   |
| canonicalUrl  | string      | The canonical URL of the post. If canonicalUrl was not specified in the creation of the post, this field will not be present.  |
| publishStatus | string      | The publish status of the post.                 |
| publishedAt   | timestamp   | The post’s published date. If created as a draft, this field will not be present.                                              |
| license       | enum        | The license of the post.                        |
| licenseUrl    | string      | The URL to the license of the post.             |

Possible errors:

| Error code           | Description                                         |
| ---------------------|-----------------------------------------------------|
| 400 Bad Request      | Required fields were invalid, not specified.        |
| 401 Unauthorized     | The access token is invalid or has been revoked.    |
| 403 Forbidden        | The user does not have permission to publish.       |
| 404 Not Found        | `authorId` described an invalid user.               |

### 3.3 Images

#### Uploading an image

Most integrations will not need to use this resource. **Medium will automatically side-load any images specified by the src attribute on an <img> tag in post content when creating a post.** However, if you are building a desktop integration and have local image files that you wish to send, you may use the images endpoint.

Unlike other API endpoints, this requires multipart form-encoded data.

```
POST https://api.medium.com//v1/images
```

Example request:

```
POST /v1/images HTTP/1.1
Host: api.medium.com
Authorization: Bearer 181d415f34379af07b2c11d144dfbe35d
Content-Type: multipart/form-data; boundary=FormBoundaryXYZ
Accept: application/json
Accept-Charset: utf-8

--FormBoundaryXYZ
Content-Disposition: form-data; name="image"; filename="filename.png"
Content-Type: image/png

IMAGE_DATA
--FormBoundaryXYZ--
```

The field name must be `image`. All lines in the body must be terminated with `\r\n`. Only one image may be sent per request. The following image content types are supported:

* `image/jpeg`
* `image/png`
* `image/gif`
* `image/tiff`

Animated gifs are supported. Use your power for good.

The response is an Image object within a data envelope. Example response:

```
HTTP/1.1 201 OK
Content-Type: application/json; charset=utf-8
{
 "data": {
   "url": "https://images.medium.com/0*fkfQiTzT7TlUGGyI.png",
   "md5": "fkfQiTzT7TlUGGyI"
 }
}
```

Where an Image object is:

| Field         | Type        | Description                                     |
| --------------|-------------|-------------------------------------------------|
| url           | string      | The URL of the image.                           |
| md5           | string      | An MD5 hash of the image data.                  |

You may choose to persist the md5 and url of uploaded images in a local store, so that you can quickly determine in future whether an image needs to be uploaded to Medium, or if an existing URL can be reused.


## 4 Testing

We do not have a sandbox environment yet. To test, please feel free to create a testing account. *We recommend you do this by registering using an email address rather than Facebook or Twitter, as registering with the latter two automatically creates follower relationships on Medium between your connections on those networks.*

These endpoints will perform actions on production data on `medium.com`. **Please test with care.**

## 5 SDKs

- [Medium SDK for Go](https://github.com/Medium/medium-sdk-go)
- [Medium SDK for Python](https://github.com/Medium/medium-sdk-python)
