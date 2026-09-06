# devops-lab

Proyecto de portafolio DevOps/SRE: un Raspberry Pi 4B como servidor real,
aprovisionado con Ansible y corriendo un cluster Kubernetes (k3s).

**En construcción.** El README completo (arquitectura, diagrama, decisiones)
se escribe al final, cuando el resto de las piezas del proyecto estén listas.

## Qué existe hasta ahora

- [`ansible/`](ansible/) — provisioning: hardening de SSH, cluster k3s.
- [`k8s/`](k8s/) — manifiestos de prueba desplegados en el cluster (nginx,
  PostgreSQL con almacenamiento persistente, OpenBao con inyección de secretos).
- [`docs/`](docs/) — runbooks: migración de microSD a SSD, estado de backups.

## Pendiente

- Firewall (ufw) vía Ansible.
- Pipeline CI/CD (GitHub Actions + Argo CD).
- Monitoreo (Prometheus + Grafana).
- README final con diagrama de arquitectura.
