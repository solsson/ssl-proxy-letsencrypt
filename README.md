### ssl-proxy-letsencrypt

Note: this fork was the inception of what is now maintained at https://github.com/Reposoft/docker-httpd-letsencrypt.

Based on https://github.com/GoogleCloudPlatform/nginx-ssl-proxy
and inspired by http://blog.ployst.com/development/2015/12/22/letsencrypt-on-kubernetes.html
but
the proxy container itself requests a cert from https://letsencrypt.org/ upon startup. No need to run kubectl from within the pod.

Schedule restart of the container/pod within 90 days to renew before cert expiry.

A service could look like this:
```
---
kind: Service
apiVersion: v1
metadata:
  name: ssl-proxy-service
  labels:
    role: ssl-proxy
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
  - name: https
    port: 443
    targetPort: https
    protocol: TCP
  selector:
    role: ssl-proxy
  type: LoadBalancer
```

And the proxy pod like this:
```
---
kind: ReplicationController
apiVersion: v1
metadata:
  name: ssl-proxy-letsencrypt
  labels:
    role: ssl-proxy
spec:
  replicas: 1
  selector:
    role: ssl-proxy
  template:
    metadata:
      name: ssl-proxy-letsencrypt
      labels:
        role: ssl-proxy
    spec:
      containers:
      - name: ssl-proxy-letsencrypt
        image: solsson/ssl-proxy-letsencrypt:latest
        env:
        - name: TARGET_SERVICE
          value: my-actual-service:80
        - name: ENABLE_SSL
          value: 'true'
        - name: cert_email
          value: webmaster@example.net
        - name: cert_domains
          value: my.example.net my2.example.net
        # remove this when it's time to get a real cert
        - name: LETSENCRYPT_ENDPOINT
          value: https://acme-staging.api.letsencrypt.org/directory
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```

Make sure to create the k8s service before the pod, so letsencrypt validation can get through on startup.

### nginx-ssl-proxy

#nginx-ssl-proxy
This repository is used to build a Docker image that acts as an HTTP [reverse proxy](http://en.wikipedia.org/wiki/Reverse_proxy) with optional (but strongly encouraged) support for acting as an [SSL termination proxy](http://en.wikipedia.org/wiki/SSL_termination_proxy). The proxy can also be configured to enforce [HTTP basic access authentication](http://en.wikipedia.org/wiki/Basic_access_authentication). Nginx is the HTTP server, and its SSL configuration is included (and may be modified to suit your needs) at `nginx/proxy_ssl.conf` in this repository.

## Building the Image
Build the image yourself by cloning this repository then running:

```shell
docker build -t nginx-ssl-proxy .
```

## Using with Kubernetes
This image is optimized for use in a Kubernetes cluster to provide SSL termination for other services in the cluster. It should be deployed as a [Kubernetes replication controller](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/replication-controller.md) with a [service and public load balancer](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md) in front of it. SSL certificates, keys, and other secrets are managed via the [Kubernetes Secrets API](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/design/secrets.md).

Here's how the replication controller and service would function terminating SSL for Jenkins in a Kubernetes cluster:

![](img/architecture.png)

See [https://github.com/GoogleCloudPlatform/kube-jenkins-imager](https://github.com/GoogleCloudPlatform/kube-jenkins-imager) for a complete tutorial that uses the `nginx-ssl-proxy` in Kubernetes.

## Run an SSL Termination Proxy from the CLI
To run an SSL termination proxy you must have an existing SSL certificate and key. These instructions assume they are stored at /path/to/secrets/ and named `cert.crt` and `key.pem`. You'll need to change those values based on your actual file path and names.

1. **Create a DHE Param**

    The nginx SSL configuration for this image also requires that you generate your own DHE parameter. It's easy and takes just a few minutes to complete:

    ```shell
    openssl dhparam -out /path/to/secrets/dhparam.pem 2048
    ```

2. **Launch a Container**

    Modify the below command to include the actual address or host name you want to proxy to, as well as the correct /path/to/secrets for your certificate, key, and dhparam:

    ```shell
    docker run \
      -e ENABLE_SSL=true \
      -e TARGET_SERVICE=THE_ADDRESS_OR_HOST_YOU_ARE_PROXYING_TO \
      -v /path/to/secrets/cert.crt:/etc/secrets/proxycert \
      -v /path/to/secrets/key.pem:/etc/secrets/proxykey \
      -v /path/to/secrets/dhparam.pem:/etc/secrets/dhparam \
      nginx-ssl-proxy
    ```
    The really important thing here is that you map in your cert to `/etc/secrets/proxycert`, your key to `/etc/secrets/proxykey`, and your dhparam to `/etc/secrets/dhparam` as shown in the command above.

3. **Enable Basic Access Authentication**

    Create an htpaddwd file:

    ```shell
    htpasswd -nb YOUR_USERNAME SUPER_SECRET_PASSWORD > /path/to/secrets/htpasswd
    ```

    Launch the container, enabling the feature and mapping in the htpasswd file:

    ```shell
    docker run \
      -e ENABLE_SSL=true \
      -e ENABLE_BASIC_AUTH=true \
      -e TARGET_SERVICE=THE_ADDRESS_OR_HOST_YOU_ARE_PROXYING_TO \
      -v /path/to/secrets/cert.crt:/etc/secrets/proxycert \
      -v /path/to/secrets/key.pem:/etc/secrets/proxykey \
      -v /path/to/secrets/dhparam.pem:/etc/secrets/dhparam \
      -v /path/to/secrets/htpasswd:/etc/secrets/htpasswd \
      nginx-ssl-proxy
    ```

4. **Enabling frames**

    The container does by default not allow any frames to be opened due to
    security issues doing this. While this is a recommended default behavior,
    it might break certain applications.
    
    If you are encountering issues you can either turn of the frame denial
    completely by passing the `ENABLE_FRAMES` environment variable to it:
    
    ```shell
    docker run \
      -e ENABLE_SSL=true \
      -e ENABLE_FRAMES=true \
      -e TARGET_SERVICE=THE_ADDRESS_OR_HOST_YOU_ARE_PROXYING_TO \
      -v /path/to/secrets/cert.crt:/etc/secrets/proxycert \
      -v /path/to/secrets/key.pem:/etc/secrets/proxykey \
      -v /path/to/secrets/dhparam.pem:/etc/secrets/dhparam \
      -v /path/to/secrets/htpasswd:/etc/secrets/htpasswd \
      nginx-ssl-proxy
    ```
    
    Alternatively if you only need frames from your own domain you can also
    use the [SAMEORIGIN](https://developer.mozilla.org/en-US/docs/Web/HTTP/X-Frame-Options) policy
    through setting the `ENABLE_FRAMES_SAMEORIGIN` variable to true.
    
    ```shell
    docker run \
      -e ENABLE_SSL=true \
      -e ENABLE_FRAMES_SAMEORIGIN=true \
      -e TARGET_SERVICE=THE_ADDRESS_OR_HOST_YOU_ARE_PROXYING_TO \
      -v /path/to/secrets/cert.crt:/etc/secrets/proxycert \
      -v /path/to/secrets/key.pem:/etc/secrets/proxykey \
      -v /path/to/secrets/dhparam.pem:/etc/secrets/dhparam \
      -v /path/to/secrets/htpasswd:/etc/secrets/htpasswd \
      nginx-ssl-proxy
    ```
