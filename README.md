# Guide d'utilisation des Sealed Secrets pour GitHub PAT

Ce guide explique comment gérer de manière sécurisée votre Personal Access Token (PAT) GitHub avec Sealed Secrets dans Kubernetes.

## Prérequis

1. Installation de kubeseal :
```bash
# Homebrew (macOS)
brew install kubeseal

# Linux
# Télécharger la dernière version depuis :
# https://github.com/bitnami-labs/sealed-secrets/releases
```

2. Installation du controller Sealed Secrets dans votre cluster :
```yaml
# Dans votre values-dev.yaml d'ArgoCD
sealed-secrets:
  enabled: true
  version: "2.14.1"
  url: https://bitnami-labs.github.io/sealed-secrets

apps:
  - name: sealed-secrets
    namespace: sealed-secrets
    project: infrastructure
    source:
      chart: sealed-secrets
      repoURL: https://bitnami-labs.github.io/sealed-secrets
      targetRevision: 2.14.1
```

## Étapes de configuration

### 1. Création du template
Créez le fichier `umbrella-optionsviz/templates/technical-secret.yaml` :
```yaml
{{- if eq .Values.technicalSecret.type "sealedSecret" }}
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: technical-secret
  labels:
    {{- include "label-generator" . | nindent 4 }}
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
spec:
  encryptedData:
    .dockerconfigjson: {{ .Values.technicalSecret.githubSecret | quote}}
  template:
    type: kubernetes.io/dockerconfigjson
{{- else }}
# ... le reste du template pour les autres cas
{{- end }}
```

### 2. Création du secret Docker Registry

```bash
# Créer le fichier de configuration Docker
echo -n '{
  "auths": {
    "ghcr.io": {
      "username": "VOTRE_USERNAME",
      "password": "VOTRE_PAT",
      "auth": "'$(echo -n "VOTRE_USERNAME:VOTRE_PAT" | base64)'"
    }
  }
}' > docker-config.json
```

### 3. Génération du Sealed Secret

```bash
# Créer le secret Kubernetes
kubectl create secret docker-registry technical-secret \
  --from-file=.dockerconfigjson=docker-config.json \
  --namespace votre-namespace \
  --dry-run=client -o yaml > temp-secret.yaml

# Créer le sealed secret
kubeseal \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets \
  --format yaml \
  --scope namespace-wide \
  < temp-secret.yaml > sealed-secret.yaml
```

### 4. Configuration des values

Dans `umbrella-optionsviz/values/values.dev.yaml` :
```yaml
technicalSecret:
  type: "sealedSecret"
  githubSecret: "COLLEZ_ICI_LA_VALEUR_ENCRYPTEE"  # Depuis sealed-secret.yaml

optionsviz:
  module: optionsviz
  name: optionsviz
  environment: dev
  containers:
    image: ghcr.io/votre-repo/votre-image
    imagePullSecrets:
      - name: technical-secret
```

### 5. Nettoyage

```bash
rm docker-config.json temp-secret.yaml sealed-secret.yaml
```

## Vérification

```bash
# Vérifier le sealed secret
kubectl get sealedsecret technical-secret

# Vérifier le secret déchiffré
kubectl get secret technical-secret

# Vérifier le contenu
kubectl get secret technical-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

## Important

- ⚠️ Ne jamais commiter les fichiers temporaires (docker-config.json, temp-secret.yaml)
- ⚠️ Sauvegarder la clé du controller sealed-secrets
- 🔄 Créer des secrets différents pour chaque environnement
- 🔒 S'assurer que le PAT a les droits minimums nécessaires

## Troubleshooting

Si `kubeseal` n'est pas trouvé :
```bash
# Sur macOS
brew install kubeseal

# Sur Linux
KUBESEAL_VERSION=0.24.5  # Vérifier la dernière version
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```
