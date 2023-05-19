
# NGINX Ingress Controller for Consul on Kubernetes Example

This example configuration deploys and configures a [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/) as described by the Consul Service Mesh docs on integrating [Ingress Controllers](https://developer.hashicorp.com/consul/docs/k8s/connect/ingress-controllers) with Consul K8s using transparent proxy. It is heavily inspired by [@dhiaayachi](https://github.com/dhiaayachi/eks-consul-ingressnginx) and [@david-yu](https://github.com/david-yu/consul-k8s-nginx-inc-ingress-controller).

## Requirements

- Kubernetes cluster on any [cloud provider](cloud-provider-setup)
- `kubectl` installed locally
- `helm` installed locally
- `jq` installed locally

Once you have the requirements, follow the instructions in the [Runbook](#runbook) section to configure an NGINX Inc ingress controller with Consul on Kubernetes

### Cloud Provider Setup

Consul on K8s can be deployed on any K8s distro such as EKS, GKE, and AKS. The following show you how to deploy and configure an EKS or EKS in AWS and Google Cloud respectively.

#### AWS Setup 

#### Requirements 

- An AWS account and a region that supports EKS
- Environment variables to access AWS account locally
- `eksctl` installed locally

#### Steps

1. Create a EKS cluster.

    ```bash
    eksctl create cluster --name=<cluster name> --region=<region> --nodes=3 
    ```
  
2. Configure `kubectl`.

    ```bash
    aws eks update-kubeconfig --region <region> --name <cluster name>
    ```

#### Google Cloud Setup

##### Requirements

- A Google account and a region that supports GKE
- Environment variables to access GKE account locally
- `gcloud` installed locally

##### Steps

1. Set environment variables.

    ```bash
    export PROJECT=<PROJECT ID>
    gcloud config set project $PROJECT
    gcloud config set compute/zone us-west1-c
    ```
  
2. Create a GKE cluster.

    ```bash
    gcloud container clusters create nginx-consulk8s --num-nodes=3 --machine-type "e2-highcpu-4" --enable-autoscaling --min-nodes 1 --max-nodes 4
    ```

3. Configure `kubectl`.

    ```bash
    gcloud container clusters get-credentials nginx-consulk8s
    ```

## Runbook

1. Deploy Consul.

    ```bash
    helm repo add hashicorp https://helm.releases.hashicorp.com
    helm install consul hashicorp/consul --values consul-values.yaml --version "1.0.7" --create-namespace --namespace consul
    ```

2. Add deny all intention.

    ```bash
    kubectl apply -f deny-all.yaml
    ```

3. Deploy NGINX Inc Ingress Controller ([ingress-nginx](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/)).

    ```bash
    helm upgrade --install nginx-ingress oci://ghcr.io/nginxinc/charts/nginx-ingress --version 0.17.1 --namespace nginx-ingress --create-namespace --values nginx-ingress-values.yaml
    ```

4. Configure ServiceDefaults to enable [DialedDirectly](https://developer.hashicorp.com/consul/docs/connect/config-entries/service-defaults#dialeddirectly) for transparent proxy.

    ```bash
    kubectl apply -f sd-direct.yaml
    ```

5. Set NGINX load balancer IP as an environment variable.

    ```bash
    export NGINX_INGRESS_IP=$(kubectl get service nginx-ingress-controller -n nginx-ingress -o json | jq -r '.status.loadBalancer.ingress[].ip')
    ```

6. Generate Ingress resource configuration with NGINX load balancer IP.

    ```bash
    cat <<EOF > ingress-resource.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: test-nginx-ingress
      annotations:
        nginx.org/client-max-body-size: "4m"
        nginx.org/underscores-in-headers: "on"
        nginx.org/proxy-read-timeout: "300"
        nginx.org/proxy-send-timeout: "300"
        nginx.org/proxy-connect-timeout: "300"
        nginx.org/keepalive: "300"
        nginx.org/proxy-buffer-size: 8k
    spec:
      ingressClassName: nginx
      rules:
      - host: "$NGINX_INGRESS_IP.nip.io"
        http:
          paths:
          - path: /server
            pathType: Prefix
            backend:
              service:
                name: static-server
                port: 
                  number: 8080
      defaultBackend:
        service:
          name: static-server
          port:
            number: 8080
    EOF
    ```

7. Configure Ingress config to route traffic to `static-server`.

    ```bash
    kubectl apply -f ingress-resource.yaml
    ```

8. Deploy `static-server`. 

    ```bash
    kubectl apply -f static-server.yaml
    ```

9. Apply intention from ingress to `static-server`.

    ```bash
    kubectl apply -f allow-static-server.yaml
    ```

10. Verify NGINX ingress by making a request to the NGINX hostname for the `static-server` route. 

    ```bash
    curl ${NGINX_INGRESS_IP}.nip.io
    ```

    Response:

    ```text
    "hello world"
    ```

