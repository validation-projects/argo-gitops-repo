# Cluster lab: infraestrutura e GitOps

Este repositório contém o provisionamento do cluster Kubernetes e os manifests
da plataforma e do workload canary. O workflow da aplicação atualiza a imagem
(por digest) em `clusters/lab/app-canary/kustomization.yaml`.

## Conteúdo

- `infra/ansible/`: preparação dos nós, inicialização do control plane, entrada
  dos workers e instalação da plataforma.
- `cluster/argocd-expose/`: emissor de certificados,
  TLS, Gateway, VirtualService e políticas para expor o Argo CD.
- `clusters/lab/app-canary/`: Rollout, Services, tráfego canary e políticas da
  aplicação.
- `clusters/lab/argocd/`: projeto e aplicação do Argo CD.

O Ansible instala Kubernetes, Calico, Istio, Argo CD e cert-manager. O ingress
usa NodePort por padrão; MetalLB é opcional e habilitado definindo `metallb_ip`
em `infra/ansible/group_vars/all.yml`.

## Provisionar a infraestrutura

1. Ajuste os endereços e hosts em `infra/ansible/inventory/hosts.yml` e os
   parâmetros/versões em `infra/ansible/group_vars/all.yml`.
2. Instale Ansible e as collections usadas pelos playbooks:

   ```sh
   ansible-galaxy collection install community.general ansible.posix
   ```

3. Configure acesso SSH com privilégios administrativos aos nós e execute:

   ```sh
   cd infra/ansible
   ansible-playbook site.yml
   ```

O playbook `reset.yml` limpa o estado Kubernetes de todos os nós. Ele é
destrutivo e exige confirmação explícita.

## Pós-instalação

- Crie o Secret `digitalocean-dns` no namespace `cert-manager` com o token da
  API da DigitalOcean para que o cert-manager possa emitir o certificado.
- Configure o DNS de `argo.diegovieira.app` e `ingress.diegovieira.app` e o
  encaminhamento WAN de acordo com o modo de ingress escolhido: NodePort
  `30080/30443` ou o IP configurado no MetalLB.
- Troque a senha inicial do usuário `admin` do Argo CD e remova o Secret de
  senha inicial após a troca.

## Instalação e operação do canary

1. Crie o Secret `canary-demo-tls` em `istio-system` (fora do Git), se usar
   `app.canary.lab`.
2. Cadastre este repositório no Argo CD com uma deploy key SSH somente leitura.
3. Aplique o projeto e a aplicação:

   ```sh
   kubectl apply -f clusters/lab/argocd/project.yaml \
     -f clusters/lab/argocd/application.yaml
   ```

4. Crie o token da role `ci` com `argocd proj role create-token canary-demo ci`
   e guarde-o em um secret do GitHub.
5. Configure `app.canary.lab` no DNS interno ou em `/etc/hosts`, apontando para
   o gateway.

O deploy é automático pelo pipeline. O Rollout pausa em 50% (VirtualService
50/50 aplicado pelo Argo Rollouts).

- Promover: `argocd app actions run canary-demo resume --kind Rollout --resource-name canary-demo`
- Abortar: `argocd app actions run canary-demo abort --kind Rollout --resource-name canary-demo`
- Rollback por Git: reverta o commit de deploy e sincronize no Argo CD.
- Rollback por Argo CD: `argocd app rollback canary-demo <ID>` (exige auto-sync
  desabilitado).
- Rotação de token: gere um novo token da role `ci`, atualize
  `ARGOCD_AUTH_TOKEN` e revogue o antigo.

Para validar a proporção do tráfego, use o script do repositório da aplicação:

```sh
SMOKE_URL=https://canary.diegovieira.app/ \
  STABLE_VERSION=x CANARY_VERSION=y node scripts/smoke-ratio.js
```

O app também responde em `canary.diegovieira.app` com o certificado wildcard
`argocd-tls`; `app.canary.lab` continua exigindo o Secret `canary-demo-tls` e
`SMOKE_HOST=app.canary.lab SMOKE_INSECURE=true`.

## Limitações do laboratório

Um control plane (sem HA), mTLS em PERMISSIVE, aprovação por PR dispensada
(push direto em `main`), sem análise Prometheus (`AnalysisTemplate` fica para a
fase 8).
