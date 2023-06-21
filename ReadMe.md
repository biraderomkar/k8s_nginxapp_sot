# Account used by fluxcd to communicate with SOT.
export GITHUB_USER=biraderomkar

export GITHUB_TOKEN=""

# Bootstrapping fluxcd toolkits in k8s cluster

flux bootstrap github \
 --owner=$GITHUB_USER \
 --repository=fluxcdInfraSOT \
 --branch=main \
 --path=deployinfra \
 --personal

# Define source SOT
flux create source git k8s_nginxapp_sot \
 --url=https://github.com/biraderomkar/k8s_nginxapp_sot.git \
 --branch=main \
 --interval=30s \
 --export > ./deploy/flux_source.yaml

# Apply above source SOT using kustomization controller
 flux create kustomization k8s_nginxapp_sot \
 --source=k8s_nginxapp_sot  \
 --path="./deploy/" \
 --prune=true \
 --validation=client \
 --interval=30s  \
 --export > ./deploy/flux_sync.yaml

 # Setup Sealed Secret Controller
 wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.1/controller.yaml

 kubectl apply -f controller.yaml

 # Install kubeseal client 
 wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.4/kubeseal-0.19.4-linux-amd64.tar.gz

 tar -xvzf kubeseal-0.19.4-linux-amd64.tar.gz kubeseal

 install -m 755 kubeseal /usr/local/bin/kubeseal

# First create a secret file with base64 encoded secrets

  echo -n "token/password" |  base64

 # Create Sealed secret manifest file
 
 kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system  -n fluxcdtutorial --format yaml < secret.yaml > sealedSecret.yaml

 # Locally decrypt existing sealed secrete using deployed controller in k8s environment

 Check logs : kubectl logs -f <<SealedSecretPodName>> -n kube-system

 GetPrivateKeys: kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml

 kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system < sealedSecret.yaml --recovery-unseal --recovery-private-key sealed-secrets-key.yaml -o yaml

 Use this link to decode base64 password https://www.base64decode.org/


  # Setting Up prometheus stack
 flux create source git flux-monitoring \
  --interval=1m \
  --url=https://github.com/fluxcd/flux2 \
  --branch=main \
  --export > ./deploy/flux_monitoring_source.yaml

# Apply kube-prom-stack using below kustomization definition
flux create kustomization kube-prometheus-stack \
  --interval=1m \
  --prune \
  --source=flux-monitoring \
  --path="./manifests/monitoring/kube-prometheus-stack" \
  --health-check-timeout=5m \
  --wait \
  --export > ./deploy/flux_monitoring_sync.yaml

# Collect metrics in prometheus and setup grafana dashboard
flux create kustomization kube-monitoring-config \
  --interval=1m \
  --prune \
  --source=flux-monitoring \
  --path="./manifests/monitoring/monitoring-config" \
  --health-check-timeout=5m \
  --wait \
  --export > ./deploy/flux_monitoring-config.yaml