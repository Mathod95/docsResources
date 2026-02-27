# IAM - Gestion des utilisateurs et permissions

Gestion des accès AWS via **IAM Identity Center (SSO)** et **Crossplane V2** (resources namespace-scoped, `*.aws.m.upbound.io`).

---

## Architecture multi-comptes

IAM Identity Center est activé **une seule fois** sur le **Management Account**. Les users et Permission Sets sont créés ici, puis les `AccountAssignment` donnent accès aux autres comptes de l'organisation.

```
AWS Organizations
└── Management Account  ←── IAM Identity Center activé ici
    │                        Users + Permission Sets définis ici
    │
    ├── Account RH (111111111111)
    │   └── Marie → ps-rh-billing
    │
    ├── Account Production (222222222222)
    │   └── Alice → ps-devops
    │
    └── Account Staging (333333333333)
        ├── Alice → ps-devops
        └── Bob   → ps-devops
```

Quand un user se connecte via SSO, il voit tous les comptes auxquels il a accès et choisit lequel cibler. Les credentials sont **temporaires** et renouvelés automatiquement.

---

## Providers utilisés

| Provider | Rôle | API Group |
|----------|------|-----------|
| `identitystore` | Création des users humains | `identitystore.aws.m.upbound.io/v1beta1` |
| `ssoadmin` | Permission Sets, attachments, assignments | `ssoadmin.aws.m.upbound.io/v1beta1` |
| `iam` | Policies granulaires custom | `iam.aws.m.upbound.io/v1beta1` |

> **Crossplane V2** : toutes les resources sont **namespace-scoped** et utilisent le groupe `.aws.m.upbound.io`.

---

## Types de permissions

Un Permission Set peut embarquer deux types de policies :

**AWS Managed Policies** → policies gérées par AWS, référencées par ARN via `ManagedPolicyAttachment`

**Customer Managed Policies** → policies IAM custom créées via la resource `iam/Policy`, attachées via `CustomerManagedPolicyAttachment` avec leur `name` et `path`

C'est grâce aux Customer Managed Policies qu'on obtient la **granularité fine** (ex: autoriser modify VPC mais interdire delete).

---

## Structure des dossiers

```
iam/
├── users/                              # Identités (Identity Center)
│   ├── rh-marie.yaml
│   ├── devops-alice.yaml
│   └── devops-bob.yaml
│
├── permission-sets/                    # Conteneurs de permissions (un par rôle)
│   ├── ps-rh-billing.yaml
│   └── ps-devops.yaml
│
├── policies/                           # Policies IAM granulaires custom
│   ├── policy-devops-vpc.yaml          # VPC : modify OK / delete DENY
│   └── policy-devops-ec2.yaml          # EC2 : start/stop OK / terminate DENY
│
├── policy-attachments/                 # Colle entre policies et permission sets
│   ├── ps-rh-billing-policy.yaml       # AWS Managed  : Billing
│   ├── ps-devops-eks.yaml              # AWS Managed  : EKSClusterPolicy
│   ├── ps-devops-vpc.yaml              # Custom       : devops-vpc-no-delete
│   └── ps-devops-ec2.yaml              # Custom       : devops-ec2-no-delete
│
└── assignments/                        # Assignation user <-> permission set <-> compte AWS
    ├── rh-marie-account-rh.yaml        # Marie  → RH       → compte RH
    ├── devops-alice-account-production.yaml  # Alice  → DevOps   → compte Production
    ├── devops-alice-account-staging.yaml     # Alice  → DevOps   → compte Staging
    └── devops-bob-account-staging.yaml       # Bob    → DevOps   → compte Staging
```

---

## Valeurs à remplacer

| Placeholder | Description | Où trouver |
|-------------|-------------|------------|
| `d-xxxxxxxxxx` | Identity Store ID | AWS Console → IAM Identity Center → Settings |
| `ssoins-xxxxxxxx` | SSO Instance ARN | AWS Console → IAM Identity Center → Settings |
| `111111111111` | AWS Account ID - RH | AWS Console → Organizations |
| `222222222222` | AWS Account ID - Production | AWS Console → Organizations |
| `333333333333` | AWS Account ID - Staging | AWS Console → Organizations |

---

## Ordre de déploiement (ArgoCD sync-waves)

```
Wave 1  →  users/               (identités)
Wave 2  →  permission-sets/     (conteneurs de permissions)
Wave 3  →  policies/            (policies IAM custom)
Wave 4  →  policy-attachments/  (attachements policies → permission sets)
Wave 5  →  assignments/         (user → permission set → compte AWS)
```

---

## Opérations courantes

### Ajouter un nouvel utilisateur

1. Copier un fichier dans `users/` et adapter nom/email
2. Créer les assignments dans `assignments/` vers le(s) compte(s) souhaité(s)
3. Merger la PR → ArgoCD applique

### Donner accès à un compte supplémentaire

Créer un fichier dans `assignments/` :

```yaml
apiVersion: ssoadmin.aws.m.upbound.io/v1beta1
kind: AccountAssignment
metadata:
  name: devops-alice-account-production2
  namespace: platform
spec:
  forProvider:
    instanceArn: arn:aws:sso:::instance/ssoins-xxxxxxxx
    permissionSetArnSelector:
      matchLabels:
        crossplane.io/name: ps-devops
    principalType: USER
    principalIdSelector:
      matchLabels:
        crossplane.io/name: devops-alice
    targetType: AWS_ACCOUNT
    targetId: "444444444444"
  providerConfigRef:
    name: aws-provider
```

### Ajouter des droits à un Permission Set

**AWS Managed Policy :**

```yaml
apiVersion: ssoadmin.aws.m.upbound.io/v1beta1
kind: ManagedPolicyAttachment
metadata:
  name: ps-devops-s3
  namespace: platform
spec:
  forProvider:
    instanceArn: arn:aws:sso:::instance/ssoins-xxxxxxxx
    permissionSetArnSelector:
      matchLabels:
        crossplane.io/name: ps-devops
    managedPolicyArn: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
  providerConfigRef:
    name: aws-provider
```

**Customer Managed Policy (granulaire) :**

1. Créer la policy dans `policies/` avec les actions Allow/Deny précises
2. Créer l'attachment dans `policy-attachments/` via `CustomerManagedPolicyAttachment`
3. Merger la PR → ArgoCD applique

### Retirer des droits

Supprimer le fichier d'attachment ou d'assignment concerné + merger la PR.

---

## Connexion CLI pour les DevOps

**Configuration initiale (une seule fois) :**

```bash
aws configure sso
# SSO Start URL : https://mathod.awsapps.com/start
# SSO Region    : eu-west-1
# Profile name  : devops
```

**Connexion :**

```bash
aws sso login --profile devops
```

**Ajouter un cluster EKS au kubeconfig local :**

```bash
aws eks update-kubeconfig \
  --name mon-cluster \
  --region eu-west-1 \
  --profile devops
```

Pas d'access key statique. Les credentials sont temporaires et renouvelés automatiquement via SSO.