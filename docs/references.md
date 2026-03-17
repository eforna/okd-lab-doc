# Referències — OKD SNO Lab

## OKD 4.16 — Documentació oficial

| Recurs | URL |
|--------|-----|
| Documentació principal OKD 4.16 | https://docs.okd.io/4.16/welcome/index.html |
| Instal·lació SNO — Preparació | https://docs.okd.io/4.16/installing/installing_sno/install-sno-preparing-to-install-sno.html |
| Instal·lació SNO — Instal·lació | https://docs.okd.io/4.16/installing/installing_sno/install-sno-installing-sno.html |
| Agent-Based Installer (overview) | https://docs.okd.io/4.16/installing/installing_with_agent_based_installer/understanding-disconnected-installation-mirroring.html |
| Post-instal·lació | https://docs.okd.io/4.16/post_installation_configuration/index.html |
| Autenticació HTPasswd | https://docs.okd.io/4.16/authentication/identity_providers/configuring-htpasswd-identity-provider.html |
| Monitoratge (Prometheus/Grafana) | https://docs.okd.io/4.16/monitoring/monitoring-overview.html |
| Registry — afegir CA de confiança | https://docs.okd.io/4.16/registry/configuring-registry-operator.html |

---

## OKD SCOS — Releases i binaris

| Recurs | URL |
|--------|-----|
| Releases okd-project/okd-scos (GitHub) | https://github.com/okd-project/okd-scos/releases |
| Release 4.16.0-okd-scos.1 (versió usada) | https://github.com/okd-project/okd-scos/releases/tag/4.16.0-okd-scos.1 |
| Binari `openshift-install` macOS ARM | https://github.com/okd-project/okd-scos/releases/download/4.16.0-okd-scos.1/openshift-install-mac-arm64-4.16.0-okd-scos.1.tar.gz |
| Binari `oc` client macOS ARM | https://github.com/okd-project/okd-scos/releases/download/4.16.0-okd-scos.1/openshift-client-mac-arm64-4.16.0-okd-scos.1.tar.gz |

---

## Agent-Based Installer (ABI) — Guies comunitat

| Recurs | URL |
|--------|-----|
| Guia ABI OKD SCOS SNO (HackMD oficial) | https://hackmd.io/@sherine-k/SkVYj_FXh |
| OKD SNO Assisted Installer Guide | https://okd.io/docs/project/guides/sno-assisted-installer/ |
| OKD SNO disconnected install (blog, feb 2026) | https://blog.stderr.at/openshift-platform/infrastructure/2026-02-09-okd-sno-disconnected/ |
| Fake pullSecret per OKD (GitHub discussion) | https://github.com/okd-project/okd/discussions/1930 |

---

## Maquinari — GEEKOM IT12

| Recurs | URL |
|--------|-----|
| GEEKOM IT12 — BIOS / Boot menu tutorial (PDF) | https://support.geekompc.com/uploads/Technical-Support/SOP/English-Version/GEEKOM_IT12-Set-BIOS-Tutourial.pdf |
| GEEKOM IT12 — Suport oficial | https://www.geekom.com/support/ |

> **Tecles GEEKOM IT12:**
> - `F7` → Boot menu (selecció dispositiu d'arrencada)
> - `DEL` → Entrar al BIOS/UEFI setup

---

## Xarxa del lab

| Servei | IP | URL admin |
|--------|----|-----------|
| Router GL-MT3000 (AdGuard Home + DHCP + DNS) | `192.168.2.1` | http://192.168.2.1 |
| IT12-OKD (servidor OKD SNO) | `192.168.2.4` | — |
| IT12-DEVOPS (Gitea + Jenkins + Harbor) | `192.168.2.5` | — |

### DNS Rewrites a configurar (AdGuard Home)

| Nom | IP |
|-----|----|
| `api.okd.devops.lab` | `192.168.2.4` |
| `api-int.okd.devops.lab` | `192.168.2.4` |
| `*.apps.okd.devops.lab` | `192.168.2.4` |

---

## URLs del clúster OKD (post-instal·lació)

| Servei | URL |
|--------|-----|
| Consola web OKD | https://console-openshift-console.apps.okd.devops.lab |
| API Kubernetes | https://api.okd.devops.lab:6443 |
| Prometheus | https://prometheus-k8s-openshift-monitoring.apps.okd.devops.lab |
| Grafana | https://grafana-openshift-monitoring.apps.okd.devops.lab |
| AlertManager | https://alertmanager-main-openshift-monitoring.apps.okd.devops.lab |

---

## Eines macOS

| Eina | Instal·lació |
|------|-------------|
| `openshift-install` | Vegeu secció *Releases i binaris* |
| `oc` (OpenShift CLI) | Vegeu secció *Releases i binaris* |
| `htpasswd` | `brew install httpd` |
| `nslookup` / `dig` | Inclòs a macOS |
| Ventoy (USB multiboot) | https://www.ventoy.net/en/download.html |
