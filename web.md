Developing a web integration for Medium
===========================================

This document describes how to use the Medium API to create posts on Medium on behalf of a Medium account.

Overview
--------

Medium’s API is a JSON-based OAuth2 API. All requests are made to endpoints beginning:
`https://api.medium.com/v1`

All requests must be secure, i.e. `https`, not `http`.

Authentication
--------------

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

| Parameter     | Type     | Required?  | Description                                     |
| ------------- |----------|------------|-------------------------------------------------|
| `client_id`   | string   | required   | The clientId we will supply you that identifies your integration. |
| `scope`       | string   | required   | The access that your application is requesting, comma separated. Currently, there are only two valid scope values, and you should request both, thus: ‘basicProfile,publishPost’ |
