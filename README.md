# Pitch oral - Personne 4 (Redis + DB + Infra GKE)

## Role (ce que je dis en 10 secondes)

"Je suis la personne 4. Je couvre la couche donnees Redis/PostgreSQL et l'infrastructure GKE avec Terraform, y compris l'extension Part 3: Redis externalise sur VM."

---

## Ordre d'ouverture des fichiers pendant l'oral

1. `docker-compose.yaml`
2. `healthchecks/redis.sh` et `healthchecks/postgres.sh`
3. `k8s/redis-deployment.yaml`, `k8s/redis-service.yaml`
4. `k8s/db-deployment.yaml`, `k8s/db-pvc.yaml`, `k8s/db-service.yaml`
5. `k8s/app-configmap.yaml` et `k8s/kustomize/kustomization.yaml`
6. `terraform/modules/redis/main.tf` et `terraform/modules/postgres/main.tf`
7. `terraform-gcp/gke.tf` puis `terraform-gcp/gke-config.tf`
8. `terraform-gcp/k8s-manifests.tf`
9. `terraform-gcp/redis-vm.tf`
10. `vote/app.py`, `worker/Program.cs`, `result/server.js`

---

## Script oral detaille + blocs exacts a montrer

## 1) Docker data layer (45s)

Ce que je dis:
"En local, Redis et PostgreSQL sont deployes directement via images officielles, sans Dockerfile custom pour ces deux services. Ils restent sur le reseau interne back-net. PostgreSQL persiste les donnees via le volume db-data."

"Les variables PostgreSQL sont explicites dans compose: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB."

Bloc a montrer dans `docker-compose.yaml`:

```yaml
redis:
	image: redis:alpine
	networks:
		- back-net

db:
	image: postgres:15-alpine
	environment:
		POSTGRES_USER: postgres
		POSTGRES_PASSWORD: postgres
		POSTGRES_DB: postgres
	volumes:
		- db-data:/var/lib/postgresql/data
```

---

## 2) Healthchecks shell (25s)

Ce que je dis:
"Les checks sont metier: Redis doit renvoyer PONG, PostgreSQL doit valider SELECT 1."

Blocs a montrer:

```sh
# healthchecks/redis.sh
ping="$(redis-cli -h "$host" ping)"
[ "$ping" = 'PONG' ]
```

```sh
# healthchecks/postgres.sh
select="$(echo 'SELECT 1' | psql --host "$host" --username "$user" --dbname "$db" --quiet --no-align --tuples-only)"
[ "$select" = '1' ]
```

---

## 3) Kubernetes Redis + DB (1 min)

Ce que je dis:
"Sur Kubernetes, Redis et DB ne sont jamais exposes publiquement: les deux services sont en ClusterIP."

"Pour les probes Redis/DB, on injecte les scripts via initContainer busybox dans un volume emptyDir. Donc pas besoin d'images custom Redis/Postgres."

"Pour PostgreSQL, la persistance est assuree par un PVC 1Gi ReadWriteOnce, monte avec subPath: data."

Blocs a montrer:

```yaml
# k8s/redis-service.yaml
spec:
	type: ClusterIP
	ports:
		- port: 6379
```

```yaml
# k8s/db-pvc.yaml
spec:
	accessModes:
		- ReadWriteOnce
	resources:
		requests:
			storage: 1Gi
```

```yaml
# k8s/db-deployment.yaml
volumeMounts:
	- name: db-data
		mountPath: /var/lib/postgresql/data
		subPath: data
```

---

## 4) Config centralisee (20s)

Ce que je dis:
"La config applicative est centralisee via ConfigMap et consommee avec envFrom. Kustomize permet une generation alternative de cette config."

Blocs a montrer:

```yaml
# k8s/app-configmap.yaml
data:
	POSTGRES_HOST: db
	POSTGRES_PORT: "5432"
	REDIS_HOST: redis
	REDIS_PORT: "6379"
```

```yaml
# k8s/kustomize/kustomization.yaml
configMapGenerator:
	- name: app-config
		behavior: replace
```

---

## 5) Terraform local (important: etat actuel du code) (35s)

Ce que je dis:
"En Terraform local, Redis/Postgres sont geres dans des modules dedies. Ici, le code actuel n'utilise plus de bind mount healthchecks pour ces modules: les checks sont nativement `redis-cli ping` et `pg_isready`."

Blocs a montrer:

```hcl
# terraform/modules/redis/main.tf
healthcheck {
	test = ["CMD", "redis-cli", "ping"]
}
```

```hcl
# terraform/modules/postgres/main.tf
healthcheck {
	test = ["CMD-SHELL", "pg_isready -U ${var.postgres_user} -d ${var.postgres_db}"]
}
```

---

## 6) Terraform GCP (Part 2) (1 min)

Ce que je dis:
"Cote GCP, Terraform cree le cluster GKE et le node pool separe, en zone europe-west3-a, avec deletion_protection a false pour autoriser un destroy propre."

"Le provider Kubernetes est configure depuis le cluster GKE via token OAuth (`google_client_config`) dans gke-config.tf."

"Ensuite, on applique tous les YAML via kubernetes_manifest dans k8s-manifests.tf."

Blocs a montrer:

```hcl
# terraform-gcp/gke.tf
resource "google_container_cluster" "primary" {
	deletion_protection = false
}

resource "google_container_node_pool" "primary" {
	node_count = var.gke_node_count
}
```

```hcl
# terraform-gcp/gke-config.tf
data "google_client_config" "default" {}

provider "kubernetes" {
	token = data.google_client_config.default.access_token
}
```

---

## 7) Part 3 Redis VM (1 min)

Ce que je dis:
"La Part 3 decouple Redis du cluster. Terraform cree une VM Debian, installe redis-server, configure le requirepass, puis Kubernetes bascule le service Redis en headless avec un Endpoints manuel vers l'IP interne de la VM."

"Concretement: quand enable_redis_vm=true, on retire le deployment redis du cluster et on garde le nom logique redis pour l'application."

Blocs a montrer:

```hcl
# terraform-gcp/redis-vm.tf
resource "google_compute_instance" "redis_vm" {
	metadata_startup_script = <<-EOT
		apt install -y redis-server
		sed -e '/# requirepass/s/.*/requirepass ${var.redis_password}/' -i /etc/redis/redis.conf
	EOT
}
```

```hcl
# terraform-gcp/k8s-manifests.tf
resource "kubernetes_manifest" "redis_endpoints" {
	count = var.enable_redis_vm ? 1 : 0
}
```

---

## 8) Etat reel du code applicatif (point important a assumer) (35s)

Ce que je dis:
"Pour Redis, vote et worker lisent bien REDIS_HOST/REDIS_PASSWORD via variables d'environnement."

"Pour PostgreSQL, dans l'etat actuel du code, result et worker utilisent encore une connection string hardcodee en postgres/postgres. Donc pour une demo stable, on aligne le mot de passe PostgreSQL sur postgres."

Blocs a montrer:

```python
# vote/app.py
redis_host = os.getenv('REDIS_HOST', 'redis')
redis_password = os.getenv('REDIS_PASSWORD')
```

```csharp
// worker/Program.cs
var pgsql = OpenDbConnection("Server=db;Username=postgres;Password=postgres;");
```

```javascript
// result/server.js
connectionString: "postgres://postgres:postgres@db/postgres";
```

---

## 9) Commandes de demo (a executer + phrase orale)

1. ```bash
   gcloud container clusters get-credentials projet-nuage-tf --zone europe-west3-a --project lab-gcp-kube
   ```

````
Phrase: "Je connecte kubectl au cluster cree par Terraform."

2. ```bash
kubectl get nodes
kubectl get pods,svc,pvc,hpa,jobs -o wide
````

Phrase: "Je valide l'etat runtime global."

3. ```bash
   terraform -chdir=terraform-gcp output redis_vm_internal_ip
   kubectl get svc redis
   kubectl get endpoints redis -o wide
   ```

```
Phrase: "Je prouve la Part 3 avec l'IP VM + service headless + endpoint manuel."

---

## Reponses flash (Q/R)
Q: Pourquoi Redis/DB en ClusterIP ?
R: "Pour limiter l'exposition: trafic interne seulement."

Q: Pourquoi initContainer pour les healthchecks ?
R: "Pour injecter les scripts sans maintenir des images custom."

Q: Pourquoi apply cible avant apply complet ?
R: "Pour creer cluster/nodepool et kubeconfig avant les kubernetes_manifest."

Q: Si un pod est Pending ?
R: "Sur cluster 1 node, c'est souvent une contrainte de capacite CPU/memoire."

---

## Closing (10s)
"Notre stack data et infra est declarative, reproductible et verifiable de bout en bout avec Terraform et Kubernetes."
```
