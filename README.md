# canary-demo-gitops

Estado desejado do cluster `lab`. O workflow da aplicação atualiza a imagem (por digest) em `clusters/lab/app-canary/kustomization.yaml`.

## Instalação da plataforma (executar nas VMs; versões a fixar e registrar abaixo)

1. VMs: hostname, NTP, swap off, módulos `overlay` e `br_netfilter`, sysctl de rede, containerd, kubeadm/kubelet/kubectl fixados.
2. `kubeadm init --pod-network-cidr=<CIDR>`; instalar o CNI (Calico ou Cilium); `kubeadm join` nos workers.
3. Istio: `istioctl install` com requests/limits no `istiod` e no ingress gateway; expor via NodePort ou MetalLB.
4. Argo CD: `kubectl create ns argocd` e aplicar o manifest oficial da versão fixada; configurar TLS e trocar a senha admin.
5. Argo Rollouts: `kubectl create ns argo-rollouts` e aplicar o manifest oficial da versão fixada.
6. Certificado TLS: criar o Secret `canary-demo-tls` em `istio-system` (fora do Git).
7. Cadastrar este repositório no Argo CD (deploy key SSH somente leitura).
8. `kubectl apply -f clusters/lab/argocd/project.yaml -f clusters/lab/argocd/application.yaml`.
9. Criar o token da role `ci`: `argocd proj role create-token canary-demo ci`; guardar em secret do GitHub.
10. Adicionar `app.canary.lab` ao DNS interno ou `/etc/hosts` apontando para o gateway.

### Versões utilizadas (preencher)

Kubernetes: ? | CNI: ? | Istio: ? | Argo CD: ? | Argo Rollouts: ?

## Operação

- Deploy: automático pelo pipeline. O Rollout pausa em 50% (VirtualService 50/50 aplicado pelo Argo Rollouts).
- Promover: `argocd app actions run canary-demo resume --kind Rollout --resource-name canary-demo`
- Abortar: `argocd app actions run canary-demo abort --kind Rollout --resource-name canary-demo`
- Rollback por Git: `git revert` do commit de deploy e sync no Argo CD.
- Rollback por Argo CD: `argocd app rollback canary-demo <ID>` (exige auto-sync desabilitado).
- Validar proporção: `SMOKE_URL=https://canary.diegovieira.app/ STABLE_VERSION=x CANARY_VERSION=y node scripts/smoke-ratio.js` (script do repositório da aplicação). O app também responde em `canary.diegovieira.app` com o certificado wildcard `argocd-tls`; `app.canary.lab` continua exigindo o Secret `canary-demo-tls` e `SMOKE_HOST=app.canary.lab SMOKE_INSECURE=true`.
- Rotação de token: gerar novo token da role `ci`, atualizar `ARGOCD_AUTH_TOKEN`, revogar o antigo.

## Limitações do laboratório

Um control plane (sem HA), mTLS em PERMISSIVE, aprovação por PR dispensada (push direto em `main`), sem análise Prometheus (`AnalysisTemplate` fica para a fase 8).
