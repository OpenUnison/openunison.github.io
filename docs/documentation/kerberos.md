# Kerberos Support

It's common for enterprise deployments to require the use of Kerberos for authenticating to internal resources.  This is because Kerberos doesn't require a system's password to be used "over the wire", and often doesn't require a password at all.  OpenUnison is capable of using Kerberos either with a keytab file or a password.  Once configured a sidecar container is added to the operator, orchestra, and amq containers to ensure that OpenUnison can communicate with any resources that know how to work with a Kerberos ticket.  This document will step through configuring an OpenUnison Namespace-as-a-Service (NaaS) deployment to use Kerberos when communicating with SQL Server.

First, make sure your nodes are configured to use your Kerberos system's DNS service.  This is the number one reason why Kerberos implementations fail to work and the errors received from your domain controller are generally cryptic.  

Once your nodes' DNS is setup, you will next need either a keytab file or password.  A keytab file is essentially a password stored in a file, so take care to keep it secret.  First, create your `openunison` namespace.  Next, create the `Secret` that will store your credentials:

***If using a keytab***

Copy the keytab into an empty directory and create a `Secret`:

```sh
kubectl create secret generic kerb-keytab -n openunison --from-file=.
``` 

***If using a password***

After creating a file named `password` in an empty directory:

```bash
kubectl create secret generic kerb-keytab -n openunison --from-file=.
```

With your `Secret` created, next you need to create a `ConfigMap` that includes your Kerberos domain configuration.  If using SQL Server, you'll also need to add a key called `SQLJDBCDriver.conf` with:

```
SQLJDBCDriver {  
 com.sun.security.auth.module.Krb5LoginModule required useTicketCache=true;  
};
```

First, create an empty directory.  Copy your krb5.conf file into it and create the above `SQLJDBCDriver.conf` file, then run:

```
kubectl create configmap kerb-config -n openunison --from-file=.
```

Finally, update your values.yaml to include the kerberos configuration:

```yaml
kerberos:
  enabled: true
  keytab: false
  principal: unison-sql-server@ENT2K22.TREMOLO.DEV
  sidecar_image: docker.io/mlbiam/kerberos-sidecar:latest
```

In your SQL Server configuration in your values.yaml, add `;integratedSecurity=true;authenticationScheme=JavaKerberos` to your SQL Server JDBC URL.  The `user` needs to be specified, but is ignored.  The same is true of your JDBC Secret using the `ouctl` command.  