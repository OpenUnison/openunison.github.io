# Kubernetes Security Token Service

***REQUIRES OPENUNISON 1.0.43+***

Since Kubernetes 1.24 the default configuration has provided every running `Pod` with a unique, short lived identity.  There are two ways this identity can be leveraged by external services:

1. Use a `TokenReviewRequest` to check if the identity is still valid
2. Use the cluster's issuer and OIDC discovery document to get the public key needed to validate a `Pod`'s `ServiceAccount` token like any other JWT.

This lets you use your `Pod`'s identity to access external services.  There's a couple of catches though:

1. To use a `TokenReviewRequest`, your external service needs its own `ServiceAccount` token, which means that your external service now likely needs its own static, long lived token.  This breaks a real basic rule of Kubernetes security, don't use `ServiceAccount` tokens from outside the cluster.
2. If you just use JWT validation, you could be open to using tokens that are no longer associated with a running `Pod`, even though the token its self hasn't expired.  Also, if you rely on this method you're going to need to make sure your cluster's issuer is an accessible URL.  Finally, re-keying is a cluster-wide operation.

OpenUnison provides a Security Token Service that allows you to easily provide your `Pod`s with tokens ready to use with external services.  OpenUnison does the work of keeping this token updated with a lightweight sidecar.  You can connect to as many endpoints with your STS as you want, and you can also have separate keys for each one.  For instance, if you need to provide access to multiple AWS accounts, you can separate your STS' by keys, so that they can't be abused to access accounts they shouldn't be able to.

(need picture of an STS)

## Amazon Web Services

The AWS client SDks all know how to use JWT tokens that are scoped for AWS.  When the token expires, the client SDKs know to reload them.  Creating an STS for AWS requires a place to host your OIDC discovery documents that's accessible from the public internet.  Our example will use AWS CloudFront with an S3 bucket.  This gives us our issuer with a commercially signed certificate authority (CA).  Once we have our issuer created, we'll deploy our STS, copy our oidc discovery data into our S3 bucket, then create a trust with AWS via IAM.  Once that's all done, we can launch a `Pod` that will have access to AWS' services using our short lived tokens instead of a static identity!

### Create an Issuer on AWS CloudFront with S3

AWS' CloudFront is a content delivery service that provides an entry point to other services, such as web sites and applications, but gives you some added benefits like denial of service (DoS) protection and cacheing.  In our example, we're going to use it as a front-end for an S3 bucket we're going to use to store our oidc discovery documents in.  CloudFront will provide you with a default URL that doesn't have any identifiable information in it.  I like using the default because it's not identifiable!  Your security team will love it.

That said, you don't have to use CloudFront+S3.  If you have another way to publish the documents, that'll work too!

First, let's create our S3 bucket:

```sh
aws s3api create-bucket \
  --bucket openunison-sts-example \
  --region us-east-1 
```



Next, we'll need to create a CloudFront to access our S3 bucket.  THe first step is to generate an Origin Access Rule (OAC) that CloudFront will use:

```sh
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "oac-openunison-sts-1",
    "Description": "OAC for S3 origin",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' > /tmp/oac.json
export OAC_ID=$(jq -r '.OriginAccessControl.Id' < /tmp/oac.json)
echo $OAC_ID
```

Update the below JSON to create your CloudFront distribution.  Update appropriately for your S3 bucket.  Also, make sure to set `.Origins.Items[0].OriginAccessControlId` to the value of `OAC_ID` from the command you just generated.



```json
{
  "CallerReference": "ou-sts-001",
  "Comment": "My ou sts 001",
  "Enabled": true,
  "Origins": {
    "Items": [
      {
        "Id": "S3-openunison-sts-docs",
        "DomainName": "openunison-sts-docs.s3.us-east-1.amazonaws.com",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        },
        "OriginAccessControlId": "E...."
      }
    ],
    "Quantity": 1
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-openunison-sts-docs",
    "ViewerProtocolPolicy": "redirect-to-https",
    "TrustedSigners": {
      "Enabled": false,
      "Quantity": 0
    },
    "ForwardedValues": {
      "QueryString": false,
      "Cookies": { "Forward": "none" }
    },
    "MinTTL": 0
  }
}
```

Save this file and then create your cloudfront distribution:

```sh
aws cloudfront create-distribution --distribution-config file:///path/to/cf-config.json > /tmp/cf-cfg-output.json
export DIST_ARN=$(jq -r '.Distribution.ARN' < /tmp/cf-cfg-output.json)
echo "ARN:  $DIST_ARN"
export DIST_DOMAIN_NAME=$(jq -r '.Distribution.DomainName' < /tmp/cf-cfg-output.json)
echo "Domain: $DIST_DOMAIN_NAME"
```

Use the ARN for your distribution to update the policy for your bucket, so that CloudFormation can access it:

```json
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::openunison-sts-docs/*",
            "Condition": {
                "ArnLike": {
                    "AWS:SourceArn": "arn:aws:cloudfront::XXXXXX:distribution/E1BKYHE1TT8Z10"
                }
            }
        }
    ]
}
```

Also, create a cache invalidation so that if you rotate your certs, it won't be missed by AWS:

```sh
aws cloudfront create-invalidation --distribution-id E1BKYHE1TT8Z10 --paths "/certs"
```

Finally apply the bucket policy:

```sh
aws s3api put-bucket-policy \
  --bucket openunison-sts-docs \
  --policy file:///tmp/buacket.json
```

### Deploying the STS

First, deploy OpenUnison into your cluster.  If you aren't planning on using OpenUnison for user authentication, you can make updates to deploy just the base system and use the STS charts.  Next, add the below yaml to your openunison's values.yaml for your STS:

```yaml
sts:
  endpoints:
  - name: aws
    audience: sts.aws.com
    # if you want to have multiple STS in a single OpenUnison
    # change this to a different path
    path: /aws
    # authorize service accounts to access the STS
    # Available attributes:
    # namespace - the name of the ServiceAccount namespace
    # saname - The name of the ServiceAccount
    # cluster - The name of your cluster
    azRules:
    - scope: filter
      constraint: "(namespace=myns)"
    issuer:
      # this is the domain of your CloudFront deployment
      host: "d1dexjaad4ian0.cloudfront.net"
      # The keypair to use for your STS
      # See https://openunison.github.io/knowledgebase/certificates/#how-do-i-include-additional-keys-and-certificates for how to generate a distinct
      # keypair
      keypair: "unison-saml2-rp-sig"
    injector:
      # the label and annotation to use to identify if the Pod
      # should use the admission controller
      label: aws-role
      # The environment variable that will point to our JWT
      token_environment_variable_name: "AWS_WEB_IDENTITY_TOKEN_FILE"
      # The environment variable used to store the value of our annotation, 
      # for AWS the value is the role to assume
      label_value_environment_variable_name: "AWS_ROLE_ARN"
      # if your OpenUnison uses locally signed cert, uncomment and
      # create a ConfigMap in the openunison namespace called ouca with the
      # a key called ca.crt with OpenUnison's certificate chain
      # explicit_certificate_trust: true
```

Once you've updated your values.yaml, deploy the sts chart:

```sh
helm install openunison-sts tremolo/openunison-kube-sts -n openunison -f /path/to/values.yaml
```

After the chart is deployed, we're able to get the issuer docs for upload to S3 by using the host from our OpenUnison values.yaml and prefix from our sts values:

```sh
export S3DIR=$(mktemp -d)
mkdir $S3DIR/.well-known
curl https://k8sou.awsstsnew.tremolo.dev/aws/issuer-docs | jq -r '.discovery' > $S3DIR/.well-known/openid-configuration
curl https://k8sou.awsstsnew.tremolo.dev/aws/issuer-docs | jq -r '.keys' | jq -r > $S3DIR/certs
```

This will generate an OIDC discovery document and a keys document.  Next, we'll upload to our S3 bucket:

```sh
cd $S3DIR
aws s3 sync .  s3://openunison-sts-docs/ --exact-timestamps
```

if everything is setup correctly, you should be able to pull both the discovery and keys document from your issuer URL:

```sh
curl https://d30f9e2jcv4sq2.cloudfront.net/.well-known/openid-configuration 
{
  "issuer": "https://d30f9e2jcv4sq2.cloudfront.net",
  "authorization_endpoint": "https://d30f9e2jcv4sq2.cloudfront.net/auth",
  "token_endpoint": "https://d30f9e2jcv4sq2.cloudfront.net/token",
  "userinfo_endpoint": "https://d30f9e2jcv4sq2.cloudfront.net/userinfo",
  "revocation_endpoint": "https://d30f9e2jcv4sq2.cloudfront.net/revoke",
  "jwks_uri": "https://d30f9e2jcv4sq2.cloudfront.net/certs",
  "response_types_supported": [
    "code",
    "token",
    "id_token",
    "code token",
    "code id_token",
    "token id_token",
    "code token id_token",
    "none"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "scopes_supported": [
    "openid",
    "email",
    "profile"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_post"
  ],
  "claims_supported": [
    "sub",
    "aud",
    "iss",
    "exp",
    "sub",
    "cluster",
    "namespace",
    "saname"
  ],
  "code_challenge_methods_supported": [
    "plain",
    "S256"
  ]
}
```

So far we've deployed our STS and have published our discovery document.  Next we'll need to configure AWS to trust the JWTs that OpenUnison generates.  

### Configuring AWS IAM

Thus far, you have deployed the OpenUnison STS and published the issuer documents to make it so that AWS can trust our tokens.  The next step is to add a trust to our AWS IAM configuration and to associate that trust with a role.  First, create your identity provider:

```sh
aws iam create-open-id-connect-provider --url "https://d30f9e2jcv4sq2.cloudfront.net" --client-id-list sts.aws.com > /tmp/idp.json
export IDP_ARN=$(jq -r '.OpenIDConnectProviderArn' < /tmp/idp.json)
echo "IdP ARN: $IDP_ARN"
```

Make sure to update the URL with the URL of your distribution.  Next, use the IdP ARN to create a trust policy that will allow users to assume a role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::XXXXXXXXXXXX:oidc-provider/d30f9e2jcv4sq2.cloudfront.net" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "d30f9e2jcv4sq2.cloudfront.net:aud": "sts.aws.com"
      }
    }
  }]
}
```

Create the role from the trust document:
```sh
aws iam create-role --role-name openunison-sts-docs --assume-role-policy-document file:///tmp/trust.json
```

Finally, attach a policy to the role.  I created a simple policy to let me read an S3 bucket from my `Pod`:

```sh
aws iam attach-role-policy --role-name openunison-sts-docs --policy-arn 'arn:aws:iam::XXXXXXXXX:policy/unit-test-s3-read-only'
```

At this point AWS has been configured to trust OpenUnison, the last step is to configure Kubernetes to use OpenUnison as an admission controller for some `Pod`s.

### Injecting Tokens into Your Workloads

With the prep work of deploying our STS and configuring AWS, the last step is to deploy the STS' admission controller configuration and annotate our workloads.  In your values.yaml, you'll need to update the OpenUnison operator to tell it to keep our mutating webhook admission controller configuration's certificate up to date.  This is done by adding the following to your values.yaml:

```yaml
operator:
  mutators:
   - injector-ENDPOINT_NAME
```

In our case, `ENDPOINT_NAME` is `aws`, from `sts.endpoints[0].name` in our values.yaml.

Assuming you're using the [ouctl](/documentation/ouctl), add two options to your command:

```sh
ouctl install-auth-portal  -u openunison-sts-webhooks=tremolo/openunison-kube-sts-pre -r openunison-sts=tremolo/openunison-kube-sts  ~/values.yaml
```

The `-u` will deploy the configuration for our webhook before the operator, while the `-r` will deploy our sts.  Once `ouctl` is done running, you'll be able to deploy a workload.  The injector works on all `Pod` objects with the label `tremolo.io/LABEL` where `LABEL` is the value from `sts.endpoints[*].injector.label`.  In our case that's `aws-role`.  We created a simple `Deployment` with a `Pod` that includes this label:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: aws-test-python-simple
  name: aws-test-python-simple
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: aws-test-python-simple
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        tremolo.io/aws-role: arn:aws:iam::XXXXXXXXXXXX:role/ou-sts-oidc
      creationTimestamp: null
      labels:
        app: aws-test-python-simple
        tremolo.io/aws-role: "true"
    spec:
      containers:
      - env:
        - name: AWS_REGION
          value: us-east-1
        image: harbor.qalab.tremolo.dev/lab/aws-test-python
        imagePullPolicy: Always
        name: aws-cli
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

When the `Pod` is launched, you'll see that there are two containers.  In addition to the workload container, there's a side car that is deployed to generate and maintain your token.  You'll also see that your container has additional environment variables to support the AWS integration.

Now that we're working with short lived tokens for our interactions with AWS, next we'll make sure we know how to rotate our private key.

### Rotating Signature Keys

It's good security practice to rotate your signature keys on a regular basis.  The easiest way to do this is to delete the `Secret` that stores the key and redeploy OpenUnison.  This will trigger the operator to generate a new key.  Next, you'll need to upload the newly generated OIDC discovery documents:

```sh
export S3DIR=$(mktemp -d)
mkdir $S3DIR/.well-known
curl https://k8sou.awsstsnew.tremolo.dev/aws/issuer-docs | jq -r '.discovery' > $S3DIR/.well-known/openid-configuration
curl https://k8sou.awsstsnew.tremolo.dev/aws/issuer-docs | jq -r '.keys' | jq -r > $S3DIR/certs
```

This will generate an OIDC discovery document and a keys document.  Next, we'll upload to our S3 bucket:

```sh
cd $S3DIR
aws s3 sync .  s3://openunison-sts-docs/ --exact-timestamps
aws cloudfront create-invalidation --distribution-id EXXXXXXXXXXXX --paths "/certs"
```

The last command is important because it tells CloudFront to stop caching our signing verification certificate.  This way when AWS checks if there's a new signing certificate, it will get the new one and your workloads can continue to run.