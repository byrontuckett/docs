---
description: Introduction to Single Sign On (SSO) with Auth0.
toc: true
---
# What is Single Sign On?

Single Sign On (SSO) occurs when a user logs in to one application and is then signed in to other applications automatically, regardless of the platform, technology, or domain the user is using. The user signs in only one time hence the naming of the feature (Single Sign On).

Google's implementation of login for their products, such as Gmail, YouTube, Google Analytics, and so on, is an example of SSO. Any user that is logged in to one of Google's products are automatically logged in to their other products as well.

Single Sign On usually makes use of a **Central Service** which orchestrates the single sign on between multiple clients. In the example of Google, this central service is [Google Accounts](https://accounts.google.com). When a user first logs in, Google Accounts creates a cookie, which persists with the user as they navigate to other Google-owned services. The process flow is as follows:

1. The user accesses the first Google product.
1. The user receives a Google Accounts-generated cookie.
1. The user navigates to another Google product.
1. The user is redirected again to Google Accounts.
1. Google Accounts sees that the user already has an authentication-related cookie, so it redirects the user to the requested product.

## An overview of how SSO works with Auth0

In the case of SSO with Auth0, the **Central Service** is the Auth0 Authorization Server.

Let's look at an example of how the SSO flow looks when using Auth0 and the [Lock](/libraries/lock) widget and a user visits your application for the first time:

1. Your application will redirect the user to the Auth0 Hosted Login page where they can log in.
1. Auth0 will check to see whether there is an existing SSO cookie.
1. Because this is the first time the user visits this Hosted Login page, and no SSO cookie is present, they may be presented with username and password fields and also possibly some Social Identity Providers such as LinkedIn, GitHub, etc. (The exact layout of the Lock screen will depend on the [Identity Providers](/identityproviders) you have configured.

    ![](/media/articles/sso/single-sign-on/lock-no-sso-cookie.png)

1. Once the user has logged in, Auth0 will set an SSO cookie
1. Auth0 will also redirect back to your web application and will return an `id_token` containing the identity of the user.

Now let's look at flow when the user returns to your website for a subsequent visit:

1. Your application will redirect the user to the Auth0 Hosted Login page where they can sign in.
1. Auth0 will check to see whether there is an existing SSO cookie.
1. This time Auth0 finds an SSO cookie and instead of displaying the normal Lock screen with the username and password fields, it will display a Lock screen which indicates that we know who the user is, as they have already logged in before. They can simply confirm that they want to log in with that same account.

    ![](/media/articles/sso/single-sign-on/lock-sso-cookie.png)

1. Auth0 will update the SSO cookie if required
1. Auth0 will also redirect back to your web application and will return an `id_token` containing the identity of the user.

If an SSO cookie is present you can also sign the user in silently, i.e. without even displaying Lock so they can enter their credentials. This is covered in more detail in the next section.

The [Hosted Login Page](/hosted-pages/login) is the easiest and most secure way to implement SSO with Auth0. If, however, you need to use embedded Lock or an embedded custom authentication UI in your application, you can read here about safely conducting SSO with [cross-origin authentication](/cross-origin-authentication).

## How to Implement SSO with Auth0

::: note
Prior to enabling SSO for a given Client, you must first [configure the Identity Provider(s)](/identityproviders) you want to use.
:::

To enable SSO for one of your Clients (recall that each Client is independent of one another), navigate to the Clients section of the [Dashboard](${manage_url}/#/clients). Click on **Settings** (represented by the gear icon) for the Client with which you want to use SSO.

![](/media/articles/sso/single-sign-on/clients-dashboard.png)

Near the bottom of the **Settings** page, toggle **Use Auth0 instead of the IdP to do Single Sign On**.

![](/media/articles/sso/single-sign-on/sso-flag.png)

Alternatively you can also set the Client's SSO flag using the [Auth0 Management API](/api/management/v2#!/Clients/patch_clients_by_id).

### Checking the User's SSO Status from the Client

Whenever you need to determine the user's SSO status, you'll need to check the following:

* The Auth0 `accessToken`, which is used to access the desired resource
* The `expirationDate` on the `accessToken`, which is calculated using the `expires_in` response parameter after successful authentication on the part of the user

If you don't have a valid `accessToken`, the user is *not* logged in. However, they may be logged in via SSO to another associated application -- you can determine if this is the case or not by calling the `renewAuth` method of the auth0.js library, which will attempt to silently authenticate the user within an iframe. Whether the authentication is successful or not indicates whether the user has an active SSO cookie.

For more detailed information on how to implement this, please refer to [Client-Side SSO (Single Page Apps)](/sso/current/single-page-apps-sso).

::: note
The [Auth0 OIDC SSO Sample](https://github.com/auth0-samples/oidc-sso-sample) repo is an example of how to implement OIDC-compliant SSO.
:::

### Length of SSO Sessions

If the SSO flag is set for a Client, Auth0 will maintain an SSO session for any user authenticating via that Client. In the [Dashboard](${manage_url}/#/account/advanced) under Account settings (top right) and Advanced, there is an SSO **Session Timeout** setting. This setting determines how long the session will stay valid, and can be set to a custom value (in minutes). The default value is `10080` minutes (which equals to `7` days).

![](/media/articles/sso/single-sign-on/accountsettings-ssotimeout.png)

This is the session timeout for the Auth0 session. You can configure separately the timeouts used with tokens issued by Auth0, such as the OpenID Connect ID Token expiration claim or the SAML lifetime assertions. These are often used to drive the sessions on the applications (SAML Service Providers) themselves and are independent of the Auth0 (Identity Provider) session.

However, if there's no activity for **3 days**, the token expires. For example, if no web application on that user's machine (in the same browser) performed a login using the SSO session itself (such as using silent authentication), then the cookie would disappear after 3 days, even though a server side session might persist. That server session would essentially be unusable/orphaned, since you need a valid cookie to use it. Performing a new standard login would reset the whole SSO session.

The session inactivity duration is 3 days and is not configurable on the Public Cloud. PSaaS Appliance users, however, can control this account-level setting.

## What is Single Log Out?

Single Logout is the process where you terminate the session of each application or service where the user is logged in. To continue with the Google example, if you logout from Gmail, you get logged out also from YouTube, Google Analytics, etc.

There may be up to three different layers of sessions for a user with SSO.

* A session from an Identity Provider such as Google, Facebook or an enterprise SAML Identity Provider
* A session from Auth0 if the above SSO flag is turned on
* A session maintained by an application

See the [Logout URL docs](/logout) for information on terminating the first two sessions listed above.

## Using Social Identity Providers

Single Sign On works with Social Identity Providers given the following conditions:

1. You need to enable **Use Auth0 instead of the IdP to do Single Sign On** when configuring the Client
1. Your social connection can not be using the developer keys. You will need to register an app in the relevant social provider and then use that Client ID and Client Secret when configuring the connection. You can read more about the [caveats of using the Auth0 developer keys](/connections/social/devkeys#caveats).
