# ingresses-n-classes
Just a repo to facilitate the compreension of internal and external, nlb and alb ingress installation on AWS


````markdown
# Kubernetes Ingress/IngressClass: External vs Internal (ALB & NLB)

Este guia resume os principais pontos a observar ao executar dois Ingress (ou IngressClasses) num mesmo cluster — um para tráfego externo e outro para tráfego interno, utilizando ALB ou NLB na AWS.

---

## 1. ALB vs NLB: Visão Geral

| Load Balancer | Integra com          | Configuração                                                                                          |
|---------------|----------------------|------------------------------------------------------------------------------------------------------|
| ALB           | Ingress (HTTP/HTTPS) | 1. CRDs IngressClassParams + IngressClass  
  2. Annotation no Ingress:
  ```yaml
  alb.ingress.kubernetes.io/scheme: internal  # ou internet-facing
  ```                                            |
| NLB           | Service LoadBalancer | 1. Annotation no Service:
  ```yaml
  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"  # ou internet-facing
````

2. Sem IngressClass nativo para NLB.                                            |

---

## 2. Criando ALB IngressClass (Externa & Interna)

```yaml
# IngressClassParams
apiVersion: elbv2.k8s.aws/v1beta1
kind: IngressClassParams
metadata:
  name: alb-internal    # ou alb-external
spec:
  scheme: internal     # ou internet-facing
---
# IngressClass
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb-internal    # ou alb-external
spec:
  controller: ingress.k8s.aws/alb
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: alb-internal
```

**Opt-in via Helm:**

```bash
helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set createIngressClassResource=true \
  --set ingressClass=alb-internal \
  --set ingressClassConfig.default=false
```

Use em seus Ingresses:

```yaml
spec:
  ingressClassName: alb-internal
```

---

## 3. Criando NLB via Service (Externa & Interna)

> **Nota:** NLB é gerado a partir de Service, não via Ingress.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: meu-nlb-interno
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: minha-app
```

Para roteamento HTTP, posicione um Ingress Controller (ex: NGINX) atrás deste Service.

---

## 4. Dois Controllers NGINX: Externo × Interno

1. Namespaces separados:

   - ingress-nginx (internet-facing)
   - ingress-nginx-internal (internal)

2. Configuração interna (`values-internal.yaml`):

```yaml
controller:
  ingressClass: nginx-internal
  ingressClassResource:
    name: nginx-internal
    default: false
  publishService:
    enabled: true  # publica hostname do Service no status do Ingress
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
```

3. Instalação/Upgrade via Helm:

```bash
helm upgrade --install nginx-internal ingress-nginx/ingress-nginx \
  --namespace ingress-nginx-internal \
  -f values-internal.yaml
```

4. Em seus Ingresses internos:

```yaml
spec:
  ingressClassName: nginx-internal
```

---

## 5. Permissões IAM & Tags de Subnets

- Tagueie suas subnets privadas:

  ```bash
  aws ec2 create-tags --resources <subnet-id> \
    --tags Key=kubernetes.io/role/internal-elb,Value=1
  ```

- Permissões IRSA para aws-load-balancer-controller:

  - EC2: DescribeSubnets, DescribeRouteTables, DescribeSecurityGroups, DescribeVpcs
  - ELBv2: CreateLoadBalancer, CreateTargetGroup, ModifyLoadBalancerAttributes, DescribeListenerAttributes, ModifyListenerAttributes

Use a policy oficial AmazonEKSLoadBalancingPolicy ou uma custom que inclua essas ações.

---

## 6. Publicar Endereço no Status do Ingress

Ative `publishService.enabled: true` no controller interno. Se o campo ADDRESS do Ingress continuar apontando para o ALB externo, limpe ou reaplique:

```bash
kubectl -n qa patch ingress tebas-api-ingress --type merge -p '{"status":{}}'
```

---

## 7. Checklist Final

- ALB

  - CRDs IngressClassParams + IngressClass
  - Annotation no Ingress (`alb.ingress.kubernetes.io/scheme`: internal ou internet-facing)
  - Helm opt-in (createIngressClassResource=true, ingressClass=)

- NLB

  - Service tipo LoadBalancer anotado com:
    - service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    - service.beta.kubernetes.io/aws-load-balancer-scheme: "internal" ou internet-facing

- Controllers NGINX

  - Dois controllers (externo × interno) em namespaces separados
  - publishService.enabled: true no controller interno

- Infra AWS

  - Subnets privadas tagueadas (kubernetes.io/role/internal-elb=1)
  - IAM IRSA com permissões EC2 & ELBv2 (incluindo DescribeListenerAttributes)

- Reaplicar Ingress

  - Limpar ou reaplicar Ingress para atualizar ADDRESS no status

---

## Lições Aprendidas

- **Erros que cometemos**

  - Esquecer de criar IngressClassParams para ALB, levando ao uso apenas de NLB
  - Não taguear subnets privadas, impedindo o provisionamento de balanceador interno
  - Role sem permissão DescribeListenerAttributes, bloqueando reconciliação do service

- **Pontos de atenção**

  - Sempre criar e vincular IngressClassParams antes de usar ingressClassName
  - Verificar e aplicar tags nas subnets (`kubernetes.io/role/internal-elb=1`) antes de provisionar LBs internos
  - Garantir que a IRSA do controller contenha todas as permissões EC2 e ELBv2 necessárias

```
```
