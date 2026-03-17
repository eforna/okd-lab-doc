# IT12-OKD — Instal·lar OKD SNO Baremetal (192.168.2.4)

**Versió OKD:** 4.16 — `4.16.0-okd-scos.1` (Single Node OKD — SNO)  
**Sistema base:** CentOS Stream CoreOS — SCOS (instal·lat automàticament per l'Agent-Based Installer)  
**Data:** 10 de març de 2026  

---

## Resum del rol

| Paràmetre | Valor |
|-----------|-------|
| Hostname | `it12-okd` |
| IP estàtica | `192.168.2.4` |
| Gateway | `192.168.2.1` |
| DNS primari | `192.168.2.1` (GL-MT3000 · AdGuard Home) |
| DNS secundari | `8.8.8.8` |
| Clúster OKD | `okd.devops.lab` |
| API endpoint | `api.okd.devops.lab` |
| Apps wildcard | `*.apps.okd.devops.lab` |
| RAM | 32 GB |
| SSD | 1 TB |

---

## Sistema operatiu base: FCOS vs SCOS

> ⚠️ **Important** — OKD 4.12+ utilitza un SO base diferent a les versions anteriors.

| Versió OKD | SO base del node | Nom complet |
|-----------|-----------------|-------------|
| ≤ 4.11 | **FCOS** | Fedora CoreOS |
| ≥ 4.12 | **SCOS** | CentOS Stream CoreOS |

**Aquest document usa OKD 4.16 → SO base = SCOS (CentOS Stream CoreOS)**

### Per què importa?

- **El SO no s'instal·la manualment** — l'Agent-Based Installer inclou la imatge SCOS correcta per a la versió d'OKD.
- **No és Ubuntu, ni Fedora Server, ni Debian** — és un SO immutable i de lectura majoritàriament, gestionat per OKD via Machine Config Operator.
- **Les actualitzacions del SO es fan via OKD**, no amb `dnf update`. No s'ha d'actualitzar SCOS manualment mai.
- Per a documentació, cal buscar **"OKD 4.16 SCOS"** o **"OKD/SCOS"**, no "OKD/FCOS" (és documentació més antiga).

### Arquitectura del SO immutable

```
┌─────────────────────────────────────────┐
│  OKD (Cluster Operators)                │
│    └── Machine Config Operator (MCO)    │
│          └── gestiona SCOS              │
├─────────────────────────────────────────┤
│  SCOS — CentOS Stream CoreOS            │
│  (SO immutable, read-only majoritàriament)│
│  - /usr  → read-only                    │
│  - /etc  → read-write (via overlay)     │
│  - /var  → read-write                   │
└─────────────────────────────────────────┘
```

---



```
Developer (MacBook/Portàtil)
     │ git push
     ▼
  Gitea (IT12-DEVOPS 192.168.2.5)
     │ webhook
     ▼
  Jenkins (build + push Harbor)
     │ pull image
     ▼
┌─────────────────────────────┐
│   IT12-OKD (192.168.2.4)   │
│   OKD Single Node           │
│   ┌──────────┐ ┌──────────┐ │
│   │  POD     │ │  POD     │ │
│   │ Angular  │ │  Spring  │ │
│   └──────────┘ └──────────┘ │
└─────────────────────────────┘
```

---

## ✅ FASE 1 — Preparació prèvia: DNS al Synology

Abans d'instal·lar res al servidor, cal afegir els registres DNS necessaris per a OKD al Synology.

### 1.1 Afegir registres A per a OKD

**AdGuard Home → Filters → DNS Rewrites** (GL-MT3000 · `192.168.2.1`)

Crear els tres registres següents:

| Nom | Tipus | IP |
|-----|-------|----|
| `api.okd.devops.lab` | A | `192.168.2.4` |
| `api-int.okd.devops.lab` | A | `192.168.2.4` |
| `*.apps.okd.devops.lab` | A (wildcard) | `192.168.2.4` |

> ⚠️ El registre wildcard `*.apps.okd.devops.lab` és imprescindible per a les rutes de les aplicacions desplegades a OKD.

### 1.2 Verificar resolució DNS des del MacBook

```bash
nslookup api.okd.devops.lab 192.168.2.1
# Ha de retornar: 192.168.2.4

nslookup test.apps.okd.devops.lab 192.168.2.1
# Ha de retornar: 192.168.2.4
nslookup api.okd.devops.lab 192.168.2.1
# Ha de retornar: 192.168.2.4

nslookup test.apps.okd.devops.lab 192.168.2.1
# Ha de retornar: 192.168.2.4

✅ Si ambdues consultes retornen `192.168.2.4`, el DNS està correctament configurat.
```

---

## ✅ FASE 2 — Preparar el MacBook (eines i clau SSH)

> ℹ️ Tot es fa des del **MacBook**. No cal instal·lar cap SO al IT12 prèviament.  
> L'Agent-Based Installer genera una ISO que instal·la **CentOS Stream CoreOS (SCOS) + OKD** directament.

### 2.1 Instal·lar `openshift-install` al MacBook (macOS ARM)

```bash
# Definir la versió d'OKD
OKD_VERSION="4.16.0-okd-scos.1"
OKD_REPO="okd-project/okd-scos"

# Crear directori de treball
mkdir -p ~/okd-install
cd ~/okd-install

# Descarregar el binari per macOS ARM (Apple Silicon)
curl -L -o openshift-install-mac.tar.gz \
  "https://github.com/${OKD_REPO}/releases/download/${OKD_VERSION}/openshift-install-mac-arm64-${OKD_VERSION}.tar.gz"

# Descarregar el client oc per macOS ARM
curl -L -o openshift-client-mac.tar.gz \
  "https://github.com/${OKD_REPO}/releases/download/${OKD_VERSION}/openshift-client-mac-arm64-${OKD_VERSION}.tar.gz"

# Extreure
tar xzf openshift-install-mac.tar.gz
tar xzf openshift-client-mac.tar.gz

# Instal·lar (crear el directori si no existeix a macOS ARM)
sudo mkdir -p /usr/local/bin
sudo cp openshift-install /usr/local/bin/
sudo cp oc kubectl /usr/local/bin/

# Verificar
openshift-install version
oc version
```

### 2.2 Generar la clau SSH (si no existeix)

```bash
# Verificar si existeix
ls ~/.ssh/id_rsa.pub

# Si no existeix, crear-la:
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
```

---

## FASE 3 — Obtenir la MAC address del IT12

L'`agent-config.yaml` necessita la MAC de la targeta de xarxa del IT12.

**Opció A — Des del router GL-MT3000** (si el IT12 té algun SO arrencant):
- `http://192.168.2.1` → Clients → busca `192.168.2.4` o el hostname `it12-okd`

**Opció B — Si el IT12 té Ubuntu o qualsevol Linux**:
```bash
ssh edu@192.168.2.4
ip link show
# Cal cercar la interfície ethernet (enp3s0, eth0, eno1...) i copiar la MAC
# Exemple: link/ether aa:bb:cc:dd:ee:ff
```

**Opció C — Sticker físic** a la caixa o placa base del GEEKOM IT12.

> Cal anotar la MAC i el nom de la interfície (normalment `enp3s0` o `eno1`).  
> **Exemple:** `38:f7:cd:d6:c9:4b`

---

## FASE 4 — Crear els fitxers de configuració

### 4.1 Crear `install-config.yaml`

```bash
cd ~/okd-install

cat > install-config.yaml << 'EOF'
apiVersion: v1
baseDomain: devops.lab
metadata:
  name: okd
compute:
  - architecture: amd64
    hyperthreading: Enabled
    name: worker
    replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 192.168.2.0/24
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"fake":{"auth":"Zm9vOmJhcg=="}}}'
sshKey: |
EOF

# Afegir la clau SSH al fitxer
echo "  $(cat ~/.ssh/id_rsa.pub)" >> install-config.yaml
```

> **Nota:** El `pullSecret` amb valor `fake` és suficient per a OKD (open source). No cal compte Red Hat.

### 4.2 Crear `agent-config.yaml`

> ℹ️ S'utilitza DHCP en lloc de IP estàtica al yaml (nmstatectl no disponible a macOS).
> La IP fixa `192.168.2.4` es garanteix via **reserva DHCP al GL-MT3000** per la MAC `38:F7:CD:D6:C9:4B`.
> MAC real del IT12 ja anotada. Interfície: `enp3s0`.

```bash
cat > agent-config.yaml << 'EOF'
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: okd
rendezvousIP: 192.168.2.4
hosts:
  - hostname: it12-okd
    role: master
    interfaces:
      - name: enp3s0
        macAddress: "38:F7:CD:D6:C9:4B"
EOF
```

### 4.3 Fer còpia de seguretat dels fitxers

```bash
# IMPORTANT: openshift-install esborra install-config.yaml durant la generació
cp install-config.yaml install-config.yaml.bak
cp agent-config.yaml agent-config.yaml.bak
```

---

## FASE 5 — Generar la ISO de l'agent i carregar-la a Ventoy

### 5.1 Generar la ISO

```bash
cd ~/okd-install

openshift-install agent create image --dir=.

# Verificar — ha de pesar ~1 GB
ls -lh agent.x86_64.iso
```

> ⏳ Triga 1-2 minuts. Genera el fitxer `agent.x86_64.iso` al directori actual.

### 5.2 Copiar la ISO a l'USB Ventoy

Amb Ventoy **no cal `dd`** — simplement copia el fitxer ISO a la partició Ventoy:

**Des del MacBook** (si l'USB Ventoy és visible com a volum):
```bash
cp ~/okd-install/agent.x86_64.iso /Volumes/Ventoy/
```

**O des del portàtil Windows** (arrossega i deixa anar, o copia/enganxa):
- Primer transfereix la ISO del MacBook al Windows via xarxa o USB addicional
- Després copia `agent.x86_64.iso` a l'arrel de la partició Ventoy

### 5.3 Arrencar el IT12 des de l'USB

1. Cal connectar l'USB Ventoy al GEEKOM IT12.
2. Cal engegar el servidor i prémer **F7** (o **DEL**) per al menú d'arrancada.
3. Cal seleccionar l'USB com a dispositiu d'arrancada.
4. Ventoy mostrarà la ISO → cal seleccionar **`agent.x86_64.iso`**.
5. El sistema arrenca i comença la instal·lació automàtica de CentOS Stream CoreOS (SCOS) + OKD.

> ⚠️ A partir d'aquí **no cal fer res al servidor**. La instal·lació és completament automàtica (30-60 min).

---

## FASE 6 — Monitoritzar la instal·lació d'OKD

Durant l'arrancada des de la ISO de l'agent, **cal deixar el IT12 arrencat i connectat a la xarxa**. Es monitoritza el progrés des del MacBook:

```bash
cd ~/okd-install

# Monitoritzar l'arrancada del clúster (bootstrap)
openshift-install agent wait-for bootstrap-complete --dir=. --log-level=info

# Un cop fet bootstrap, esperar que el clúster estigui llest (30-60 min)
openshift-install agent wait-for install-complete --dir=. --log-level=info
```

✅ Quan finalitzi, apareixerà un missatge similar a:

```
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run
INFO     export KUBECONFIG=/root/okd-install/auth/kubeconfig
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.okd.devops.lab
INFO Login to the console with user: kubeadmin, and password: XXXXX-XXXXX-XXXXX-XXXXX
```

> ✅ **Cal anotar la contrasenya de `kubeadmin`** — és l'única forma d'accedir inicialment a la consola web!

---

## FASE 7 — Configuració post-instal·lació OKD

### 7.1 Configurar `oc` per accedir al clúster

Des del servidor (o MacBook):

```bash
# Exportar KUBECONFIG
export KUBECONFIG=~/okd-install/auth/kubeconfig

# Verificar accés al clúster
oc get nodes
# Ha de mostrar un node "it12-okd" amb status "Ready"

oc get clusteroperators
# Tots els operadors han d'estar Available=True
```

### 7.2 Instal·lar `oc` al MacBook

```bash
# Descarregar el client per macOS ARM (M3)
OKD_VERSION="4.16.0-okd-scos.1"
curl -L "https://github.com/okd-project/okd-scos/releases/download/${OKD_VERSION}/openshift-client-mac-arm64-${OKD_VERSION}.tar.gz" \
  | tar xz -C /usr/local/bin/

# Verificar
oc version

# Configurar accés (copiant el kubeconfig del servidor)
mkdir -p ~/.kube
scp edu@192.168.2.4:~/okd-install/auth/kubeconfig ~/.kube/config

# Verificar
oc get nodes
```

### 7.3 Crear usuari administrador (HTPasswd)

```bash
# Instal·lar htpasswd
sudo dnf install -y httpd-tools

# Crear fitxer d'usuaris
htpasswd -c -B ~/okd-htpasswd admin
# Introduir contrasenya segura

# Crear el Secret a OKD
oc create secret generic htpass-secret \
  --from-file=htpasswd=~/okd-htpasswd \
  -n openshift-config

# Crear l'OAuth config per activar HTPasswd
cat << 'EOF' | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: htpasswd_provider
      mappingMethod: claim
      type: HTPasswd
      htpasswd:
        fileData:
          name: htpass-secret
EOF

# Assignar rol de cluster-admin a l'usuari 'admin'
oc adm policy add-cluster-role-to-user cluster-admin admin
```

### 7.4 Accedir a la consola web

Cal obrir el navegador a:

```
https://console-openshift-console.apps.okd.devops.lab
```

- **Usuari inicial:** `kubeadmin`
- **Contrasenya:** la que es va mostrar al final de la instal·lació
- **Usuari admin permanent:** `admin` (configurat al pas anterior)

> ✅ Si no resol el domini, cal comprovar que el MacBook tingui el DNS configurat a `192.168.2.1`.  
> Vegeu el document [14_Configurar DNS manual.md](14_Configurar%20DNS%20manual.md)

### 7.5 Afegir Harbor com a registry de confiança

OKD necessita confiar en el registry Harbor (que utilitza un certificat autosignat):

```bash
# Obtenir el certificat de Harbor
openssl s_client -showcerts -connect harbor.devops.lab:443 </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > /tmp/harbor-ca.crt

# Crear ConfigMap amb el certificat
oc create configmap harbor-registry-trust \
  --from-file=harbor.devops.lab=/tmp/harbor-ca.crt \
  -n openshift-config

# Configurar OKD per confiar en Harbor
oc patch image.config.openshift.io/cluster \
  --type=merge \
  -p '{"spec":{"additionalTrustedCA":{"name":"harbor-registry-trust"}}}'
```

---

## FASE 8 — Desplegar una aplicació de prova (Angular + Spring)

### 8.1 Crear un projecte

```bash
oc new-project demo-app
```

### 8.2 Desplegar una imatge de backend (Spring)

```bash
# Substituir per la imatge real construïda per Jenkins i pujada a Harbor
oc new-app harbor.devops.lab/demo/spring-backend:latest \
  --name=spring-backend \
  -n demo-app

# Exposar el servei com a ruta
oc expose svc/spring-backend -n demo-app

# Verificar la ruta
oc get route spring-backend -n demo-app
# URL: http://spring-backend-demo-app.apps.okd.devops.lab
```

### 8.3 Desplegar una imatge de frontend (Angular)

```bash
oc new-app harbor.devops.lab/demo/angular-frontend:latest \
  --name=angular-frontend \
  -n demo-app

oc expose svc/angular-frontend -n demo-app

oc get route angular-frontend -n demo-app
# URL: http://angular-frontend-demo-app.apps.okd.devops.lab
```

### 8.4 Verificar els pods

```bash
oc get pods -n demo-app
oc get routes -n demo-app
oc logs deployment/spring-backend -n demo-app
```

---

## FASE 9 — Integració amb el pipeline CI/CD

### 9.1 Flux complet

```
git push → Gitea → Jenkins pipeline
              │
              │ 1. docker build
              │ 2. docker push harbor.devops.lab/<project>/<app>:<build>
              ▼
           Harbor Registry
              │
              │ OKD pull image
              ▼
        IT12-OKD (192.168.2.4)
        oc rollout latest deployment/<app>
```

### 9.2 Permetre a Jenkins fer `oc` rollouts

```bash
# Crear Service Account per a Jenkins
oc create serviceaccount jenkins -n demo-app

# Assignar permisos
oc policy add-role-to-user edit \
  system:serviceaccount:demo-app:jenkins \
  -n demo-app

# Obtenir el token per a Jenkins
oc create token jenkins -n demo-app --duration=8760h
# Copiar el token i configurar-lo a Jenkins com a credential de tipus "Secret text"
```

### 9.3 Jenkinsfile exemple

```groovy
pipeline {
  agent any
  environment {
    HARBOR_URL = "harbor.devops.lab"
    IMAGE_NAME = "${HARBOR_URL}/demo/spring-backend"
    OKD_API    = "https://api.okd.devops.lab:6443"
  }
  stages {
    stage('Build') {
      steps {
        sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
      }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-creds',
                          usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh "docker login ${HARBOR_URL} -u $USER -p $PASS"
          sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
        }
      }
    }
    stage('Deploy to OKD') {
      steps {
        withCredentials([string(credentialsId: 'okd-token', variable: 'TOKEN')]) {
          sh """
            oc login ${OKD_API} --token=\$TOKEN --insecure-skip-tls-verify=false
            oc set image deployment/spring-backend \
              spring-backend=${IMAGE_NAME}:${BUILD_NUMBER} \
              -n demo-app
            oc rollout status deployment/spring-backend -n demo-app
          """
        }
      }
    }
  }
}
```

---

## FASE 10 — Monitoratge

### 10.1 Prometheus i Grafana integrats

OKD SNO inclou Prometheus i Grafana de forma nativa:

```bash
# Verificar que l'stack de monitoratge funciona
oc get pods -n openshift-monitoring

# Accedir a Prometheus (via ruta)
oc get route prometheus-k8s -n openshift-monitoring

# Accedir a Grafana (via ruta)
oc get route grafana -n openshift-monitoring
```

URLs disponibles un cop configurat el DNS:
- Prometheus: `https://prometheus-k8s-openshift-monitoring.apps.okd.devops.lab`
- Grafana: `https://grafana-openshift-monitoring.apps.okd.devops.lab`
- AlertManager: `https://alertmanager-main-openshift-monitoring.apps.okd.devops.lab`

---

## Resolució de problemes

### El clúster no arrenca / timeout

```bash
# Consultar logs de l'instal·lador
openshift-install agent wait-for install-complete --dir=~/okd-install --log-level=debug

# Comprovar estat dels operators
oc get clusteroperators | grep -v "True.*False.*False"

# Comprovar pods problemàtics
oc get pods -A | grep -v Running | grep -v Completed
```

### DNS no resol correctament

```bash
# Des del servidor OKD
nslookup api.okd.devops.lab 192.168.2.1
nslookup console-openshift-console.apps.okd.devops.lab 192.168.2.1

# Verificar la configuració de DNS al node
cat /etc/resolv.conf
resolvectl status
```

### Error de certificat a la consola web

```bash
# Verificar l'Ingress operator
oc get clusteroperator ingress

# Regenerar certificats si cal
oc delete secret -n openshift-ingress router-certs-default
# OKD regenera el certificat automàticament
```

### Problema d'espai en disc

```bash
# Verificar ús de disc
df -hT

# Netejar imatges no usades
podman system prune -af

# Pujar l'OKD a un nou punt de muntatge si té poc espai /var/lib/containers
df -h /var/lib/containers
```

---

## Resum d'URLs del clúster OKD

| Servei | URL |
|--------|-----|
| Consola web OKD | `https://console-openshift-console.apps.okd.devops.lab` |
| API Kubernetes | `https://api.okd.devops.lab:6443` |
| Prometheus | `https://prometheus-k8s-openshift-monitoring.apps.okd.devops.lab` |
| Grafana | `https://grafana-openshift-monitoring.apps.okd.devops.lab` |
| AlertManager | `https://alertmanager-main-openshift-monitoring.apps.okd.devops.lab` |

---

## Referència ràpida de comandes `oc`

```bash
# Estat general
oc get nodes
oc get clusteroperators
oc get pods -A

# Gestió de projectes
oc new-project <nom>
oc get projects
oc project <nom>

# Desplegaments
oc get deployments -n <projecte>
oc rollout status deployment/<nom> -n <projecte>
oc scale deployment/<nom> --replicas=2 -n <projecte>

# Rutes
oc get routes -n <projecte>
oc expose svc/<servei> -n <projecte>

# Logs
oc logs deployment/<nom> -n <projecte> --tail=50 -f

# Events (per depurar problemes)
oc get events -n <projecte> --sort-by='.lastTimestamp'
```
