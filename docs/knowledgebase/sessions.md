# Session Management

The default implementation of OpenUnison uses one minute long sessions with refresh tokens that are good for fifteen minutes.  This means, that as you work with your Kubernetes CLI or other tools the client-go SDK (or any other SDK that you use), is constantly refreshing your `id_token`.  This is done for security, because the `id_token` is a bearer token, which means that anyone who has the token can use it.  Unlike mTLS, where the "secret" doesn't travel over the wire, a bearer token must be sent with every request.  This means that any system that receives this token, which can include anything from CNIs to webhooks, could leak the token and be a vector to breach your cluster.  By having such short lived tokens, by the time an attacker gets the token and recognizes it the token is useless.

A key to this scheme is that the refresh token, which is used to validate that you should receive a new token when the current one has expired, never is included in communication with the Kubernetes API and is one time use only.  This means that once you use a refresh token, it should no longer be valid.

Unfortunately, there is a known issue in the client-go SDK that if you have multiple tools working off of the same kubectl configuration file, then the refreshs can overlap causing one or all your tools to lose their session.  That's because if the client-go SDK fails to refresh the token, all the tools running on that configuration are broken and you need to get a new kubectl configuration file.  There are a few ways to manage this issue.

## Individual Configurations Per Tool 

The most secure way to manage this issue is to use a single configuration file per tool.  For instance, if you are going to run k9s:

```bash
$ export KUBECONFIG=$(mktemp)
$ kubectl oulogin --host=openunison.mydomain.dev
$ k9s
```

Then, in another window run kubectl:

```bash
$ export KUBECONFIG=$(mktemp)
$ kubectl oulogin --host=openunison.mydomain.dev
$ kubectl get nodes
```

Since these two tools are working off of their own kubectl configuration, they can't interfere with eachother.

## Refresh Token Grace Period

OpenUnison supports a grace period for refresh tokens.  This allows a refresh token to be used for a short time after it's already been used once.  A short grace period, such as 5-10 seconds, is usually enough to account for the issues the client-go SDK has with refresh tokens.  ***BEWARE: THIS FEATURE WILL OPEN YOU TO POTENTIAL REPLAY ATTACKS!!! BE VERY CAREFUL WITH THIS FUNCTION!*** If this feature is dangerous, why did we build it?  Because while it can open your cluster to a replay attack, it's less risky then a longer lived token.  To help mitigate this risk, we added additional logging when an expired refresh token is used within the grace period:

```
[2022-12-24 00:34:56,424][XNIO-1 task-1] INFO  OpenIDConnectIdP - [ExpiredRefreshTokenUsed] - https://k8sou.lab.tremolo.dev/auth/idp/k8sIdp/token - uid=mmosley,ou=shadow,o=Tremolo - Expired token used within grace period [10.1.67.56] - [f9b20cfb111e9d922865823e502ddb0585c7b1bb2]
```

Your SIEM should look for this log line and look for potential issues such as an unusual source IP address.  To enable this feature, add the following configuration to your values.yaml and redeploy:

```
openunison:
  authentication:
    refresh_token:
      grace_period_millis: 5000
```

Once redeployed, used refresh tokens can be re-used for up to 5 seconds.