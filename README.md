# okd-lab-doc

Documentació del servidor OKD (IT12-OKD, 192.168.2.4).

Guies d'instal·lació, notes de configuració i resolució d'errors
del clúster OKD Single Node (SNO) del laboratori DevOps.

## Estructura

```
okd-lab-doc/
├── README.md
└── docs/
    ├── architecture.md    ← arquitectura OKD SNO
    ├── install.md         ← instal·lació pas a pas
    ├── network.md         ← xarxa i DNS
    └── pipelines.md       ← integració amb Jenkins i Harbor
```

## Integració amb el lab

```
Gitea (it12-devops) → Jenkins → Harbor → OKD (it12-okd)
```

## Fitxers de configuració

Consulta el repo [it12-okd](https://github.com/efornaguera/it12-okd)
per als manifests i scripts del servidor.
