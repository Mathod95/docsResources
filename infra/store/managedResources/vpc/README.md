# Crossplane — VPC Networking (Managed Resources)

Manifests Crossplane pour provisionner un réseau AWS complet via des managed resources individuelles.  
Ces fichiers sont la base "brute" sur laquelle la [Composition](README-composition-vpc.md) est construite.

## Ressources provisionnées

| Resource | Nom | Description |
|---|---|---|
| VPC | `main-vpc` | `10.0.0.0/16`, DNS activé |
| InternetGateway | `main-igw` | Attaché au VPC |
| Subnet public | `public-subnet-a/b/c` | AZ a/b/c — `10.0.1-3.0/24` |
| Subnet privé | `private-subnet-a/b/c` | AZ a/b/c — `10.0.11-13.0/24` |
| EIP | `nat-eip` | IP fixe pour le NAT Gateway |
| NATGateway | `main-nat-gw` | Placé dans `public-subnet-a` |
| RouteTable | `public-rt` | Table de routage publique |
| RouteTable | `private-rt` | Table de routage privée |
| Route | `public-internet-route` | `0.0.0.0/0 → IGW` |
| Route | `private-nat-route` | `0.0.0.0/0 → NAT GW` |
| RouteTableAssociation | `public-rta-a/b/c` | Subnets publics → `public-rt` |
| RouteTableAssociation | `private-rta-a/b/c` | Subnets privés → `private-rt` |

## Architecture réseau

```
VPC 10.0.0.0/16
│
├── public-subnet-a  10.0.1.0/24  (eu-west-1a)  ─┐
├── public-subnet-b  10.0.2.0/24  (eu-west-1b)  ─┼─ public-rt → Internet Gateway
├── public-subnet-c  10.0.3.0/24  (eu-west-1c)  ─┘       │
│                                                    Internet
├── private-subnet-a  10.0.11.0/24  (eu-west-1a)  ─┐
├── private-subnet-b  10.0.12.0/24  (eu-west-1b)  ─┼─ private-rt → NAT Gateway (EIP)
└── private-subnet-c  10.0.13.0/24  (eu-west-1c)  ─┘       │
                                                       Internet
```

> Les nodes EKS spawneront dans les **subnets privés**. Les subnets publics hébergent uniquement le NAT Gateway et les Load Balancers.

## Prérequis

```bash
# Providers installés et healthy
kubectl get providers

# Secret AWS credentials présent
kubectl get secret aws-credentials -n crossplane-system

# Vérifier le ProviderConfig
kubectl get providerconfig aws-provider-config -n crossplane-system
```

## Déploiement

```bash
# Appliquer l'ensemble des resources
kubectl apply -f crossplane-vpc-networking.yaml

# Suivre la création en temps réel
kubectl get managed -w
```

> ⚠️ L'ordre de création est géré automatiquement par Crossplane via les `xIdSelector`. Le NAT Gateway attend que l'EIP et le subnet soient `Ready` avant de se créer.

## Commandes de gestion

### État global

```bash
# Toutes les managed resources et leur état
kubectl get managed

# Filtrer par type de resource
kubectl get vpc,internetgateway,subnet,natgateway,eip -n crossplane-system
kubectl get routetable,route,routetableassociation -n crossplane-system
```

### Inspecter une resource

```bash
# Détail complet + conditions + atProvider (IDs AWS)
kubectl describe vpc main-vpc -n crossplane-system

# Récupérer l'ID AWS d'une resource
kubectl get vpc main-vpc -n crossplane-system -o jsonpath='{.status.atProvider.id}'
kubectl get natgateway main-nat-gw -n crossplane-system -o jsonpath='{.status.atProvider.id}'
kubectl get eip nat-eip -n crossplane-system -o jsonpath='{.status.atProvider.publicIp}'
```

### Vérifier la santé

```bash
# Vue synthétique Synced/Ready sur toutes les managed resources
kubectl get managed \
  -o custom-columns='NAME:.metadata.name,SYNCED:.status.conditions[?(@.type=="Synced")].status,READY:.status.conditions[?(@.type=="Ready")].status'
```

### Déboguer

```bash
# Events sur une resource en erreur
kubectl events vpc/main-vpc -n crossplane-system
kubectl events natgateway/main-nat-gw -n crossplane-system

# Logs du provider
kubectl logs -l pkg.crossplane.io/revision=provider-aws-ec2 -n crossplane-system --tail=100 -f
```

### Suspendre la réconciliation

```bash
# Suspendre une resource (Crossplane arrête de la réconcilier avec AWS)
kubectl annotate vpc main-vpc -n crossplane-system crossplane.io/paused=true

# Reprendre
kubectl annotate vpc main-vpc -n crossplane-system crossplane.io/paused-
```

### Supprimer

```bash
# Supprimer toutes les resources (propagé à AWS)
kubectl delete -f crossplane-vpc-networking.yaml

# Suivre la suppression
kubectl get managed -w
```

> ⚠️ Le **NAT Gateway** et l'**EIP** génèrent des coûts AWS tant qu'ils existent. Vérifier la suppression complète dans la console AWS après le `delete`.

## Tags AWS appliqués

Toutes les resources portent les tags suivants :

| Tag | Valeur |
|---|---|
| `Name` | Nom de la resource (ex: `main-vpc`) |
| `ManagedBy` | `crossplane` |
| `kubernetes.io/role/elb` | `1` *(subnets publics uniquement)* |
| `kubernetes.io/role/internal-elb` | `1` *(subnets privés uniquement)* |

Les tags `kubernetes.io/role/*` sont requis par le **AWS Load Balancer Controller** pour identifier les subnets cibles lors du provisioning d'ALB/NLB depuis EKS.

## Fichiers

```
.
├── crossplane-vpc-networking.yaml   # Ce fichier — managed resources individuelles
├── xrd-vpc-networking.yaml          # XRD — version composée (API custom)
├── composition-vpc-networking.yaml  # Composition — orchestration via XR
└── xr-vpc-example.yaml             # Exemples d'instanciation via XR
```