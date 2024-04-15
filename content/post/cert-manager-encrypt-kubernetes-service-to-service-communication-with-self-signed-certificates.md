+++
categories = ["Kubernetes", "Security", "cert-manager"]
date = 2023-06-23T23:41:23Z
description = ""
draft = false
image = "https://images.unsplash.com/photo-1533922829859-83c5b158a336?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=M3wxMTc3M3wwfDF8c2VhcmNofDh8fGVuY3J5cHRpb24lMjBpbiUyMG1vdGlvbnxlbnwwfHx8fDE2ODc1NjM2MTJ8MA&ixlib=rb-4.0.3&q=80&w=2000"
slug = "cert-manager-encrypt-kubernetes-service-to-service-communication-with-self-signed-certificates"
tags = ["Kubernetes", "Security", "cert-manager"]
title = "Cert-Manager: Encrypt Kubernetes service to service communication with self-signed certificates"

+++


Cert-manager is an open-source certificate management solution for Kubernetes clusters. It automates the issuance, renewal, and management of SSL/TLS certificates within a Kubernetes environment. Cert-manager is crucial in securing communication between services, encrypting HTTP traffic, and establishing secure connections to external systems.

Cert-manager integrates with popular certificate authorities (CAs) such as [Let's Encrypt](https://letsencrypt.org), [Venafi](https://venafi.com/?utm_adgroup=131519153019&gclid=CjwKCAjwhdWkBhBZEiwA1ibLmO96fec69F8BPVPLODTo-D68AsjnSZfHFE_HRF56FDxXzcxtFUM3WxoCMvQQAvD_BwE&utm_medium=cpc&utm_term=venafi&utm_content=hp&utm_source=gog&utm_campaign=15633869885), and [HashiCorp Vault](https://www.vaultproject.io/) to get certificates. It acts as a bridge between Kubernetes and these CAs, handling the certificate lifecycle management process.

I recently helped a customer create a self-signed CA. While researching, I found that there were no examples showing how to use self-signed private CA with cert-manager. There are plenty of examples about Let's Encrypt, Venafi, and [AWS ACM Private CA](https://aws.amazon.com/private-ca/). But none when you want to issue client certificates using a self-signed CA. This post shows how to create a private CA in cert-manager and generate client certificates using [cert-manager CSI driver](https://cert-manager.io/docs/usage/csi/).

## Setup cert-manager

Let's start by setting cert-manager in your Kubernetes cluster. We'll use Helm for installation. Cert-manager website includes alternate [installation methods](https://cert-manager.io/docs/installation/) if Helm doesn't work for you.  If you use Amazon EKS, there are EKS Blueprints to install cert-manager and cert-manager CSI driver ([Terraform](https://aws-ia.github.io/terraform-aws-eks-blueprints/add-ons/cert-manager/), [CDK](https://aws-quickstart.github.io/cdk-eks-blueprints/addons/cert-manager/)) .

Use Helm to install cert-manager and cert-manager CSI driver:

```
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true

helm upgrade -i \
  cert-manager-csi-driver \
  jetstack/cert-manager-csi-driver \
  --namespace cert-manager \
  --wait

```

You should now have the following pods running in the `cert-manager` namespace:

```
$ kubectl get pods -n cert-manager
NAME                                      READY   STATUS    RESTARTS      AGE
cert-manager-559b5d5b7d-gr8rt             1/1     Running   0             1h
cert-manager-cainjector-f5c6565d4-z8hzg   1/1     Running   0             1h
cert-manager-csi-driver-c8jmg             3/3     Running   0             1h
cert-manager-csi-driver-vkq5d             3/3     Running   0             1h
cert-manager-webhook-5f44bc85f4-ghqkm     1/1     Running   0             1h

```

Cert-manager-cainjector is a component of cert manager that injects certificate-related data into pods running in the cluster.It injects certificates, private keys, and other cryptographic material into application workloads, making them readily available to secure communication within the cluster.By injecting certificates directly into the pods, cert-manager-cainjector eliminates the need for manual configuration or mounting of certificates.  It simplifies securing communication between services within the Kubernetes cluster. Cert-manager-cainjector automatically updates and rotates certificates according to the defined policy

### Create a Private Certificate Authority (CA)

Next, we'll create private CA, which cert-manager will use to issue and manage certificates for services that run in your cluster.

Cert-manager has two issuers that create client and CA certificates: **ClusterIssuer** and **Issuer**. The fundamental difference between ClusterIssuer and Issuer is their scope. Issuer is limited to a specific namespace, while ClusterIssuer has a cluster-wide scope.

An Issuer can only create certificates within the same namespace where it is defined. A ClusterIssuer can issue certificates across multiple namespaces in a Kubernetes cluster. For example, if you have a web application deployed in the "webapp" namespace and you want to issue certificates for that specific namespace, you can define an Issuer resource within the "webapp" namespace. If you have a cluster-wide API service that needs to communicate securely with services in other namespaces, you can define a ClusterIssuer to issue certificates for all those namespaces.

I'll create a ClusterIssuer to issue a root certificate and an Issuer to create service certificates. The Issuer resource is created in `sandbox` namespace where I'll create a TLS server later in the post. If I create more namespaces, I can create Issuers in those namespaces and use the ClusterIssuer to issue certificates using the private CA.

Create a ClusterIssuer, a root certificate, and an Issuer:

```yaml
cat << EOF > cert-manager-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sandbox
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-selfsigned-ca
  namespace: sandbox
spec:
  isCA: true
  commonName: my-selfsigned-ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: my-ca-issuer
  namespace: sandbox
spec:
  ca:
    secretName: root-secret
EOF

kubectl apply -f cert-manager-resources.yaml

```

### Create sample workloads to test encryption

With cert-manager setup, let's create a TLS server. The key thing to note about service certificates is that we need to specify the **Common Name** (also known as CN) when creating a certificate. The CN represents the DNS name of the service the certificate is protecting.

Say, you're hosting your service at service.example.com, then you'd have to specify service.example.com as the CN. If a client tries to access this service using an alternate name (e.g., service.subdomain.example.com) it will get an SSL error.

In a Kubernetes cluster, a service can have multiple DNS names. For example, a service called `my-simple-service`, can also be called `my-simple-service.my-namespace.svc`, `my-simple-service.my-namespace.svc.cluster.local`, or example.com/mysimpleservice depending on how clients access the service. So, when generating a certificate for a Kubernetes service, we need to ensure that the certificate is also valid for all alternate DNS names.

When generating a certificate for a service that's accessible at multiple DNS names, you'll have to include all alternate DNS names in the certificate. This type of certificate is called a **Subject Alternate Name** (or SAN) certificate as it allows multiple hostnames to be protected by a single certificate.

If you were to manually generate a certificate with alternate names, the [process is pretty tedious](https://gist.github.com/KeithYeh/bb07cadd23645a6a62509b1ec8986bbc).  Cert-manager simplifies this process. You can create a Certificate resource with alternate names like this:

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: sample-certificate
spec:
  secretName: sample-certificate-secret
  issuerRef:
    name: private-ca-issuer
    kind: Issuer
  commonName: example.com
  dnsNames:
    - example.com
    - www.example.com
  ipAddresses:
    - 192.168.0.1

```

When you create a certificate, cert-manager creates a Kubernetes secret in the same namespace. You can then mount this secret in your pods.

But, there's an even easier way to get certificates for your applications. This is where [cert-manager CSI driver](https://cert-manager.io/docs/usage/certificate/) comes in. You can use cert-manager CSI driver to create service certificates on the fly. You can define certificates request right in your pod template.  It runs as daemonset and creates certificates without creating a secret. Your keys are stored on the node and they never leave the node.

Here are benefits of using cert-manager CSI driver:

* Ensure private keys never leave the node and are never sent over the network. All private keys are stored locally on the node.
* Unique key and certificate per application replica with a grantee to be present on application run time.
* Reduce resource management overhead by defining certificate request spec in-line of the Kubernetes Pod template.
* Automatic renewal of certificates based on expiry of each individual certificate.
* Keys and certificates are destroyed during application termination.
* Scope for extending plugin behavior with visibility on each replica's certificate request and termination.

Create a sample web service:

```
cat << EOF > https-server-using-csi-driver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sample-api
  name: sample-api
  namespace: sandbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-api
  template:
    metadata:
      labels:
        app: sample-api
    spec:
      restartPolicy: Always
      volumes:
      - name: tls
        csi:
          driver: csi.cert-manager.io
          readOnly: true
          volumeAttributes:
             csi.cert-manager.io/issuer-name: my-ca-issuer
             csi.cert-manager.io/dns-names: \${POD_NAME}.\${POD_NAMESPACE}.svc.cluster.local,sample-api-service.sandbox.svc.cluster.local,sample-api-service.sandbox.svc.,sample-api-service.
      containers:
      - name: sample-api
        image: public.ecr.aws/k9t3d5o9/secureservice:latest
        volumeMounts:
        - mountPath: "/etc/tls"
          name: tls
          readOnly: true
        ports:
        - containerPort: 8443
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sample-api-service
  name: sample-api-service
  namespace: sandbox
spec:
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    app: sample-api
  type: ClusterIP
EOF

kubectl apply -f https-server-using-csi-driver.yaml

```

In the template, I've created a volume of type `csi.driver = csi.cert-manager.io`. I've also specified the Issuer `csi.cert-manager.io/issuer-name: my-ca-issuer`, which is the Issuer I created earlier in the post. Finally, I've also added all the DNS names at which my service is accessible using `csi.cert-manager.io/dns-names: \${POD_NAME}.\${POD_NAMESPACE}.svc.cluster.local,sample-api-service.sandbox.svc.cluster.local,sample-api-service.sandbox.svc.,sample-api-service`.

The container image runs a simple Python HTTPS server. The code uses a PEM file to load certificates. I use the `tls.crt` and `tls.key` files that the CSI driver creates to generate the PEM file.

```
cat /etc/tls/tls.crt /etc/tls/tls.key > tls.pem

```

Here's the code for the HTTPS server:

```python
import ssl
from http.server import HTTPServer, BaseHTTPRequestHandler

class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(b'<h1>Serving page securely</h1>')

def run(server_class, handler_class, pem_file, port):
    server_address = ('0.0.0.0', port)
    httpd = server_class(server_address, handler_class)
    httpd.socket = ssl.wrap_socket(httpd.socket, certfile=pem_file, server_side=True)
    print(f'Starting server on port {port}...')
    httpd.serve_forever()

if __name__ == '__main__':
    pem_file = 'tls.pem'
    port = 8443
    run(HTTPServer, SimpleHTTPRequestHandler, pem_file, port)

```

### Create a client

Next we want to make sure that the certificate works. If I access this service with a web browser, I'll get an error stating the certificate is invalid. That's because it is a self-signed certificate. My browser rightfully doesn't trust the CA I just created. But if I import the root certificate, then I'll be able to access the service securely without an invalid certificate error.

Let's create a client application and we'll use `curl` to access the service:

```
cat << EOF > client-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sample-api-client
  name: sample-api-client
  namespace: sandbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-api-client
  template:
    metadata:
      labels:
        app: sample-api-client
    spec:
      volumes:
      - name: tls
        csi:
          driver: csi.cert-manager.io
          readOnly: true
          volumeAttributes:
             csi.cert-manager.io/issuer-name: my-ca-issuer
      containers:
      - name: sample-api
        image: alpine/curl
        command: ["/bin/sh", "-c"]
        args: ["while true; do sleep 30; done"]
        volumeMounts:
        - mountPath: "/etc/tls"
          name: tls
          readOnly: true
EOF

kubectl apply -f client-app.yaml

```

Note that since this is a client application, I am not specifying any dns-names.

Now let's test connecting from the client to the service:

```
kubectl -n sandbox get pods -l app=sample-api-client -o name \
  | xargs -I{} kubectl -n sandbox exec {} -- \
  curl -s https://sample-api-service

```

I get an error:

```
curl: (60) SSL certificate problem: unable to get local issuer certificate

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
command terminated with exit code 60

```

This is because I have to specify the root certificate (as this is a self-signed certificate). Let's specify the root certificate this time using curl's `--cacert` option.

```
kubectl -n sandbox get pods -l app=sample-api-client -o name \
  | xargs -I{} kubectl -n sandbox exec {} -- \
  curl -s https://sample-api-service --cacert /etc/tls/ca.crt

```

{{< figure src="https://digitalpress.fra1.cdn.digitaloceanspaces.com/clrvv0c/2023/06/28A6FEAD-EEF3-42AA-B781-37B5CF2637C5.png" >}}

The service is accessible at [https://sample-api-service](https://sample-api-service). Now let's verify the certificate also works when we access the service using `sample-api-service.sandbox.svc.cluster.local` and `sample-api-service.sandbox.svc.` names.

```
kubectl -n sandbox get pods -l app=sample-api-client -o name \
  | xargs -I{} kubectl -n sandbox exec {} -- \
  curl -s https://sample-api-service.sandbox.svc --cacert /etc/tls/ca.crt

kubectl -n sandbox get pods -l app=sample-api-client -o name \
  | xargs -I{} kubectl -n sandbox exec {} -- \
  curl -s https://sample-api-service.sandbox.svc.cluster.local --cacert /etc/tls/ca.crt

```

{{< figure src="https://digitalpress.fra1.cdn.digitaloceanspaces.com/clrvv0c/2023/06/F3DE8CF1-1D4D-45FC-854A-4C0342CE7247.png" >}}

The service is accessible at all 3 addresses.

There you have it. The communication from client to server now uses TLS encryption.

## Cleanup

Delete the resources created in this post:

```
kubectl -n sandbox delete deployments.apps sample-api sample-api-client

```

If you'd like to uninstall cert-manager and cert-manager-csi-driver:

```
helm uninstall -n cert-manager cert-manager-csi-driver 
helm uninstall -n cert-manager cert-manager

```

## Conclusion

In this post I showed you how to encrypt traffic in a Kubernetes cluster using cert-manager. I created a private CA and used it to generate service and client certificates. I also showed you can use cert-manager CSI driver to easily issue certificates for your applications.

If your applications need additional certificates, you can use cert-manager [trust-manager](https://cert-manager.io/docs/projects/trust-manager/) to inject trust bundles into your Kubernetes pods.

