# Nginx-Authentik-OIDC-SSO
A setup for a private server with dynamic IP / reverse proxy, and SSO with Authentik or OpenID (OIDC)
## Nginx
## Authentik
### Synology SSO Server as OpenID Provider (OP) in Authentik
For setting up the SSO Server in Synology DSM, see [_Synology's KB - SSO Server_](https://kb.synology.com/en-us/DSM/help/SSOServer/sso_server_desc) or, as an example, [_How do I use Synology SSO Server to set up OIDC SSO for DSM?_](https://kb.synology.com/en-us/DSM/tutorial/set_up_oidc_for_dsm_in_sso_server).

For setting up Synology as an OP in Authentik, refer to the [Authentik Documentation](https://docs.goauthentik.io/docs/users-sources/sources/protocols/oauth/#openid-connect-authentik-20226) or:
1. Login and switch to the administration interface.
2. Go to _Directory > Federation & Social Login_.
3. Create a new authentication source of type _OpenID OAuth_ with, e.g., the following parameters:
    - **Name**: `DSM`
    - **Slug**: `dsm`
    - **Enabled**: `true`
    - **User Matching Mode**:
      - `Link to a user with identical email address` or
      - `Link to a user with identical username` (slightly less secure because of missing e-mail verification option)
    - **Group Matching Mode**: `Link to a group with identical name` or as required
    - **User Path**: `goauthentik.io/sources/%(slug)s`
    - **Protocol Settings**
      - **Consumer key/secret**: The client ID/secret from Synology's SSO application
      - **Scopes**: Leave empty unless required
    - **URL Settings**: Filled automatically once saved
    - **Flow Settings**: Leave as is
### Add Synology SSO Server as new OpenID Provider to Authentik Login Page
See [Federated and Social Sources](https://docs.goauthentik.io/docs/users-sources/sources/).
