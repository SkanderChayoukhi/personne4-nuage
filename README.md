# Pitch oral - Personne 4 (Terraform GCP + démonstration finale)

## Mon role (Personne 4)
Je présente la partie Infrastructure as Code sur GCP avec Terraform, puis je fais la validation finale de l'application en conditions d'examen.

Mes objectifs oraux:
- Montrer que l'infra est reproductible (cluster GKE + manifests Kubernetes)
- Montrer la valeur de la Part 3 (Redis externalise sur VM)
- Montrer une verification factuelle avec commandes et outputs

---

## Commande de contexte cluster (a dire + executer)
Avant toute verification Kubernetes, je recupere le contexte du cluster:

```bash
gcloud container clusters get-credentials projet-nuage-tf --zone europe-west3-a --project lab-gcp-kube
```

Phrase a dire:
"Je connecte mon client kubectl au cluster cree par Terraform pour verifier l'etat reel des ressources." 

---

## Script oral pret a dire (3 a 5 minutes)

### 1) Introduction (20-30s)
"Je suis la personne 4. Je couvre la partie Terraform GCP: creation du cluster, deploiement des manifests Kubernetes, puis extension Part 3 avec Redis sur VM. L'objectif est de montrer une architecture reproductible et verifiable par commandes."

### 2) Methode Terraform (30-40s)
"On a separe le projet en deux couches Terraform: une couche locale Docker pour la partie 1, puis une couche GCP/Kubernetes pour les parties 2 et 3. La couche GCP utilise le provider Google pour le cluster et le provider Kubernetes pour appliquer les manifests via `kubernetes_manifest`."

### 3) Part 2 - cluster + application (50-70s)
"D'abord, Terraform cree le cluster GKE (1 node, conforme au sujet), puis le node pool. Ensuite, les manifests applicatifs sont appliques: vote, result, worker, db, redis, services, PVC, job seed et HPA."

"Je valide ensuite le resultat avec `kubectl get nodes` et `kubectl get pods,svc,pvc,hpa,jobs -o wide` pour prouver que l'etat runtime correspond a l'etat declare."

### 4) Part 3 - Redis externalise (60-90s)
"La valeur ajoutee Part 3 est de decoupler Redis du cluster: Redis tourne sur une VM GCP dediee, securisee par mot de passe."

"Techniquement, Terraform:
- cree la VM Redis,
- retire le deployment Redis Kubernetes,
- transforme le service Redis en service headless,
- cree un objet Endpoints qui pointe vers l'IP privee de la VM.

Ainsi, les applications continuent d'utiliser le nom logique `redis`, mais le backend reel est externe au cluster."

### 5) Verification de preuve (40-60s)
"Je montre trois preuves:
- `terraform output redis_vm_internal_ip` donne l'IP privee de la VM,
- `kubectl get svc redis` montre `CLUSTER-IP None` (headless),
- `kubectl get endpoints redis -o wide` montre l'IP de la VM Redis.

Ces trois points prouvent que l'externalisation Redis est effective."

### 6) Conclusion (20-30s)
"En conclusion, on a une chaine complete: local Terraform valide, GKE provisionne par Terraform, deploiement applicatif Kubernetes, puis extension d'architecture avec Redis externalise. Toute la demo est reproductible et pilotable par IaC."

---

## Quoi montrer dans le code pendant que je parle

## A. Creation du cluster GKE
Fichier: `terraform-gcp/gke.tf`
A montrer:
- resource `google_container_cluster`
- node pool avec 1 node
- parametres de localisation (zone)

Phrase a dire:
"Ici, on declare le cluster et son node pool de facon declarative, sans action manuelle sur la console GCP."

## B. Application des manifests Kubernetes
Fichier: `terraform-gcp/k8s-manifests.tf`
A montrer:
- resource `kubernetes_manifest` pour les manifests applicatifs
- logique de namespace
- `computed_fields` pour stabiliser le comportement provider

Phrase a dire:
"On reutilise nos YAML Kubernetes existants, ce qui garde une source de verite unique entre k8s et Terraform."

## C. Part 3 Redis VM
Fichier: `terraform-gcp/redis-vm.tf`
A montrer:
- resource `google_compute_instance.redis_vm`
- script startup qui installe/configure redis + mot de passe

Phrase a dire:
"Redis n'est plus un pod; il devient un service infra dedie sur VM."

## D. Routage Kubernetes vers la VM Redis
Fichier: `terraform-gcp/k8s-manifests.tf`
A montrer:
- service Redis headless
- resource `kubernetes_manifest.redis_endpoints`

Phrase a dire:
"Le nom DNS interne `redis` reste stable pour l'application, mais l'endpoint pointe maintenant vers la VM externe."

## E. Cote application (compatibilite Part 3)
Fichiers:
- `vote/app.py`
- `worker/Program.cs`

A montrer:
- lecture de `REDIS_HOST`
- lecture de `REDIS_PASSWORD`

Phrase a dire:
"Le code applicatif est parametre par variables d'environnement, donc la migration infra se fait sans casser la logique metier."

---

## Sequence de demo conseillee (checklist rapide)
1. `terraform -chdir=terraform-gcp plan`
2. `terraform -chdir=terraform-gcp apply -auto-approve -target="google_container_cluster.primary" -target="google_container_node_pool.primary"`
3. `gcloud container clusters get-credentials projet-nuage-tf --zone europe-west3-a --project lab-gcp-kube`
4. `terraform -chdir=terraform-gcp apply -auto-approve`
5. `kubectl get nodes`
6. `kubectl get pods,svc,pvc,hpa,jobs -o wide`
7. `terraform -chdir=terraform-gcp output redis_vm_internal_ip`
8. `kubectl get svc redis`
9. `kubectl get endpoints redis -o wide`

---

## Reponses courtes aux questions possibles

Q: Pourquoi Terraform + YAML Kubernetes ?
R: "Terraform gere le cycle de vie infra complet, et `kubernetes_manifest` permet de reutiliser nos YAML sans duplication."

Q: Pourquoi un apply cible avant apply complet ?
R: "Pour eviter les erreurs de client Kubernetes avant que le cluster n'existe et que le kubeconfig soit charge."

Q: Comment prouver la Part 3 ?
R: "IP Redis VM dans output Terraform + service headless + endpoints Kubernetes vers cette IP."

Q: Si un pod reste Pending ?
R: "Sur cluster 1-node, c'est une contrainte capacite (CPU/memoire), pas un echec IaC."

---

## Closing final (10 secondes)
"Notre architecture est entierement scriptable, reproductible et verifiable: c'est exactement l'objectif DevOps du projet."
