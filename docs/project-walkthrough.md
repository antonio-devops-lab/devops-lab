# Guía completa: de Ansible a GitOps con Argo CD

Runbook paso a paso, con los comandos exactos usados, para replicar todo lo
construido en este proyecto: hardening con Ansible, cluster k3s, despliegues
de prueba (nginx, PostgreSQL, OpenBao), GitHub + SSH, y un pipeline CI/CD con
GitHub Actions + Argo CD (GitOps) que cierra el ciclo solo.

Contexto de hardware: Raspberry Pi 4B, Ubuntu Server 24.04 LTS (`aarch64`),
hostname `devops-lab`, acceso por SSH con alias `raspberrypi` en
`~/.ssh/config` de la laptop. La migración de microSD a SSD y el fix del
puente USB-SATA están documentados aparte en
[sd-to-ssd-migration.md](sd-to-ssd-migration.md) — no se repite aquí.

---

## 1. Ansible: preparar el control node (la laptop)

Ansible corre desde la laptop, nunca se instala en la Pi — la Pi es solo
destino de despliegue, para evitar drift por edición manual.

```bash
pipx install ansible
pipx inject ansible ansible-lint --include-apps --force   # linter, expuesto al PATH
```

**Bug de este entorno:** Ansible falla con `ERROR: Ansible requires blocking
IO on stdin/stdout/stderr` en esta terminal. Antes de cualquier comando
`ansible`/`ansible-playbook`, hay que limpiar el flag non-blocking de los
file descriptors, en la misma sesión de shell:

```bash
python3 -c "
import os, fcntl
for fd in (0,1,2):
    fl = fcntl.fcntl(fd, fcntl.F_GETFL)
    fcntl.fcntl(fd, fcntl.F_SETFL, fl & ~os.O_NONBLOCK)
" && ansible-playbook site.yml
```

### Estructura del proyecto (`ansible/`)

```
ansible/
├── ansible.cfg
├── site.yml
├── inventory/hosts.ini
└── roles/
    ├── hardening/{tasks/{main,ssh,firewall}.yml, handlers/main.yml}
    └── k3s/tasks/main.yml
```

**`ansible.cfg`**:
```ini
[defaults]
inventory = inventory/hosts.ini
host_key_checking = True
retry_files_enabled = False
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
```

**`inventory/hosts.ini`**:
```ini
[pi]
devops-lab ansible_host=10.9.8.100 ansible_user=antonio ansible_ssh_private_key_file=~/.ssh/id_rsa_raspberrypi
```

**`site.yml`**:
```yaml
---
- name: Base hardening for devops-lab
  hosts: pi
  become: true
  roles:
    - hardening

- name: K3s cluster on devops-lab
  hosts: pi
  become: true
  roles:
    - k3s
```

---

## 2. Rol `hardening`: SSH (aplicado) y firewall (preparado, sin aplicar)

`roles/hardening/tasks/main.yml` importa dos archivos separados, cada uno
con su propio tag, para poder aplicar uno sin tocar el otro:

```yaml
---
- name: SSH hardening
  ansible.builtin.import_tasks: ssh.yml
  tags: [ssh]

- name: Firewall (ufw)
  ansible.builtin.import_tasks: firewall.yml
  tags: [firewall]
```

**`tasks/ssh.yml`** — deshabilita password auth y login root, verifica que
la conexión por key siga viva antes de terminar:
```yaml
---
- name: Disable SSH password authentication (key-only access)
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config.d/50-cloud-init.conf
    regexp: '^PasswordAuthentication'
    line: 'PasswordAuthentication no'
  notify: Reload sshd

- name: Disable SSH root login
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config.d/50-cloud-init.conf
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: Reload sshd

- name: Flush handlers so sshd reload happens now
  ansible.builtin.meta: flush_handlers

- name: Verify key-based SSH still works after reload
  ansible.builtin.wait_for_connection:
    timeout: 15
```

**`handlers/main.yml`**:
```yaml
---
- name: Reload sshd
  ansible.builtin.service:
    name: ssh
    state: reloaded
```

**`tasks/firewall.yml`** (escrito, pero deliberadamente sin aplicar aún —
ver nota al final):
```yaml
---
- name: Allow SSH through ufw (must happen before enabling ufw)
  community.general.ufw:
    rule: allow
    name: OpenSSH

- name: Set ufw default deny incoming
  community.general.ufw:
    direction: incoming
    policy: deny

- name: Set ufw default allow outgoing
  community.general.ufw:
    direction: outgoing
    policy: allow

- name: Enable ufw
  community.general.ufw:
    state: enabled
```

**Aplicar solo SSH, sin tocar el firewall:**
```bash
cd ansible
ansible-playbook site.yml --tags ssh
```

**Cuando decidas aplicar el firewall** (recuerda: vas a necesitar además
`6443/tcp` para `kubectl` remoto y el rango de NodePorts que uses):
```bash
ansible-playbook site.yml --tags firewall
```

---

## 3. Rol `k3s`: el cluster

`roles/k3s/tasks/main.yml` — instala k3s si no existe, espera a que el nodo
esté `Ready`, y **fusiona** (no sobrescribe) el kubeconfig en la laptop:

```yaml
---
- name: Check if k3s is already installed
  ansible.builtin.stat:
    path: /usr/local/bin/k3s
  register: k3s_binary

- name: Download k3s install script
  ansible.builtin.get_url:
    url: https://get.k3s.io
    dest: /tmp/k3s-install.sh
    mode: '0755'
  when: not k3s_binary.stat.exists

- name: Install k3s server (single-node, stable channel)
  ansible.builtin.command: /tmp/k3s-install.sh
  environment:
    INSTALL_K3S_CHANNEL: stable
    INSTALL_K3S_EXEC: "server --write-kubeconfig-mode 644"
  when: not k3s_binary.stat.exists
  changed_when: true

- name: Ensure k3s service is enabled and running
  ansible.builtin.systemd:
    name: k3s
    enabled: true
    state: started

- name: Wait for node to report Ready
  ansible.builtin.command: k3s kubectl get nodes
  register: k3s_node_status
  until: "'Ready' in k3s_node_status.stdout"
  retries: 12
  delay: 5
  changed_when: false

- name: Read kubeconfig from the node
  ansible.builtin.slurp:
    src: /etc/rancher/k3s/k3s.yaml
  register: k3s_kubeconfig_raw

- name: Ensure local ~/.kube directory exists
  become: false
  delegate_to: localhost
  ansible.builtin.file:
    path: "{{ lookup('env', 'HOME') }}/.kube"
    state: directory
    mode: '0700'

- name: Write this cluster's kubeconfig to its own file (never the shared one directly)
  become: false
  delegate_to: localhost
  ansible.builtin.copy:
    content: "{{ (k3s_kubeconfig_raw.content | b64decode) | replace('127.0.0.1', ansible_host) | replace('default', 'devops-lab') }}"
    dest: "{{ lookup('env', 'HOME') }}/.kube/devops-lab.yaml"
    mode: '0600'

- name: Ensure ~/.kube/config exists so it can be merged into
  become: false
  delegate_to: localhost
  ansible.builtin.file:
    path: "{{ lookup('env', 'HOME') }}/.kube/config"
    state: touch
    mode: '0600'
    modification_time: preserve
    access_time: preserve

- name: Merge devops-lab into ~/.kube/config, keeping any other clusters already there
  become: false
  delegate_to: localhost
  ansible.builtin.shell: >
    KUBECONFIG={{ lookup('env', 'HOME') }}/.kube/config:{{ lookup('env', 'HOME') }}/.kube/devops-lab.yaml
    kubectl config view --flatten > {{ lookup('env', 'HOME') }}/.kube/config.tmp
    && mv {{ lookup('env', 'HOME') }}/.kube/config.tmp {{ lookup('env', 'HOME') }}/.kube/config
    && chmod 0600 {{ lookup('env', 'HOME') }}/.kube/config
  changed_when: true

- name: Make devops-lab the active context
  become: false
  delegate_to: localhost
  ansible.builtin.command: kubectl config use-context devops-lab
  changed_when: true
```

**Correr todo menos el firewall:**
```bash
ansible-playbook site.yml --skip-tags firewall
```

**Verificar:**
```bash
kubectl get nodes -o wide
kubectl get pods -A
```

*Por qué el kubeconfig se fusiona en vez de sobrescribirse*: si algún día
agregas otro cluster (de cualquier proveedor) editando `~/.kube/config` a
mano, un re-run de este rol no debe borrártelo. El patrón general para sumar
cualquier cluster nuevo sin perder los que ya tienes:
```bash
KUBECONFIG=~/.kube/config:~/nuevo-cluster.yaml kubectl config view --flatten > ~/.kube/config.tmp
mv ~/.kube/config.tmp ~/.kube/config
```

---

## 4. Herramientas locales: k9s

Se instaló el `.deb` oficial publicado en las releases de GitHub del
proyecto (no un repo de terceros), verificando el checksum antes de instalar
y extrayendo el binario sin necesitar `sudo` (ver nota de `apt` más abajo):

```bash
curl -sLO https://github.com/derailed/k9s/releases/download/v0.51.0/k9s_linux_amd64.deb
curl -sLO https://github.com/derailed/k9s/releases/download/v0.51.0/checksums.sha256
grep "k9s_linux_amd64.deb" checksums.sha256 | sha256sum -c -

# sin sudo interactivo disponible en esta terminal, se extrae el binario directo:
mkdir -p /tmp/k9s-extract
dpkg-deb -x k9s_linux_amd64.deb /tmp/k9s-extract
cp /tmp/k9s-extract/usr/bin/k9s ~/.local/bin/k9s
chmod +x ~/.local/bin/k9s
```

k9s no necesita configuración propia — lee el mismo `~/.kube/config` que
`kubectl`, y ya queda apuntando al contexto `devops-lab` activo.

**Extensión de Kubernetes en VS Code**: si se instala/activa *antes* de que
exista `~/.kube/config`, se queda con el estado cacheado de "sin clusters".
Arreglo: `Ctrl+Shift+P` → **Developer: Reload Window**.

---

## 5. Despliegues de prueba en el cluster

### nginx — Deployment + Service (stateless, `k8s/nginx-test.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 1
  selector: {matchLabels: {app: nginx-test}}
  template:
    metadata: {labels: {app: nginx-test}}
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports: [{containerPort: 80}]
          resources:
            requests: {cpu: 50m, memory: 32Mi}
            limits: {cpu: 200m, memory: 128Mi}
---
apiVersion: v1
kind: Service
metadata: {name: nginx-test}
spec:
  type: NodePort
  selector: {app: nginx-test}
  ports: [{port: 80, targetPort: 80, nodePort: 30080}]
```
```bash
kubectl apply -f k8s/nginx-test.yaml
curl http://10.9.8.100:30080/
```

### PostgreSQL — StatefulSet + PVC + Secret (con estado real, `k8s/postgres-test.yaml`)

Por qué `StatefulSet` y no `Deployment`: le da al pod un nombre estable
(`postgres-test-0`) y, vía `volumeClaimTemplates`, un volumen persistente
**dedicado** a esa réplica — con `Deployment` las réplicas compartirían el
mismo volumen. `ClusterIP` (no `NodePort`) porque una base de datos no debe
quedar expuesta a toda la LAN.

```bash
kubectl apply -f k8s/postgres-test.yaml
kubectl rollout status statefulset/postgres-test

# prueba real de persistencia:
kubectl exec postgres-test-0 -- psql -U postgres -c \
  "CREATE TABLE prueba (id serial primary key, mensaje text); INSERT INTO prueba (mensaje) VALUES ('sobrevivi al reinicio del pod');"
kubectl delete pod postgres-test-0
kubectl wait --for=condition=Ready pod/postgres-test-0 --timeout=60s
kubectl exec postgres-test-0 -- psql -U postgres -c "SELECT * FROM prueba;"
# el dato sigue ahí: el PVC vive fuera del pod
```

### OpenBao + Vault Agent Injector — inyección de secretos (`k8s/openbao-injection-test.yaml`)

Se eligió OpenBao (fork open source de Vault, licencia MPL 2.0, gobernanza
Linux Foundation) en vez de HashiCorp Vault (licencia BSL desde 2023), API
100% compatible.

**Instalar Helm** (binario oficial + checksum, el tarball real vive en
`get.helm.sh`, GitHub solo aloja firmas):
```bash
curl -sLO https://get.helm.sh/helm-v4.2.4-linux-amd64.tar.gz
curl -sLO https://get.helm.sh/helm-v4.2.4-linux-amd64.tar.gz.sha256sum
sha256sum -c helm-v4.2.4-linux-amd64.tar.gz.sha256sum
tar -xzf helm-v4.2.4-linux-amd64.tar.gz
cp linux-amd64/helm ~/.local/bin/helm
```

**Instalar OpenBao (modo dev)**:
```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
kubectl create namespace openbao
helm install openbao openbao/openbao \
  --namespace openbao \
  --set server.dev.enabled=true \
  --set server.dev.devRootToken="root" \
  --set injector.enabled=true
```

**Configurar el secreto, la identidad y la política** (todo dentro del pod
del servidor, vía `bao` CLI):
```bash
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=root \
  bao kv put secret/myapp/config username=admin password="s3cr3t-desde-openbao"

kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=root bao auth enable kubernetes
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=root bao write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"

kubectl exec -n openbao openbao-0 -- sh -c 'cat <<EOF | env BAO_TOKEN=root bao policy write myapp-policy -
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF'

kubectl create serviceaccount myapp-sa
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=root bao write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=default \
  policies=myapp-policy \
  ttl=1h
```

**El deployment con las anotaciones que disparan la inyección** — nota el
prefijo `vault.hashicorp.com/` (OpenBao lo mantiene por compatibilidad, NO
usa `openbao.org/`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: myapp-secret-test}
spec:
  replicas: 1
  selector: {matchLabels: {app: myapp-secret-test}}
  template:
    metadata:
      labels: {app: myapp-secret-test}
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: app
          image: nginx:stable
          resources:
            requests: {cpu: 50m, memory: 32Mi}
            limits: {cpu: 200m, memory: 128Mi}
```
```bash
kubectl apply -f k8s/openbao-injection-test.yaml
# el pod queda 2/2 (app + sidecar vault-agent inyectado)
kubectl exec <pod> -c app -- cat /vault/secrets/config.txt
```

**Ver OpenBao en el navegador** (Service es `ClusterIP`, igual que Postgres
— nunca `NodePort` para algo con secretos):
```bash
kubectl port-forward -n openbao svc/openbao 8200:8200
# abrir http://localhost:8200/ui, token: root
```

---

## 6. Git + GitHub

### Identidad y repo local
```bash
git config --global user.name "Antonio V."
git config --global user.email "<ID>+antonio-devops-lab@users.noreply.github.com"  # correo privado de GitHub, Settings → Emails
git config --global init.defaultBranch main

git init
git add -A
git commit -m "mensaje..."
```

### Autenticación: `gh` CLI + SSH dedicado para git

```bash
gh auth login   # interactivo: GitHub.com → HTTPS → Login with a web browser

# llave SSH dedicada solo para GitHub (no reusar la de la Pi)
ssh-keygen -t ed25519 -C "antonio-devops-lab-git" -f ~/.ssh/id_ed25519_github -N ""
gh ssh-key add ~/.ssh/id_ed25519_github.pub --title "nexcore-laptop"
```

`~/.ssh/config` — agregar (sin tocar el bloque de `raspberrypi` existente):
```
Host github.com
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

**Verificar la huella del host antes de confiar en ella** (compararla contra
la que GitHub publica oficialmente en
`docs.github.com/.../githubs-ssh-key-fingerprints`, no aceptar a ciegas):
```bash
ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts
ssh-keygen -lf ~/.ssh/known_hosts | grep github
# debe coincidir: SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU
```

### Crear el repo remoto y apuntar a SSH
```bash
gh repo create devops-lab --private --source=. --remote=origin --push
git remote set-url origin git@github.com:antonio-devops-lab/devops-lab.git
git push
```

---

## 7. CI: GitHub Actions, cruzando de amd64 a arm64

La app de prueba (`app/app.py`) es un servidor HTTP mínimo, solo librería
estándar de Python — deliberadamente sin dependencias, para que el build
(emulado) sea rápido.

**`app/Dockerfile`**:
```dockerfile
FROM python:3.12-alpine

ARG APP_VERSION=dev
ENV APP_VERSION=$APP_VERSION

RUN adduser -D -u 10001 appuser
USER appuser

WORKDIR /app
COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```

**El problema a resolver**: los runners de GitHub Actions son `amd64`, la Pi
es `arm64` — no se puede simplemente `docker build` y esperar que corra ahí.
La solución estándar es **QEMU + Buildx**: QEMU emula la arquitectura arm64
dentro del runner amd64 durante el build (más lento que nativo, pero
funciona sin hardware arm64 real), Buildx es el motor de build de Docker que
sabe pedirle a QEMU esa emulación por plataforma.

**`.github/workflows/build-and-push.yml`**:
```yaml
name: Build and push app image

on:
  push:
    branches: [main]
    paths:
      - "app/**"
      - ".github/workflows/build-and-push.yml"
  workflow_dispatch: {}

env:
  IMAGE: ghcr.io/${{ github.repository_owner }}/devops-lab-app

permissions:
  contents: write
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU (para poder construir arm64 en un runner amd64)
        uses: docker/setup-qemu-action@v3

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: app
          platforms: linux/arm64
          push: true
          build-args: |
            APP_VERSION=${{ github.sha }}
          tags: |
            ${{ env.IMAGE }}:latest
            ${{ env.IMAGE }}:${{ github.sha }}

      - name: Update manifest with the new immutable image tag
        run: |
          sed -i "s|image: ${{ env.IMAGE }}:.*|image: ${{ env.IMAGE }}:${{ github.sha }}|" \
            k8s/apps/devops-lab-app/deployment.yaml

      - name: Commit and push manifest update
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git diff --quiet k8s/apps/devops-lab-app/deployment.yaml && exit 0
          git add k8s/apps/devops-lab-app/deployment.yaml
          git commit -F - <<EOF
          deploy: devops-lab-app @ ${{ github.sha }}

          Auto-generated by build-and-push.yml, not by a person.
          EOF
          git push
```

Puntos clave de este workflow, porque son el "por qué" detrás de cada línea:

- **`${{ secrets.GITHUB_TOKEN }}`**: GitHub lo genera e inyecta solo, por
  ejecución — nunca hay que crear ni guardar una credencial para el login al
  registry.
- **Tag por SHA, nunca solo `:latest`**: es el estándar de la industria
  (los propios docs de Kubernetes recomiendan evitar `:latest` en
  producción) — cada imagen queda trazable 1:1 a un commit exacto,
  permitiendo rollback confiable.
- **`permissions: contents: write`**: sin esto, el `git push` del último
  paso fallaría — por defecto el token de un workflow solo puede leer.
- El filtro `paths: app/**` evita que el propio commit del bot (que solo
  toca `k8s/apps/.../deployment.yaml`) dispare una segunda corrida —
  sin este cuidado, sería un bucle infinito.

**Probarlo localmente antes de confiar en el CI** (buena práctica, evita
gastar minutos de CI en errores triviales):
```bash
cd app
docker build -t devops-lab-app:test .
docker run -d --rm -p 18080:8080 -e APP_VERSION=local-test devops-lab-app:test
curl http://localhost:18080/
```

**Ver el workflow correr desde la terminal**:
```bash
gh run list --limit 5
gh run watch <run-id> --exit-status
```

---

## 8. Registry privado: `imagePullSecret`

Un paquete publicado a `ghcr.io` desde un repo privado nace privado. Se
comprobó así, sin gastar scopes de API:
```bash
docker pull ghcr.io/antonio-devops-lab/devops-lab-app:latest   # falla: unauthorized
```

Se optó por el patrón realista de empresa (en vez de hacer el paquete
público): un **Personal Access Token clásico, con el único scope
`read:packages`**, generado a mano en
`github.com/settings/tokens/new` (esto no se puede automatizar vía `gh` —
requiere la web UI por diseño de seguridad de GitHub).

Con el PAT ya generado, crear el secret **imperativamente**, nunca en un
archivo versionado:
```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=antonio-devops-lab \
  --docker-password=<PAT> \
  --docker-email=<tu-correo> \
  -n default
```

Referenciarlo en el Deployment:
```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-pull-secret
```

*Nota de seguridad*: si el PAT queda expuesto en texto plano en algún
momento (por ejemplo, pegado en un chat), rotarlo — revocar el viejo y
generar uno nuevo — es la práctica correcta, no hay que esperar a que
"pase algo".

---

## 9. CD: Argo CD (GitOps)

### Por qué Argo CD y no un `kubectl apply` manual o Tekton

- Es **pull-based**: un controlador *dentro* del cluster observa git y
  reconcilia solo — si alguien cambia algo a mano (`kubectl edit`), Argo CD
  lo detecta como drift y lo revierte. Eso es lo que específicamente se
  llama GitOps (formalizado por **OpenGitOps**, proyecto de la CNCF: 4
  principios — Declarative, Versioned and Immutable, Pulled Automatically,
  Continuously Reconciled).
- Tekton también puede dispararse por webhook de git, pero es **push-based**
  (algo externo empuja el cambio) — no reconcilia continuamente ni corrige
  drift por sí solo. Además duplicaría el trabajo que ya hace GitHub Actions
  compitiendo por los recursos limitados de la Pi.

### Instalar Argo CD
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
```

### Darle acceso de lectura al repo privado: Deploy Key

Una llave SSH dedicada, de **solo lectura**, atada únicamente a este repo
(nunca reusar una llave personal amplia para esto):
```bash
ssh-keygen -t ed25519 -C "argocd-devops-lab-readonly" -f /tmp/argocd_deploy_key -N ""
gh repo deploy-key add /tmp/argocd_deploy_key.pub \
  --title "argocd-in-cluster-readonly" \
  --repo antonio-devops-lab/devops-lab

# el secret de Argo CD se crea desde el ARCHIVO, nunca como texto literal:
kubectl create secret generic devops-lab-repo \
  -n argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:antonio-devops-lab/devops-lab.git \
  --from-file=sshPrivateKey=/tmp/argocd_deploy_key

kubectl label secret devops-lab-repo -n argocd argocd.argoproj.io/secret-type=repository

rm -f /tmp/argocd_deploy_key /tmp/argocd_deploy_key.pub   # no dejar copia local
```

### La `Application`: qué observar y cómo mantenerlo sincronizado

**`argocd/devops-lab-app-application.yaml`**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-lab-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:antonio-devops-lab/devops-lab.git
    targetRevision: main
    path: k8s/apps/devops-lab-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Este `kubectl apply` es el **único paso manual de todo el flujo** — de aquí
en adelante, Argo CD se sincroniza solo con cada push a `main`:
```bash
kubectl apply -f argocd/devops-lab-app-application.yaml
kubectl get application devops-lab-app -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}'
```

---

## 10. El ciclo completo, de punta a punta

```
1. push a app/ (código nuevo)
2. GitHub Actions:
     - QEMU + Buildx construyen la imagen arm64
     - se publica en ghcr.io con tag = SHA del commit
     - el propio workflow edita k8s/apps/devops-lab-app/deployment.yaml
       con ese SHA, y hace commit + push como "github-actions[bot]"
3. Argo CD nota el nuevo commit en git (poll automático, ~3 min por
   defecto, o al instante si configuras un webhook)
4. Argo CD aplica el manifiesto → nuevo ReplicaSet → pod nuevo corriendo
```

**Verificación real** (no solo "está Running", sino "es exactamente el
commit que yo empujé"):
```bash
curl http://10.9.8.100:30090/
# {"message": "...", "hostname": "...", "version": "<SHA exacto del commit>"}
```

Desde el `git push` hasta el pod nuevo corriendo: cero comandos manuales de
por medio.

---

## Notas finales

- **ufw sigue sin activarse** a propósito — quedó preparado en el rol
  `hardening` (tag `firewall`) para cuando se decida aplicarlo. Recordar
  permitir `6443/tcp` (API de Kubernetes) y el rango de NodePorts en uso
  antes de encenderlo.
- **El registry (`ghcr-pull-secret`) y el repo de Argo CD (`devops-lab-repo`)
  usan credenciales completamente separadas**, cada una con el mínimo
  permiso posible para su propósito — es el principio de menor privilegio
  aplicado en la práctica, no solo en la teoría.
- Ver [backups-status.md](backups-status.md) para el estado de los backups
  del sistema (independiente de todo lo de este documento).
