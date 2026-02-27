# Infrastructure IAM & Organizations - mathod.io

Gestion complète des accès AWS via **Crossplane V2** (resources namespace-scoped, `*.aws.m.upbound.io`) et **GitOps ArgoCD**.

---

## Architecture

![Architecture AWS Organizations](./aws-organizations-diagram.svg)

```
AWS Organizations
└── Root
    ├── OU: Management
    │   └── Compte: Management (existant) - Crossplane, ArgoCD, Identity Center
    │
    ├── OU: Environments
    │   ├── OU: Production
    │   │   └── Compte: Production
    │   └── OU: Non-Production
    │       ├── Compte: Staging
    │       └── Compte: Dev
    │
    ├── OU: Business
    │   └── Compte: RH
    │
    └── OU: Security
        └── Compte: Audit
```

### Matrice des accès

| Utilisateur | Rôle | Compte(s) accessible(s) | Permission Set |
|-------------|------|--------------------------|----------------|
| Mathod | Admin | Tous | ps-admin (AdministratorAccess) |
| Alice | DevOps | Production, Staging, Dev | ps-devops |
| Bob | DevOps | Production, Staging, Dev | ps-devops |
| Marie | RH | RH | ps-rh-billing |

### SCPs en place

| SCP | Cible | Effet |
|-----|-------|-------|
| DenyNonEURegions | Root (tous) | Bloque toute action hors eu-west-1 et eu-west-3 |
| DenyRootAccount | Root (tous) | Bloque toute action avec le compte root |
| DenyLeaveOrganization | Root (tous) | Empeche un compte de quitter l'org |
| DenyDeleteProduction | OU Production | Bloque la suppression de ressources critiques |

---

## Providers utilisés

| Provider | Rôle | API Group |
|----------|------|-----------|
| `organizations` | OUs, comptes, SCPs | `organizations.aws.m.upbound.io/v1beta1` |
| `identitystore` | Users humains (SSO) | `identitystore.aws.m.upbound.io/v1beta1` |
| `ssoadmin` | Permission Sets, attachments, assignments | `ssoadmin.aws.m.upbound.io/v1beta1` |
| `iam` | Policies granulaires custom | `iam.aws.m.upbound.io/v1beta1` |

---

## Structure des fichiers

```
.
├── organizations/
│   ├── ous/
│   │   ├── ou-management.yaml
│   │   ├── ou-environments.yaml
│   │   ├── ou-production.yaml         # enfant de ou-environments
│   │   ├── ou-non-production.yaml     # enfant de ou-environments
│   │   ├── ou-business.yaml
│   │   └── ou-security.yaml
│   │
│   ├── accounts/
│   │   ├── account-production.yaml    # dans ou-production
│   │   ├── account-staging.yaml       # dans ou-non-production
│   │   ├── account-dev.yaml           # dans ou-non-production
│   │   ├── account-rh.yaml            # dans ou-business
│   │   └── account-audit.yaml         # dans ou-security
│   │
│   ├── scps/
│   │   ├── scp-deny-non-eu-regions.yaml
│   │   ├── scp-deny-root-account.yaml
│   │   ├── scp-deny-leave-org.yaml
│   │   └── scp-deny-delete-production.yaml
│   │
│   └── scp-attachments/
│       ├── attach-deny-non-eu-regions-root.yaml    # Root
│       ├── attach-deny-root-account-root.yaml      # Root
│       ├── attach-deny-leave-org-root.yaml         # Root
│       └── attach-deny-delete-production-ou.yaml   # OU Production
│
└── iam/
    ├── users/
    │   ├── admin-mathod.yaml
    │   ├── devops-alice.yaml
    │   ├── devops-bob.yaml
    │   └── rh-marie.yaml
    │
    ├── permission-sets/
    │   ├── ps-admin.yaml
    │   ├── ps-devops.yaml
    │   └── ps-rh-billing.yaml
    │
    ├── policies/                        # Policies IAM granulaires custom
    │   ├── policy-devops-vpc.yaml       # VPC : modify OK / delete DENY
    │   └── policy-devops-ec2.yaml       # EC2 : start/stop OK / terminate DENY
    │
    ├── policy-attachments/
    │   ├── ps-admin-policy.yaml         # AWS Managed : AdministratorAccess
    │   ├── ps-rh-billing-policy.yaml    # AWS Managed : Billing
    │   ├── ps-devops-eks.yaml           # AWS Managed : EKSClusterPolicy
    │   ├── ps-devops-vpc.yaml           # Custom      : devops-vpc-no-delete
    │   └── ps-devops-ec2.yaml           # Custom      : devops-ec2-no-delete
    │
    └── assignments/
        ├── admin-mathod-account-management.yaml
        ├── admin-mathod-account-production.yaml
        ├── admin-mathod-account-staging.yaml
        ├── admin-mathod-account-dev.yaml
        ├── admin-mathod-account-rh.yaml
        ├── admin-mathod-account-audit.yaml
        ├── devops-alice-account-production.yaml
        ├── devops-alice-account-staging.yaml
        ├── devops-alice-account-dev.yaml
        ├── devops-bob-account-production.yaml
        ├── devops-bob-account-staging.yaml
        ├── devops-bob-account-dev.yaml
        └── rh-marie-account-rh.yaml
```

---

## Prérequis manuels (bootstrap)

Ces 3 actions sont à faire **une seule fois** avant tout déploiement Crossplane.

### 1. Créer l'Organization AWS

```bash
aws organizations create-organization --feature-set ALL
```

### 2. Activer les SCPs

```bash
aws organizations enable-policy-type \
  --root-id $(aws organizations list-roots --query 'Roots[0].Id' --output text) \
  --policy-type SERVICE_CONTROL_POLICY
```

### 3. Activer IAM Identity Center

AWS Console → IAM Identity Center → **Enable**. Doit être fait depuis le Management Account.

---

## Valeurs à remplacer

| Placeholder | Description | Commande |
|-------------|-------------|----------|
| `r-xxxx` | Root ID | `aws organizations list-roots --query 'Roots[0].Id' --output text` |
| `d-xxxxxxxxxx` | Identity Store ID | `aws sso-admin list-instances --query 'Instances[0].IdentityStoreId' --output text` |
| `ssoins-xxxxxxxx` | SSO Instance ARN | `aws sso-admin list-instances --query 'Instances[0].InstanceArn' --output text` |
| `ACCOUNT_ID_MANAGEMENT` | ID compte Management | `aws sts get-caller-identity --query Account --output text` |
| `ACCOUNT_ID_PRODUCTION` | ID compte Production | Disponible après création via Crossplane |
| `ACCOUNT_ID_STAGING` | ID compte Staging | Disponible après création via Crossplane |
| `ACCOUNT_ID_DEV` | ID compte Dev | Disponible après création via Crossplane |
| `ACCOUNT_ID_RH` | ID compte RH | Disponible après création via Crossplane |
| `ACCOUNT_ID_AUDIT` | ID compte Audit | Disponible après création via Crossplane |

> Les IDs des comptes enfants sont disponibles dans la console AWS Organizations après leur création par Crossplane, ou via `aws organizations list-accounts`.

---

## Ordre de déploiement (ArgoCD sync-waves)

```
Wave 1  → organizations/ous/             (OUs racines : management, environments, business, security)
Wave 2  → organizations/ous/             (OUs enfants : production, non-production)
Wave 3  → organizations/accounts/        (comptes AWS enfants)
Wave 4  → organizations/scps/            (Service Control Policies)
Wave 5  → organizations/scp-attachments/ (attachement SCPs sur Root et OUs)
Wave 6  → iam/users/                     (identités SSO)
Wave 7  → iam/permission-sets/           (conteneurs de permissions)
Wave 8  → iam/policies/                  (policies IAM granulaires)
Wave 9  → iam/policy-attachments/        (attachement policies -> permission sets)
Wave 10 → iam/assignments/              (user -> permission set -> compte AWS)
```

> Note : les waves 1 et 2 doivent être séparées car les OUs enfants dépendent des OUs parents.
> Récupérer les ACCOUNT_IDs entre la wave 3 et la wave 10.

---

## Opérations courantes

### Ajouter un utilisateur

1. Copier un fichier dans `iam/users/` en adaptant nom et email
2. Créer les assignments dans `iam/assignments/` vers le ou les comptes voulus
3. Merger la PR → ArgoCD applique

### Donner accès à un compte supplémentaire

Créer un fichier dans `iam/assignments/` en pointant vers le `targetId` du compte cible.

### Ajouter des droits à un Permission Set

**AWS Managed Policy :** créer un `ManagedPolicyAttachment` dans `iam/policy-attachments/`

**Policy granulaire custom :**
1. Créer la policy dans `iam/policies/` avec les `Allow` et `Deny` précis
2. Créer un `CustomerManagedPolicyAttachment` dans `iam/policy-attachments/`
3. Merger la PR → ArgoCD applique

### Retirer des droits ou un accès

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

**Ajouter un cluster EKS au kubeconfig :**

```bash
aws eks update-kubeconfig \
  --name mon-cluster \
  --region eu-west-1 \
  --profile devops
```

Pas d'access key statique. Les credentials sont temporaires et renouvelés automatiquement via SSO.
