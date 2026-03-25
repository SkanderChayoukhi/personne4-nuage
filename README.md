Personne 4 — Redis + DB + Infrastructure GKE
Rôle dans l'architecture
Redis (stockage temporaire des votes) et PostgreSQL (stockage permanent des résultats) forment la couche données. Cette personne gère aussi toute l'infrastructure GKE et Terraform GCP.

Docker
docker-compose.yaml — pas de Dockerfile, images directes :

redis:alpine sur back-net, volume bind pour les healthchecks
postgres:15-alpine sur back-net, volume db-data pour la persistance
Variables d'environnement : POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
Healthchecks via scripts shell (healthchecks/redis.sh, healthchecks/postgres.sh)
Healthchecks :

redis.sh : exécute redis-cli ping, vérifie que la réponse est PONG
postgres.sh : exécute SELECT 1 via psql, vérifie que le résultat est 1
Kubernetes
redis-deployment.yaml (k8s/redis-deployment.yaml) :

InitContainer busybox:1.36 qui écrit le script redis.sh dans un volume emptyDir
Container principal redis:alpine avec livenessProbe exec sur le script
Astuce : pas besoin de créer une image custom pour Redis, l'initContainer injecte le healthcheck
redis-service.yaml (k8s/redis-service.yaml) :

Type ClusterIP — Redis n'est JAMAIS exposé à l'extérieur
Port 6379, accessible uniquement par vote et worker via le DNS interne redis
db-deployment.yaml (k8s/db-deployment.yaml) :

Même pattern initContainer pour le healthcheck postgres
envFrom: configMapRef: app-config pour les credentials
Volume PVC monté sur /var/lib/postgresql/data avec subPath: data
db-pvc.yaml (k8s/db-pvc.yaml) :

PersistentVolumeClaim de 1Gi, ReadWriteOnce
Garantit que les données survivent au redémarrage/suppression du pod
db-service.yaml (k8s/db-service.yaml) :

Type ClusterIP — PostgreSQL n'est JAMAIS exposé à l'extérieur
Port 5432, accessible par result et worker via le DNS interne db
app-configmap.yaml (k8s/app-configmap.yaml) :

ConfigMap centralisée avec toutes les variables : options de vote, credentials DB, hosts
Utilisée par tous les services via envFrom
Kustomize (k8s/kustomize/kustomization.yaml) :

Référence tous les manifests + configMapGenerator pour générer le ConfigMap
Alternative au manifest manuel, génère un suffixe de hash pour forcer le redéploiement si les valeurs changent
Terraform Docker (terraform/modules/redis/ et terraform/modules/postgres/)
Redis : image redis:alpine, healthcheck, réseau back-net, bind mount healthchecks
Postgres : image postgres:15-alpine, healthcheck, réseau back-net, volume db-data
Terraform GCP (terraform-gcp/)
gke.tf :

Crée un cluster GKE avec 1 noeud e2-medium dans europe-west3-a
deletion_protection = false pour pouvoir détruire avec terraform destroy
Node pool séparé du cluster (best practice GKE)
gke-config.tf :

Configure le provider Kubernetes avec le token OAuth2 du cluster GKE
depends_on sur le node pool pour s'assurer que le cluster est prêt
k8s-manifests.tf :

Lit tous les YAML de k8s/ et les applique via kubernetes_manifest
Injecte le namespace et le mot de passe PostgreSQL dynamiquement
Si enable_redis_vm = true : exclut redis-deployment.yaml et transforme le service Redis en headless
redis-vm.tf (optionnel Part 3) :

VM e2-micro Debian 12 avec script cloud-init qui installe Redis
Configure Redis pour écouter sur 0.0.0.0 avec mot de passe
kubernetes_manifest.redis_endpoints crée un Endpoint manuel pointant vers l'IP interne de la VM
variables.tf :

gcp_project_id, gcp_zone, gke_cluster_name, gke_node_count
postgres_password (sensitive), redis_password (sensitive)
enable_redis_vm : toggle pour activer/désactiver la VM Redis
manifest_files : liste des YAML à appliquer
Points clés à mentionner
ClusterIP vs LoadBalancer : Redis et DB sont TOUJOURS ClusterIP (sécurité)
PVC : sans le subPath: data, PostgreSQL échoue car le répertoire n'est pas vide
InitContainer : pattern élégant pour injecter des scripts sans créer d'images custom
ConfigMap : centralise la config, un seul endroit à modifier
Redis VM (Part 3) : découplage de Redis du cluster, service headless + Endpoints manuels
terraform destroy supprime tout le cluster et les ressources associées
