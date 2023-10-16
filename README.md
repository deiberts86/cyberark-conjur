# CyberArk Conjur-OSS on RKE2 Kubernetes

### References and docs:
- [Conjur-OSS Docs](https://docs.conjur.org/Latest/en/Content/Overview/Conjur-OSS-Suite-Overview.html?tocpath=_____1)
- [Conjur-OSS Github](https://github.com/cyberark/conjur-oss-helm-chart/tree/main/conjur-oss)
- [Helm-Install](https://helm.sh/docs/intro/install/)
- [Kubectl-Install](https://kubernetes.io/docs/tasks/tools/)
- [Docker](https://hub.docker.com/)

#### Requirements:
- Your workstation should have the following installed:
  - kubectl
  - Helm v3 
  - Docker or Podman (This creates your data key that's required)
    - If on Windows, you will need docker with WSLv2.

- A Kubernetes Cluster to use:
  - Rancher k3s, Rancher rke2, or minikube can be used for development. I went with RKE2.
    - Great link and video resource for setting all of things needed found here: [Rancher-Government-Solutions] (https://ranchergovernment.com/blog/effortless-deployment-of-rancher-rke2-rancher-manager-longhorn-and-neuvector)
  - Load balancer services (metalLB or Kube-VIP cloud provider) to service NGINX.
  - A storageClass to utilize for PostgreSQL container (in this example, Lonhorn was used).

#### Create Conjur-OSS Namespace with PSA (Pod Security Admission) Enabled Clusters

- Create namespace:

```sh
kubectl apply -f -<<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: conjur-oss
  labels:
    kubernetes.io/metadata.name: conjur
    name: conjur
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
...
EOF
```

#### Helm Installation Conjur-OSS

- I find it easier to run these commands from a Linux host. A workstation with Windows is fine as long as you have WSLv2 with Linux kernel as well.  Otherwise you will not be able to leverage the '/udev/random' command.
- You will need to adjust the environment variables as seen below.  These are the basics to get this going. 

```sh
HELM_RELEASE="https://github.com/cyberark/conjur-oss-helm-chart/releases/download/v$VERSION/conjur-oss-$VERSION.tgz"
CONJUR_NAMESPACE="conjur"
DATA_KEY="$(docker run --rm cyberark/conjur data-key generate)"
VERSION="2.0.6"
POSTGRESQL_PASS="$(tr -dc A-Za-z0-9 </dev/urandom | head -c 16)"
helm upgrade -i conjur-oss "$HELM_RELEASE" \
--namespace "$CONJUR_NAMESPACE" \
--set openshift.enabled=false \
--set dataKey="$DATA_KEY" \
--set database.password="$POSTGRESQL_PASS" \
--set ssl.expiration="365" \
--set ssl.hostname="conjur.example.com"
```

#### Helm Upgrade Conjur-OSS

- Execute the following below to `upgrade` your already installed release.
- Note, this upgrade includes using the rancher hardened NGINX container.

```sh
HELM_RELEASE="https://github.com/cyberark/conjur-oss-helm-chart/releases/download/v$VERSION/conjur-oss-$VERSION.tgz"
CONJUR_LOGLVL="info"
CONJUR_NAMESPACE="conjur"
VERSION="2.0.7"
helm upgrade -i conjur-oss "$HELM_RELEASE" \
--namespace "$CONJUR_NAMESPACE" \
--set logLevel="$CONJUR_LOGLVL" \
--reuse-values \
--set nginx.image.repository="rancher/nginx-ingress-controller" \
--set nginx.image.tag="nginx-1.9.3-hardened1rc2-amd64" \
--set nginx.pullPolicy="Always"
```

- If successful, you should get a 'Installing Now' message.

#### Create a Username
- This is required in order to even use conjurCLI
- To create an initial account and login, follow the instructions here:
  https://www.conjur.org/get-started/install-conjur.html#install-and-configure
- Note that the conjurctl account create command gives you the
  public key and admin API key for the account administrator you created.
  Back them up in a safe location.

- Execute the following below:

```sh
export POD_NAME=$(kubectl get pods --namespace conjur \
-l "app=conjur-oss,release=conjur-oss" \
-o jsonpath="{.items[0].metadata.name}")
kubectl exec --namespace conjur \
$POD_NAME \
--container=conjur-oss \
-- conjurctl account create "default" | tail -1
```

#### ConjurCLI

Connect to Conjur with Docker or Podman.  Docker desktop works fine with this as well.

  Start a container with Conjur CLI and authenticate with the new user:
```sh
docker run --rm -it --entrypoint bash cyberark/conjur-cli:8
# Or if using MiniKube, use the following command from the host:
# docker run --rm -it --network host --entrypoint bash cyberark/conjur-cli:8

# Here ENDPOINT is the DNS name https endpoint for your Conjur service.
# NOTE: Ensure that the target endpoint matches at least one of the expected server
#       SSL certificate names otherwise SSL verification will fail and you will not
#       be able to log in.
# NOTE: Also ensure that the URL does not contain a slash (`/`) at the end of the URL
conjur init -u <ENDPOINT> -a "default" --self-signed

# API key here is the key that creation of the account provided you in step #2
conjur login -i admin -p <API_KEY>

# Check that you are identified as the admin user
conjur whoami
```

#### Helm Delete Conjur-OSS
- Ensure you have backed up your values incase you ever want to re-deploy in the future.

```sh
CONJUR_NAMESPACE="conjur"
HELM_RELEASE="conjur-oss"
helm delete -n "$CONJUR_NAMESPACE" "$HELM_RELEASE"
```