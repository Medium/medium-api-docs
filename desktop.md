Developing a desktop integration for Medium
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

Your application should include some setting or dialog that allows a user to enter an accessToken. If your application supports multiple profiles, each profile will need a different accessToken.

We have tried to make the process as straightforward as possible. The standard OAuth2 authentication flow can be complicated to implement for desktop apps, so we have created the ability for users to self-issue accessTokens for this purpose.

The user can acquire this token from the Settings page of their Medium account:
https://medium.com/me/settings

You should instruct your user to visit this URL and generate an accessToken for your application from the “External Account / Tokens” section.

This UX affordance to issue these tokens is not currently implemented, but we can issue accessTokens to you in the interim so you can begin testing.

An accessToken is a consecutive string of hex digits, like this:

```
181d415f34379af07b2c11d144dfbe35d
```

Treat any accessToken as though it is a password, and store it securely.

Self-issued accessTokens do not expire, though they may be revoked by the user.


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

| Field      | Type   | Description                                     |
| -----------|--------|-------------------------------------------------|
| id         | string | A unique identifier for the user.               |
| username   | string | The user's username on Medium.                  |
| name       | string | The user's name on Medium.                      |
| url        | string | The URL to the user's profile on Medium         |
| imageUrl   | string | The URL to the user's avatar on Medium          |

Possible errors:

| Error code           | Description                                     |
| ---------------------|-------------------------------------------------|
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

| Field         | Type        | Description                                     |
| --------------|-------------|-------------------------------------------------|
| id            | string      | A unique identifier for the post.               |
| title         | string      | The post's title                                |
| authorId      | string      | The userId of the post's author                 |
| tags          | string[]    | The post's tags                                 |
| url           | string      | The URL of the post on Medium                   |
| canonicalUrl  | string      | The canonical URL of the post. If canonicalUrl was not specified in the creation of the post, this field will not be present.  |
| publishStatus | string      | The publish status of the post.                 |
| publishedAt   | timestamp   | The post's published date. If created as a draft, this field will not be present.                                              |
| license       | enum        | The license of the post.                        |
| licenseUrl    | string      | The URL to the license of the post.             |

Possible errors:

| Error code           | Description                                         |
| ---------------------|-----------------------------------------------------|
| 400 Bad Request      | Required fields were invalid, not specified.        |
| 401 Unauthorized     | The access token is invalid or has been revoked.    |
| 403 Forbidden        | The user does not have permission to publish.       |
| 404 Not Found        | `authorId` described an invalid user.               |

5. Images
---------

Medium will automatically side-load any images specified by the src attribute on an <img> tag in the post content. However, if you have local image files that you wish to send, you may use the images endpoint.

Unlike other API endpoints, this requires multipart form-encoded data.

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

The field name must be “image”. All lines in the body must be terminated with `\r\n`. Only one image may be sent per request. The following image content types are supported:

* `image/jpeg`
* `image/png`
* `image/gif`
* `image/tiff`

Animated gifs are supported.

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
| md5           | string      | An MD5 hash of the iamge data.                  |

You may choose to persist the md5 and url of uploaded images in a local store, so that you can quickly determine in future whether an image needs to be uploaded to Medium, or if an existing URL can be reused.


6. Sandbox
----------

We do not have a sandbox environment yet, sorry. To test, please feel free to create a testing account. *We recommend you do this by registering using an email address rather than Facebook or Twitter, as registering with the latter two automatically creates follower relationships on Medium between your connections on those networks.*

These endpoints will perform actions on production data on `medium.com`. Please test with care.


7. Editing a post
-----------------

There is no current support for editing an existing post. When this exists, it will live at:

```
PUT https://api.medium.com/v1/posts/{{id}}
```

Where `id` is the post identifier. A timeframe for this endpoint is not yet specified.
