# ArgoCD Sync Waves — Guide débutant

## C'est quoi une sync wave ?

Quand ArgoCD déploie des ressources, il peut tout déployer **en même temps**. Ça pose un problème : certaines ressources dépendent d'autres qui n'existent pas encore.

**Exemple concret :** Tu ne peux pas attacher une SCP à une OU qui n'existe pas encore. Si ArgoCD essaie de créer les deux en même temps, l'attachment va échouer.

Les **sync waves** résolvent ce problème en disant à ArgoCD :
> "Attends que tout ce qui est en wave 1 soit prêt avant de passer à la wave 2."

---

## Comment ça marche ?

C'est une simple annotation Kubernetes ajoutée dans le `metadata` de chaque manifest :

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

- La valeur est un **nombre entier** (peut être négatif)
- ArgoCD déploie dans l'ordre croissant : wave `0` avant wave `1`, avant wave `2`, etc.
- ArgoCD attend que **toutes les ressources d'une wave soient `Healthy`** avant de passer à la suivante
- Par défaut, sans annotation, une ressource est en wave `0`

---

## Notre architecture et pourquoi cet ordre

Voici l'ordre de déploiement de notre infrastructure AWS Organizations + IAM Identity Center, et la raison de chaque étape.

### Wave 0 — Organization

```yaml
# organizations/organization.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

**Pourquoi en premier ?** C'est la racine de tout. Sans Organization, rien d'autre ne peut exister. Tout le reste dépend d'elle.

---

### Wave 1 — OUs racines (Management, Environments, Business, Security)

```yaml
# organizations/ous/ou-management.yaml
# organizations/ous/ou-environments.yaml
# organizations/ous/ou-business.yaml
# organizations/ous/ou-security.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

**Pourquoi ?** Ces OUs sont attachées directement au Root. Elles doivent exister avant leurs enfants (Production, Non-Production).

---

### Wave 2 — OUs enfants (Production, Non-Production)

```yaml
# organizations/ous/ou-production.yaml
# organizations/ous/ou-non-production.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

**Pourquoi après wave 1 ?** Ces OUs référencent `ou-environments` comme parent via un `parentIdSelector`. Si `ou-environments` n'existe pas encore, Crossplane ne peut pas résoudre la référence.

---

### Wave 3 — Comptes AWS (Production, Staging, Dev, RH, Audit)

```yaml
# organizations/accounts/account-production.yaml
# organizations/accounts/account-staging.yaml
# ... etc
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"
```

**Pourquoi ?** Les comptes référencent leurs OUs parentes. L'OU Production doit exister avant de pouvoir y placer le compte Production.

> ⚠️ **Note importante :** Après cette wave, AWS met quelques minutes à provisionner les comptes. Il faudra récupérer manuellement les Account IDs via `aws organizations list-accounts` pour les renseigner dans les assignments (wave 10) avant le déploiement final.

---

### Wave 4 — Service Control Policies (SCPs)

```yaml
# organizations/scps/scp-deny-non-eu-regions.yaml
# organizations/scps/scp-deny-root-account.yaml
# organizations/scps/scp-deny-leave-org.yaml
# organizations/scps/scp-deny-delete-production.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "4"
```

**Pourquoi ?** Les SCPs sont des ressources indépendantes — elles n'ont pas besoin des OUs pour exister. Par contre elles doivent exister AVANT d'être attachées (wave 5).

---

### Wave 5 — Attachements des SCPs

```yaml
# organizations/scp-attachments/attach-deny-non-eu-regions-root.yaml
# organizations/scp-attachments/attach-deny-delete-production-ou.yaml
# ... etc
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

**Pourquoi ?** Un `PolicyAttachment` relie une SCP à une cible (Root ou OU). Les deux doivent exister en amont.

---

### Wave 6 — Utilisateurs IAM Identity Center

```yaml
# iam/users/admin-mathod.yaml
# iam/users/devops-alice.yaml
# iam/users/devops-bob.yaml
# iam/users/rh-marie.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "6"
```

**Pourquoi ?** Les utilisateurs sont indépendants des Permission Sets, mais ils doivent exister avant les assignments (wave 10) qui les référencent via `principalIdSelector`.

---

### Wave 7 — Permission Sets

```yaml
# iam/permission-sets/ps-admin.yaml
# iam/permission-sets/ps-devops.yaml
# iam/permission-sets/ps-rh-billing.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "7"
```

**Pourquoi ?** Un Permission Set est le "conteneur" qui va recevoir les policies (wave 8-9) et être assigné aux utilisateurs (wave 10). Il doit exister avant tout ça.

---

### Wave 8 — Policies IAM custom

```yaml
# iam/policies/policy-devops-vpc.yaml
# iam/policies/policy-devops-ec2.yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "8"
```

**Pourquoi ?** Ces policies granulaires (VPC, EC2) doivent exister dans AWS IAM avant d'être attachées aux Permission Sets via `CustomerManagedPolicyAttachment` (wave 9).

---

### Wave 9 — Attachements des policies aux Permission Sets

```yaml
# iam/policy-attachments/ps-admin-policy.yaml
# iam/policy-attachments/ps-devops-eks.yaml
# iam/policy-attachments/ps-devops-vpc.yaml
# ... etc
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "9"
```

**Pourquoi ?** On attache ici les policies (AWS managées ou custom) à chaque Permission Set. Les deux doivent exister en amont.

---

### Wave 10 — Assignments (utilisateur → Permission Set → compte AWS)

```yaml
# iam/assignments/devops-alice-account-production.yaml
# iam/assignments/admin-mathod-account-management.yaml
# ... etc
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "10"
```

**Pourquoi en dernier ?** C'est l'étape finale qui "connecte" tout :
- Un **utilisateur** (wave 6) ✓
- Un **Permission Set** (wave 7) avec ses policies (wave 9) ✓
- Un **compte AWS** (wave 3) ✓

Tout doit être prêt avant de créer l'assignment.

---

## Résumé visuel

```
Wave 0  ──→  Organization
              │
Wave 1  ──→  OUs racines (Management, Environments, Business, Security)
              │
Wave 2  ──→  OUs enfants (Production, Non-Production)
              │
Wave 3  ──→  Comptes AWS  ←── récupérer les Account IDs ici
              │
Wave 4  ──→  SCPs (policies de contrôle)
              │
Wave 5  ──→  Attachements SCPs → OUs / Root
              │
Wave 6  ──→  Utilisateurs SSO (Marie, Alice, Bob, Mathod)
              │
Wave 7  ──→  Permission Sets (RH-Billing, DevOps, Admin)
              │
Wave 8  ──→  Policies IAM custom (VPC, EC2)
              │
Wave 9  ──→  Attachements policies → Permission Sets
              │
Wave 10 ──→  Assignments (user + permission set + compte)
```

---

## Comment ArgoCD sait qu'une wave est "prête" ?

ArgoCD attend que chaque ressource soit dans l'état **`Healthy`**. Pour les ressources Crossplane, ça veut dire que la condition `Synced: True` et `Ready: True` sont présentes.

Si une ressource reste en `Progressing` ou passe en `Degraded`, ArgoCD **s'arrête** et n'avance pas à la wave suivante. C'est un filet de sécurité : une erreur en wave 3 ne provoquera jamais un assignment cassé en wave 10.

---

## Vérifier l'état d'avancement

```bash
# Voir toutes les ressources Crossplane et leur état
kubectl get managed -n platform

# Voir les events ArgoCD sur une application
argocd app get infra-organizations

# Voir les logs d'une ressource Crossplane spécifique
kubectl describe organizationalunit ou-production -n platform
```

---

## Points d'attention

**Pause entre wave 3 et wave 10**

AWS prend quelques minutes à créer les comptes. Crossplane va attendre que le compte soit `Ready` avant de passer à la suite, mais les `targetId` dans les assignments doivent être renseignés manuellement. Il faut donc :

1. Laisser ArgoCD déployer jusqu'à la wave 3
2. Récupérer les Account IDs : `aws organizations list-accounts`
3. Mettre à jour les fichiers d'assignments avec les vrais IDs
4. Merger la PR → ArgoCD termine les waves 4 à 10

**Rollback**

Si tu supprimes un fichier d'assignment, ArgoCD va supprimer l'assignment dans AWS à la prochaine sync. C'est le principe du GitOps : l'état Git = l'état réel.

**`prune: false` recommandé pour les Organizations**

Pour les ressources Organizations et les comptes AWS, il est conseillé de désactiver le pruning automatique dans l'Application ArgoCD pour éviter la suppression accidentelle d'un compte AWS.

---

Vue que c'est déployer avec crossplane quoi qu'il arrive tous passeras healthy à un moment donné grace au reconciliation non ?

Oui et non ! C'est une bonne réflexion.
Ce que tu dis est vrai pour les ressources Crossplane "pures" — si une OU n'existe pas encore quand Crossplane essaie de créer un compte dedans, Crossplane va retry en boucle via son reconciliation loop jusqu'à ce que l'OU soit là. Il finira par réussir.
Mais le vrai risque c'est ailleurs : les sync waves ArgoCD ne servent pas à "attendre que ça marche", elles servent à contrôler l'ordre dans lequel ArgoCD soumet les ressources à Kubernetes.
Le problème concret sans sync waves :
ArgoCD envoie tout en même temps
→ Crossplane reçoit l'AccountAssignment
→ il cherche l'utilisateur via principalIdSelector
→ l'utilisateur n'existe pas encore dans le cluster Kubernetes
→ Crossplane est en erreur, il retry...
→ pendant ce temps ArgoCD marque l'app comme Degraded
→ si tu as des alertes ou des pipelines qui attendent, c'est le chaos
Donc en pratique sans sync waves ça finit par marcher, mais tu auras :

Des erreurs et des états Degraded temporaires dans ArgoCD
Des logs pollués avec des erreurs de résolution de références
Des pipelines CI/CD qui peuvent fail si ils attendent un état Healthy
Un ordre de déploiement non déterministe qui rend le debug difficile

C'est surtout une question de propreté opérationnelle plutôt que de blocage dur. Si c'est un déploiement one-shot manuel sans pipeline, tu peux t'en passer. Si c'est en prod avec du GitOps automatisé, les sync waves évitent beaucoup de bruit.

---

Terraform lui gère ce type de priorité ?10:54Oui, Terraform gère ça nativement et automatiquement via son dependency graph.
Quand tu écris du Terraform, il analyse les références entre ressources et construit un graphe de dépendances tout seul :
hclresource "aws_organizations_organizational_unit" "production" {
  name      = "Production"
  parent_id = aws_organizations_organizational_unit.environments.id  # ← référence
}
Terraform voit que production dépend de environments, donc il attend que environments soit créé avant de créer production. C'est implicite, tu n'as rien à déclarer.
Tu peux aussi forcer une dépendance explicite avec depends_on quand il n'y a pas de référence directe :
hclresource "aws_organizations_policy_attachment" "scp" {
  depends_on = [aws_organizations_policy.scp]  # ← dépendance explicite
}

La différence fondamentale avec Crossplane :
TerraformCrossplaneGestion des dépendancesAutomatique via le graphManuel via sync waves ou selectorsExécutionOne-shot (terraform apply)Reconciliation loop en continuÉtatFichier tfstateRessources KubernetesErreur de dépendanceBloqué proprementRetry en boucle jusqu'au succès
C'est d'ailleurs un des arguments souvent avancés en faveur de Terraform sur ce point — le modèle déclaratif + graph est plus simple à raisonner que les sync waves manuelles. Crossplane compense par sa nature Kubernetes-native et sa réconciliation continue, mais la gestion des dépendances est clairement moins élégante.