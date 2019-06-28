## Basic setup for an oAuth2 based SSO setup with the server site also acting as a client site:
- Server site has oAuth2 server installed and configured as per instructions
- Server site has openid_connect and openid_connect_sso installed and configured as per instructions
- Server site does NOT have openid_connect_sso_client installed
- Server site has sso.php and sso.config.php installed at the web root with the network array filled with the other network sites. The site where this is installed should be last in the array. https and strict cookies on.

- Client site haste openid_connect, openid_connect_sso, and openid_connect_sso_client installed and configured
- Client site has sso.php and sso.config.php installed at the web root with the network array filled with the other network sites. The site where this is installed should be last in the array. https and strict cookies on.

- At each site's configuration form (/admin/config/services/openid-connect-sso), the cookie domain is configured for the site domain and the SSO URL points to an sso script (full url) that is not it's own. SSO is enabled. On the client site the connect client is "generic" (form defaults are currently broken).
- The server site has an oAuth2 server configured with both sites as clients. More information in docs for that module
- The client and server site both have openID connect configured at (/admin/config/services/openid-connect) following that module.

## Important Notes on this D8-version:
0. This is not yet on parity w/ the D7 version of openid_connect_sso.
1. This is not working with caching. This should be fixable quickly, but for the time being add this snippet to the settings.php
2. There's no reason not to enable cookie_domain_strict in sso.config.php - this should probably be default
3. There's no reason not to enable https either, oAuth2 requires https anyways

`if (isset($_COOKIE['Drupal_visitor_SSOLogout']) || isset($_COOKIE['Drupal_visitor_SSOLogin'])) {
  $settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';
  $settings['cache']['bins']['render'] = 'cache.backend.null';
}`

And adjust your services.yml with this chunk:

`services:
  cache.backend.null:
    class: Drupal\Core\Cache\NullBackendFactory
`

3. sso.php script:
    - I have not tested the a.-Subdomain-based approach
    - I have split config from the sso.php script (s. sso/sso-config.php)
    - There's basic syslog logging.
    - I have adjusted the workings a little bit to allow for subpaths (e.g. test-a.dev/d8_openid_connect_sso_client/web/sso.php,test-b.dev/d8_openid_connect_sso_client/web/sso.php).
    - Therefore I have adjusted the workings to provide the full destination URL instead of the path on the origin_host - Yes, THERE IS AN OPEN REDIRECT ISSUE (see also in the D7 issue queue)!

4. Currently we're redirecting to <front> on all pages on the openid_connect_sso-client side (should be a relatively easy fix).
5. The settings form for the client_id (chosing which openid_connect-clients to use for authorization) is not working ATM.

Provides a single sign-on solution based on OpenID Connect.

The OpenID Connect server (central place of login) is a Drupal site running oauth2_server.
The clients are Drupal sites running openid_connect.

After the user's login on the server or logout on any of the network sites,
the module starts a redirect chain that visits the SSO script of each site in the network.
The SSO script then sets a cookie notifying the parent site of the pending login / logout.
When the user visits the actual site, the cookie is read, and the user logged in / out automatically.

This is the same approach used by Google Accounts.
The point of the redirects is to give each site a chance to set a cookie valid for its domain,
thus going around the same-origin policy that forbids a site from setting a cookie for another domain.
The redirects are fast and unnoticeable, since the SSO script is standalone (no Drupal bootstrap) and only sets the cookie.

See the documentation for more examples and setup instructions:
https://drupal.org/node/2274367
