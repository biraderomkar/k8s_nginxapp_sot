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