# Crossplane v2 - EKS Managed Resources

Ressources managées Crossplane v2 pour déployer un cluster EKS complet sur AWS.
Inspiré du tutoriel [Anton Putra #176](https://github.com/antonputra/tutorials/tree/main/lessons/176/3-eks).

## Architecture déployée

```
VPC (10.0.0.0/16)
├── Subnet Public A (10.0.1.0/24) - eu-west-1a
├── Subnet Public B (10.0.2.0/24) - eu-west-1b
├── Subnet Private A (10.0.3.0/24) - eu-west-1a
├── Subnet Private B (10.0.4.0/24) - eu-west-1b
├── Internet Gateway
├── NAT Gateway A (EIP) -> Subnet Public A
├── NAT Gateway B (EIP) -> Subnet Public B
├── Route Tables Public (-> IGW)
└── Route Tables Private (-> NAT)

IAM
├── eks-cluster-role (AmazonEKSClusterPolicy)
├── eks-node-role (WorkerNode + CNI + ECR)
└── eks-cluster-autoscaler-role (IRSA)

EKS Cluster: eks-prod (v1.31)
├── Node Group: eks-ng-general (ON_DEMAND, t3.medium, 1-10)
├── Node Group: eks-ng-spot (SPOT, t3.medium/large, 0-10)
└── Add-ons: coredns, kube-proxy, vpc-cni, aws-ebs-csi-driver
```

## Prérequis

- Crossplane v2 installé dans le cluster
- Kubernetes 1.28+
- Credentials AWS avec les permissions nécessaires

## Ordre de déploiement

```bash
# 1. Installer les providers et configurer l'auth
kubectl apply -f 00-providers.yaml

# Créer le secret AWS (si pas IRSA)
cat > aws-credentials.txt <<EOF
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

kubectl create secret generic aws-credentials \
  -n crossplane-system \
  --from-file=credentials=./aws-credentials.txt

# Attendre que les providers soient HEALTHY
kubectl get providers

# 2. Déployer le VPC et le networking
kubectl apply -f 01-vpc.yaml

# Attendre que le VPC soit READY
kubectl wait vpc/eks-vpc --for=condition=Ready --timeout=300s

# 3. Déployer les rôles IAM
kubectl apply -f 02-iam.yaml

# 4. Déployer le cluster EKS
kubectl apply -f 03-eks.yaml

# Attendre que le cluster soit prêt (~15-20 min)
kubectl wait cluster.eks/eks-prod --for=condition=Ready --timeout=1200s
```

## Vérification

```bash
# Status global
kubectl get managed

# Status du cluster EKS
kubectl describe cluster.eks/eks-prod

# Récupérer le kubeconfig
aws eks update-kubeconfig --region eu-west-1 --name eks-prod
```

## Personnalisation

| Variable | Fichier | Description |
|----------|---------|-------------|
| `region` | tous | Région AWS cible |
| `cidrBlock` | 01-vpc.yaml | CIDR du VPC |
| `version` | 03-eks.yaml | Version Kubernetes |
| `instanceTypes` | 03-eks.yaml | Types d'instances |
| `ACCOUNT_ID` | 02-iam.yaml | ID du compte AWS pour IRSA |
| `OIDC_ID` | 02-iam.yaml | OIDC provider ID (après création cluster) |

## Notes importantes

- **IRSA** : Après création du cluster, mettre à jour le `assumeRolePolicy` du role `eks-cluster-autoscaler-role` avec les valeurs réelles `ACCOUNT_ID` et `OIDC_ID`.
- **Labels** : Les sélecteurs utilisent des labels `crossplane.io/name` générés automatiquement. Pour les subnets, ajouter manuellement les labels `network-type: eks-subnet` et `network-type: eks-subnet-private`.
- **Ordre** : Respecter l'ordre de déploiement car les ressources dépendent les unes des autres via des `selector`.