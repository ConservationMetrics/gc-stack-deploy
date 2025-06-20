# Setting up an auth0 tenant for GuardianConnector

GuardianConnector uses [Auth0](https://auth0.com/) for authentication and user management. This guide outlines how to configure a new Auth0 tenant for production use.

## GCP OAuth client configuration

You will need a Google Cloud Platform (GCP) OAuth 2.0 Client in order to [avoid using development keys which is not recommended](https://community.auth0.com/t/confusing-dev-keys-error-message-when-using-production-keys/74273).

*  If you need to create a new Google Cloud Platform (GCP) OAuth 2.0 Client:
   * Create a project on GCP.
   * Navigage to "Clients", and create a Client for a web application.
   * Add the following settings:
     * Authorized JavaScript origins:
`https://{tenant}.us.auth0.com`
     * Authorized redirect URIs:
`https://{tenant}.us.auth0.com/login/callback`
 * If you already have a GCP OAuth 2.0 Client, then you can add the authorized JavaScript origin and redirect URI for your new tenant (per the formats above).
 * Copy down the Client ID and Secret for your client.

## Tenant configuration

1. In **Settings**, select “Production” as the Environment Tag.
2. In **Actions**, configure a Login Flow Action to handle user approval. (See [ User Approval Flow](#user-approval-flow) below.)
4. In **Authentication / Database**, ensure Sign Ups are enabled (enabled by default).
5. In **Authentication / Social**, enable google-oauth2 under Social Connections. You will need to provide a Client ID and Secret (see [GCP OAuth client configuration](#gcp-oauth-client-configuration)).

1.  In **Applications**, create a separate Regular Web Application for each tool (e.g., Superset, GC-Explorer). Add appropriate production domain values under Callback URLs, Web Origins, and CORS.
    * For **Superset** (assuming Superset is hosted at the root of your subdomain; otherwise, use the appropriate subdomain i.e. `{domain}-superset`):
      * **Callback URL**: `http://{domain}.guardianconnector.net/oauth-authorized/auth0`
      * **Allowed Web Origins**: `https://{domain}.guardianconnector.net/`
    * For **GC-Explorer**:
      * **Callback URL**: `http://{domain}-explorer.guardianconnector.net/login`
      * **Allowed Web Origins**: `http://{domain}-explorer.guardianconnector.net`
    * For **Windmill**:
      * **Callback URL**: `https://windmill.{domain}.guardianconnector.net/user/login_callback/auth0`
      * **Allowed Web Origins**: `https://windmill.demo.guardianconnector.net/`
2.  (Optional) in **Branding**, a few minor customizations like adding an organization logo and setting the background color to gray #F9F9F9 instead of standard black.

## Using Terraform

It is possible to use Terraform to automate much of the above process. Please see the [private `gc-forge` repo](https://github.com/ConservationMetrics/gc-forge/blob/main/terraform/modules/auth0-client/README.md) for more information.

## User approval flow

To restrict access until a user is approved, a Login Flow Action is used in Auth0. This action intercepts login attempts and denies access unless the user’s `app_metadata` includes `"approved": true`.


Action code (influenced by [the Common Use Cases in the auth0 documentation](https://auth0.com/docs/customize/actions/flows-and-triggers/login-flow#common-use-cases)):

```jsx
exports.onExecutePostLogin = async (event, api) => {
  // Check if the user is approved
  if (event.user.app_metadata && event.user.app_metadata.approved) {
    // User is approved, continue without action
  } else {
    api.access.deny('Your approval to access the app is pending.');
  }
};
```

This Action should be added to the **Post Login **flow in the Auth0 **Actions** section.

## auth0 approval process

1. A user signs up for one of the applications using Auth0, either by email/password or a third-party service (e.g., Google, GitHub).
2. If the user is not yet approved, they will encounter a message such as:
   * “Invalid login” (Superset)
   * “Your approval to access the app is pending” (GC-Explorer)
3. A tenant administrator approves the user:
   * Navigate to **User Management > Users**
   * Select the user
   * In the **App Metadata** section, add:
     ```json
     {
     "approved": true
     }
     ```
4. Once approved, the user can log in to GuardianConnector services.
5. For Superset, the user is initially assigned the **Gamma** role by default (controlled via the `USER_ROLE` environment variable).

    A Superset admin can then upgrade the user’s role or share specific dashboards.

TODO: Figure out a more user-friendly way to approve new users that doesn't require logging in to auth0 and editing App Metadata JSON.