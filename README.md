# k8s-demo-2b

For this demo, we will be looking into ingress, and how we can utilise it in our kube cluster.

Right now, we have a pod, with it's corresponding service, which are accessible as an external service when
port-forwarding, via ```http://<LOCAL-IP>:<LOCAL-PORT>```. However, this is not what we want in a final product.
We would like to have a secure protocol and a domain name, like so ```https://my-sample-app.com```.
Therefore, our components would be: sample-app pod, service, and ingress.

We need a new ingress configuration file, named **helm-ingress.yaml**, like so:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: new-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  tls:
    - hosts:
        - test.example.info
      secretName: my-secret
  rules:
    - host: test.example.info
      http:
        paths:
          - path: /chart
            pathType: Prefix
            backend:
              service:
                name: myfirsthelmchart-mychart
                port:
                  number: 80
```
Optional additional paths, which may require some extra work to setup.
```shell
          ...
          - path: /prom-server
            pathType: Prefix
            backend:
              service:
                name: myprometheus-server
                port:
                  number: 80
          - path: /sample
            pathType: Prefix
            backend:
              service:
                name: sample-app-service
                port:
                  number: 8080
```

Note, that the rules include **http**. This doesn't define whether the domain is secure, just the protocol itself.

When configuring the ingress, the host must:
- Be a valid domain address
- Map the domain name to the Node's IP address, which will be the entrypoint to
  the cluster, i.e. **minikube**.

Now that we have the yaml configuration file for ingress, we can work on the implementation of the ingress controller.
This will be in the form of a pod, or even a set of pods, within the current node/cluster. This will evaluate and
process the ingress rules. It will manage redirections, act as the entrypoint, and configure any domains and subdomains
define. 

We will use a basic nginx deployment for the ingress controller. let's use helm, as it is built to help us with things
like this.
```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

<details>
<summary><i> Minikube option</i></summary>

You use the minikube add-on, via ```minikube addons enable ingress```.

Once complete, run the command below, and you should see something like this:
```shell
kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-<id>         0/1     Completed   0          4m4s
ingress-nginx-admission-patch-<id>          0/1     Completed   1          4m4s
ingress-nginx-controller-<id>               1/1     Running     0          4m4s
```
</details>

Let's check the status of out ingress. If you don't see the running controller, keep trying until it is set up.

```shell
$ kubectl get pods --namespace=ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-4w5x5        0/1     Completed   0          147m
ingress-nginx-admission-patch-csm67         0/1     Completed   1          147m
ingress-nginx-controller-6d5f55986b-ctm62   1/1     Running     0          147m
```
Let's actually try out ingress we created earlier:

```shell
kubectl apply -f helm-ingress.yaml
```

Run the get ingress command, and take the ADDRESS field, adding it to the bottom of your **/etc/hosts** file:
```shell
kubectl get ingress
```
```shell
# Added by user for ingress
<ADDRESS> test.example.info
```

If mentioned in minikube, you may need to run ```minikube tunnel``` to allow for ingress resources to be accessible.

```shell
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
```

Now that we have everything set up, let's generate a secret, so we can secure our connection.
Let's call the file **gen.sh**. Edit the subject below and insert it in the labelled sections <insert_subject_here>
"/C=UK/ST=<county>/L=<town>/O=FPL/OU=<unit>/CN=*.example.info/emailAddress=<user>@fat-potato.co.uk"

```shell
rm *.pem

# 1. Generate CA's private key and self-signed certificate
openssl req -x509 -newkey rsa:4096 -days 365 -nodes -keyout ca-key.pem -out ca-cert.pem -subj <insert_subject_here>

echo "CA's self-signed certificate"
openssl x509 -in ca-cert.pem -noout -text

# 2. Generate web server's private key and certificate signing request (CSR)
openssl req -newkey rsa:4096 -nodes -keyout server-key.pem -out server-req.pem -subj <insert_subject_here>
# 3. Use CA's private key to sign web server's CSR and get back the signed certificate
openssl x509 -req -in server-req.pem -days 60 -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile server-ext.cnf

echo "Server's signed certificate"
openssl x509 -in server-cert.pem -noout -text
```
Now we can create the secret:
```shell
kubectl create secret tls my-secret --cert=server-cert.pem --key=server-key.pem
```
Go back to **helm-ingress.yaml** and add the secret like so:
```shell
...
spec:
  tls:
    - hosts: test.example.info
      secretName: my-secret
  rules:
    ...
```

Now we should be able to access the helm chart at **https://test.example.info/chart**.
If everything is set up correctly, we should also be able to access **https://test.example.info/sample**