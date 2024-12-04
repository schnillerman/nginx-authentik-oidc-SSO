# Reverse Proxy and SSO on Docker for a Private Server with Dynamic DNS
A setup for a private server with dynamic IP / reverse proxy, and SSO with Authentik or OpenID (OIDC)
## TL;DR
### Current Plan
An extensive conversation with ChatGPT results in the following setup:
- **Reverse Proxy**: Traefik (streamlined configuration in `traefik.yml` which can be referenced by each `docker-compose.yml`)
- **SSL**: Letsencrypt in Traefik
- **Authentication**:
  - via traefik-forward-auth
  - Synology SSO Server as OpenID Provider
 
#### ChatGPT-Based Recommendations
- [Setup 1: Traefik with traefik-forward-auth and OpenID-based SSO](https://github.com/schnillerman/reverse-proxy-SSO-docker/blob/main/Setup%201%20-%20Traefik,%20forward-auth,%20OpenID%20SSO.md).

### Nginx Reverse "Plug-Ins" mentioned by the web & ChatGPT
|**Criterion**            |**Vouch Proxy**                              |**oauth2-proxy**                                                                               |
|-------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------|
|**Complexity**           |Simpler, fewer options                       |More configuration options but more complex                                                    |
|**OpenID Support**       |Excellent, focused directly on OpenID Connect|Supports OIDC but has a broader focus                                                          |
|**Resource Consumption** |Lower (minimalist approach)                  |- Higher (comprehensive feature set)<br>- one oauth2-proxy container per OAuth2/OpenID Provider|
|**Flexibility**          |Good, but limited in very complex scenarios  |Excellent, suitable for many scenarios                                                         |
|**Community and Support**|Smaller community                            |Larger community, better documentation                                                         |
Both Vouch and oauth2-proxy require more or less extensive configuration in the Nginx proxy host advanced settings which can be prone to inconsistencies.
Further alternatives considered:
- Caddy

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
### Setup Authentik as Authentication Provider for Nginx Proxy Hosts
Here's some more info: https://geekscircuit.com/set-up-authentik-sso-with-nginx-proxy-manager/#create-new-provider

For now, _domain level forward authentication_ in Authentik doesn't seem to work as expected[^1][^2].

## Additional Sources
- https://www.reddit.com/r/navidrome/comments/r8834t/reverse_proxy_authentication_with_authentik/
- https://www.reddit.com/r/navidrome/comments/oa8gkz/guide_how_to_use_a_sso_solution_in_front_of/
- https://github.com/vouch/vouch-proxy

[^1]: https://www.reddit.com/r/Authentik/comments/1460g3z/domain_level_forward_auth_problem/
[^2]: https://github.com/goauthentik/authentik/discussions/2780
