# Themes and Colors

*These features require OpenUnison 1.0.42*

You can customize OpenUnison's portal to make it line up better with your organization and needs.  In addition to [replacing Tremolo Security's logos](../custom_logos/), you can customize the colors, headers, and if certain pages are shown.  



## Custom Colors


Colors can be set in your `values.yaml`.  For The `openunison.html.theme.colors` section lets you set the color scheme for OpenUnison.  For instance, setting the bellow config will replace the red header with a brighter blue:

```yaml
openunison:
  html:
    theme:
      colors:
        primary: 
          main: "#004b8d"
          dark: "#1c4467"
          light: "#0067c2"
        secondary:
          main: "#16aca0"
          dark: "#0f7870"
          light: "#44bcb3"
        error: "#ff1744"
```

If you've already deployed OpenUnison, you only need to update the `orchestra-login-portal` chart.  There's no need to restart anything.  A quick refresh will get your new colors loaded:

![OpenUnison with custom colors](/assets/images/themes/colors.png)

## Custom Header

You can change the header for the portal from ***OpenUnison*** to anything else by updating `openunison.html.theme.headerTitle`.  For instance, the below configuration will change the header text from ***OpenUnison*** to ***My Company's Kubernetes Access Portal***:

```yaml
openunison:
  html:
    theme:
      headerTitle: "My Company's Kubernetes Access Portal"
```

If you've already deployed OpenUnison, you only need to update the `orchestra-login-portal` chart.  There's no need to restart anything.  A quick refresh will get your new colors loaded:

![OpenUnison with a custom header](/assets/images/themes/header.png)

## Customizing Namespace as a Service

The out-of-the-box OpenUnison portal is built on the assumption that you want to provide a self service portal for managing access to your portals.  If that's not the case, there are some customizations that can stream line what is available to your users.  For instance, if you're using NaaS with external groups only, you won't want to show the ***Request Access*** screen.  If you're integrated with OpenShift, you may not want to show the links screen.

### Hiding Pages

You can hide pages by adding them to the `openunison.html.theme.hidePages` list.  Options and use cases:

| Page to hide | Use Case |
| ------------- | -------- |
| `request-access` | If all authorizations are based on your identity providers groups, or you don't want users to request access on their own |
| `front-page` | If you don't want to use OpenUnison for SSO, just for access management |
| `ops` | If you don't want admins to be able to search for user permissions |

For example, the below yaml will hide the ***Request Access*** page in the portal:

```yaml
openunison:
  html:
    theme:
      hidePages:
      - request-access
```

If you've already deployed OpenUnison, you only need to update the `orchestra-login-portal` chart.  There's no need to restart anything.  A quick refresh will get your new colors loaded:

![OpenUnison with no "Request Access" page](/assets/images/themes/hide_screens.png)

## Choosing a Starting Page

If you're not using OpenUnison for SSO, you may want to start users on a different screen then the links page.  For instance, to start users on the ***Request Access*** page you would set `openunison.html.theme.startPage` to `request-access`:

```yaml
openunison:
  html:
    theme:
      startPage: request-access
```

You can configure the `startPage` options with:

| Page to hide | Use Case |
| ------------- | -------- |
| `request-access` | If all authorizations are based on your identity providers groups, or you don't want users to request access on their own |
| `front-page` | If you don't want to use OpenUnison for SSO, just for access management |
| `ops` | If you don't want admins to be able to search for user permissions |

If you've already deployed OpenUnison, you only need to update the `orchestra-login-portal` chart.  There's no need to restart anything.  A quick refresh will get your new colors loaded: