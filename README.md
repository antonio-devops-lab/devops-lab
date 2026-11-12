# devops-lab

> **Proyecto archivado.** Fue la primera versión de mi homelab (septiembre de 2026):
> una Raspberry Pi 4 con k3s, aprovisionada con Ansible y desplegada con Argo CD.
> La plataforma actual vive en **[homelab-k3s](https://github.com/EA-etn/homelab-k3s)**,
> y la app con su pipeline en **[devsecops-pipeline](https://github.com/EA-etn/devsecops-pipeline)**.
> Nada de este repo se ejecuta ya en el clúster: los despliegues de prueba y OpenBao
> se retiraron el 2026-11-12.

## Lo que sigue siendo útil aquí

- **[Migración del arranque de microSD a SSD en la Raspberry Pi 4](docs/sd-to-ssd-migration.md)**:
  runbook verificado. Incluye el cuelgue del controlador USB (xHCI) con el puente
  JMicron JMS578 y UAS, cómo evitarlo con `usb-storage.quirks`, la corrección de
  `cmdline.txt`/`fstab` que `rpi-clone` no hace, el `BOOT_ORDER` con la microSD como
  respaldo y un benchmark de antes y después (~6× en secuencial, ~3,4× en escritura
  pequeña síncrona).
- **[Backups y prueba de restore](docs/backups-status.md)**: backup del SSD con `tar`,
  restore validado en una imagen loop sin tocar el sistema en vivo, y las trampas
  encontradas (`--exclude` con rutas absolutas, directorios excluidos que hay que recrear).

## Histórico (sustituido por homelab-k3s y devsecops-pipeline)

- [`ansible/`](ansible/): hardening de SSH, ufw (preparado, no aplicado) y k3s de un solo nodo.
- [`app/`](app/) y [`.github/workflows/`](.github/workflows/): app mínima en Python y CI
  que construye la imagen arm64, la sube a ghcr.io y actualiza el manifiesto.
- [`k8s/`](k8s/) y [`argocd/`](argocd/): manifiestos de prueba (nginx, PostgreSQL, inyección
  de secretos con OpenBao en modo dev) y la Application de Argo CD.
- [`docs/project-walkthrough.md`](docs/project-walkthrough.md): la guía paso a paso de todo
  lo anterior, de Ansible a GitOps.
