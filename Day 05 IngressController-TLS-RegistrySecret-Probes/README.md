# TLS Certificates, Kubernetes Secrets & Traefik Ingress Setup

This guide covers generating TLS certificates with Let's Encrypt, configuring Kubernetes secrets, installing Helm, and deploying the Traefik ingress controller.

---

## Prerequisites

- A domain name you control (with DNS management access)
- A running Kubernetes cluster with `kubectl` configured
- Ubuntu/Debian-based system
- A Docker Hub account with an access token

---

## 1. Generate TLS Certificates with Let's Encrypt (Certbot)

Update your package list and install Certbot via Snap:

```bash
sudo apt update
sudo snap install --classic certbot
```

Generate a wildcard TLS certificate using DNS challenge verification. Replace `<Your-Email-ID>` and `<Your-Domain>` with your actual values:

```bash
certbot certonly --manual --preferred-challenges=dns \
  --key-type rsa \
  --email <Your-Email-ID> \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  -d *.<Your-Domain>
```

> **Note:** During this process, Certbot will ask you to add a DNS TXT record to verify domain ownership. Log in to your DNS provider and add the record before proceeding.

Once complete, retrieve your certificate and private key:

```bash
cat /etc/letsencrypt/live/<Your-Domain>/fullchain.pem
cat /etc/letsencrypt/live/<Your-Domain>/privkey.pem
```

Copy the output of these files into `tls.crt` and `tls.key` respectively for use in the next step.

---

## 2. Create TLS Secrets in Kubernetes

Create a Kubernetes TLS secret named `traefik-tls-default` using your certificate files:

```bash
# Using .crt extension
kubectl create secret tls traefik-tls-default --key="tls.key" --cert="tls.crt"

```

Verify the secret was created correctly:

```bash
kubectl describe secrets traefik-tls-default
```

---

## 3. Install Helm

Install the required dependencies and add the Helm repository:

```bash
sudo apt-get install curl gpg apt-transport-https --yes

curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-get install helm
```

Confirm Helm is installed successfully:

```bash
helm version
```

---

## 4. Install Traefik Ingress Controller via Helm

Add the official Traefik Helm chart repository and install Traefik with HTTPS redirect enabled:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --set service.type=LoadBalancer \
  --set ports.web.port=80 \
  --set ports.websecure.port=443 \
  --set additionalArguments="{--entrypoints.web.http.redirections.entrypoint.to=websecure,--entrypoints.web.http.redirections.entrypoint.scheme=https}"
```

This configuration:
- Exposes Traefik as a `LoadBalancer` service
- Listens on port `80` (HTTP) and port `443` (HTTPS)
- Automatically redirects all HTTP traffic to HTTPS

---

## 5. Configure Docker Hub Credentials as a Kubernetes Secret

Create a Kubernetes secret to allow your cluster to pull images from Docker Hub. Replace the placeholder values with your actual credentials:

```bash
kubectl create secret docker-registry docker-pwd \
  --docker-server=docker.io \
  --docker-username=<username> \
  --docker-password=<Your_Dockerhub_Token> \
  --docker-email=<Your_Email_ID>
```

> **Tip:** Use a Docker Hub **access token** rather than your account password for better security. Tokens can be generated from your Docker Hub account settings under **Security**.

---

## Summary

| Step | Description |
|------|-------------|
| 1 | Generate wildcard TLS cert via Let's Encrypt DNS challenge |
| 2 | Store TLS cert as a Kubernetes secret |
| 3 | Install Helm package manager |
| 4 | Deploy Traefik ingress controller with HTTP→HTTPS redirect |
| 5 | Store Docker Hub credentials as a Kubernetes secret |
