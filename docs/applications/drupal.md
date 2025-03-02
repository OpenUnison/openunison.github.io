# Drupal



Drupal is a widely deployed web content management system often used for building complex web sites and applications.  OpenUnison can provide SSO as well as create/manage users via the Drupal RESTful API.

## SSO Access to Drupal

There are multiple plugins to support OIDC and SAML.  Any of them will with with OpenUnison.  See 


While Drupal provides a RESTful API, it doesn't provide a way to retrieve a user's id based on their email address.  See [Custom SSO](/documentation/custom-sso/) for examples of how to setup SSO with applications.

## Automating Onboarding

You can externalize access management to Drupal by using OpenUnison to manage Drupal's users and groups.  This makes it easier to centralize access without having to manually add users to the drupal database.  To support integration with Drupal, use the [Drupal8 Target](/documentation/reference/targets/#drupal8target) to connect to Drupal via the RESTful API services.  To enable, you'll need to create a user with administrative privileges.  Then go to Drupal's Manage --> Extend and enable **HTTP Basic Authentication**, **Serialization**, and **RESTful Web Services**:

![Enable Extensions](/assets/images/drupal-extensions.png)

Next, deploy Tremolo Security's get user id by email address from https://nexus.tremolo.io/repository/drupal/get_user_by_email.tar.gz.

Finally, install and enable the [REST UI Plugin](https://www.drupal.org/project/restui).  Once enabled Enable the User and Get User ID by EMail APIs:

