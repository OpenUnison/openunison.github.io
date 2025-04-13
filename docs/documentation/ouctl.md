# ouctl

The `ouctl` utility simplifies the deployment of OpenUnison's helm charts. It also makes it much easier to onboard new clusters into a multicluster environment.  When using this utility, you do not need to create a `Secret`.  It is available on Windows, Linux, and macOS.

## Downloads

* [Linux](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.13-linux)
* [Windows](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.13-win.exe)
* [MacOS](https://nexus.tremolo.io/repository/ouctl/ouctl-0.0.13-macos)

Rename the downloaded file to `ouctl` (or `ouctl.exe` on windows).

## Homebrew

On both macOS and Linux, you can use [homebrew](https://brew.sh/) to install the `ouctl` utility:

```
brew install openunison/ouctl/ouctl
```

## kubectl plugin

The `ouctl` utility can be installed as a kubectl plugin using krew as a self-hosted plugin (the krew project does not accept installation utilities).  This method supports Linux, macOS, and Windows.

```
kubectl krew install --manifest-url=https://nexus.tremolo.io/repository/ouctl/ouctl.yaml
```


[Return to deployment](../../deployauth#site-specific-configuration)
