# EntraID / AzureAD

EntraID (formerly known as AzureAD or Azure ActiveDirectory) is Microsoft Azure's identity system.  While some of the concepts are the same as ActiveDirectory, they're generally very different.  While ActiveDirectory has an LDAP interface, EntraID is entirely HTTPS based.  OpenUnison's target for EntraID provides:

* Authentication token management and refresh
* Paging for large search results
* Translation from an email address to a User Principal Name

## Configuring EntraID

The first step is to define a new **App registration** in your EntraID configuration.  Use a name that is descriptive. For the **Redirect URI** use **Web** and specify the URL for OpenUnison's redirect uri.  It will be `https://OU_HOST/auth/oidc` where `OU_HOST` is the value from `network.openunison_host` in your values.yaml.

![EntraID Application Registration](/assets/images/identity-providers/azuread/1.png)

Once your application is registered, the next step is to create a client secret.  Click on **Add a certificate or secret** in the upper left hand section of the screen.

![Create Client Secret](/assets/images/identity-providers/azuread/2.png)

Click on **+ New client secret**, give it a description and a life span.  How long the secret is valid is based on your requirements and compliance needs.  Once you click **Add**, the secret will become available for you to copy.  Copy it, and store it in a file for when you're ready to deploy the portal.

Once you have the client secret, store it in a `Secret` in the openunison namespace.  To configure your EntraID target, you'll also need your tenant id and your client (application) id. Finally, configure your [target](/documentation/reference/targets/#azuread)

