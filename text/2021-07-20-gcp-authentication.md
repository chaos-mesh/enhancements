# GCP Authentication

## Summary

There has been a strong interest in providing a more native way to integrate
with the public cloud authentication system. However, nearly every cloud
provider has their own way of doing this. `OIDC` is a standard way and the
`OIDC` is also an auth provider shipped with kubernetes. However, it is not 
supported by all cloud providers.

In this RFC, we will outline the steps required to integrate with GCP. Other
cloud providers are also in consideration, but hasn't been investigated yet.

## Motivation

The users have to generate a `ServiceAccount` and use its token to access chaos
dashboard. However, it's not a good practice for end user, as the
`ServiceAccount` should be used by a service, but not a human. It would be more
friendly to support `User` and the end-user authentication.

## Detailed Design

The final target is to get a `Token` to access the kubernetes according to
different `KUBECONFIG`, and the user may have a lot of different ways to enter
information. In this RFC, we will support the following methods:

1. Upload KUBECONFIG file
2. Gcloud OAuth2


### Upload KUBECONFIG file

This method sounds fine, but cannot work for most cases, as the config itself
may depend on other files. For example:

```yaml
- name: k8s-1.14
  user:
    client-certificate: /home/USER/.minikube/profiles/k8s-1.14/client.crt
    client-key: /home/USER/.minikube/profiles/k8s-1.14/client.key
```

Then the user should also provide the certificate and key. Handling the
different form of KUBECONFIG is a bit complex. The kubernetes dashboard doesn't
handle this situation:

```go
if len(info.Token) == 0 && (len(info.Password) == 0 || len(info.Username) == 0) {
    return api.AuthInfo{}, errors.NewInvalid("Not enough data to create auth info structure.")
}
```

It only accepts `Token` or `Username` and `Password` as the auth info, and
cannot use `cert` and `key` to create the `AuthInfo`.

However, it suprisingly works for GCP. The normal GCP `KUBECONFIG` format is:

```yaml
- name: gke_sodium-replica-308911_asia-east2_chunibyo
  user:
    auth-provider:
      config:
        access-token: xxxxxxx
        cmd-args: config config-helper --format=json
        cmd-path: /usr/local/bin/gcloud
        expiry: "2021-07-20T08:10:33Z"
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

The `.user.auth-provider[0].config.access-token` will be copied to the
`info.Token` authomatically. Though the server may fail to find `cmd-path`, it
can also use the token to access the cluster (but it cannot refresh).

### Gcloud OAuth2

To access the GKE api-server directly, the user may need to get an access token
and the CA. With the help of `oauth2` package and
`cloud.google.com/go/container/apiv1`, we can use `ClusterManagerClient` to get
the CA. And the access token should be provided by asking user to authenticate
through OAuth2.

The following actions should be done by the user:

1. Enter projects, locations and clusters.
2. Login in the pop-up google login window.

Then it will redirect to the dashboard server automatically. After that, the
server can read access token and refresh token from the payload. A normal OAuth2 
configuration could be like:

```go
oauth = oauth2.Config{
    ClientID:     os.Getenv("CLIENT_ID"),
    ClientSecret: os.Getenv("CLIENT_SECRET"),
    RedirectURL:  os.Getenv("REDIRECT_URI"),
    Scopes: []string{
        "email", "profile",
        "https://www.googleapis.com/auth/userinfo.email",
        "https://www.googleapis.com/auth/compute",
        "https://www.googleapis.com/auth/cloud-platform",
    },
    Endpoint: google.Endpoint,
}
```

When the user first enters the dashboard, he should be redirected to
`oauth.AuthCodeURL("")`, and login to the google account. After login, it will
automatically redirect to the `REDIRECT_URI` (which is the dashboard server with
specific URI) with the `code` as the payload.

Then the server could get the token with `code` and the `oauth` configuration:

```go
oauth2Token, err := oauth.Exchange(context.TODO(), r.URL.Query().Get("code"))
```

The `AccessToken` and `RefreshToken` fields in the `oauth2Token` can be used to
access the cluster.
