# Homelab Infrastructure as Code (IaC)

## Objectif du Projet

Ce projet centralise les fichiers Docker Compose, les scripts et les variables nécessaires à la création et à la gestion des machines et services de notre homelab. L'objectif est d'automatiser le déploiement et la configuration de l'infrastructure pour une gestion simplifiée et reproductible.

## Structure des Répertoires

L'organisation des fichiers suit une structure hiérarchique claire :

```
.
├── srvDev/                   # Répertoire représentant une machine (ex: serveur de développement)
│   ├── portainer/            # Sous-répertoire pour un service spécifique (ex: Portainer)
│   │   ├── docker-compose.yml # Fichier Docker Compose pour ce service
│   │   └── .env               # Fichier d'environnement pour les variables spécifiques à ce service (optionnel)
│   ├── traefik/
│   │   ├── docker-compose.yml
│   │   └── .env
│   └── ...                   # Autres services sur la machine srvDev
├── srvProd/                  # Répertoire pour une autre machine (ex: serveur de production)
│   ├── nextcloud/
│   │   ├── docker-compose.yml
│   │   └── .env
│   └── ...                   # Autres services sur la machine srvProd
└── README.md                 # Ce fichier
```

- Chaque répertoire à la racine du projet (ex: `srvDev`, `srvProd`) correspond à une machine physique ou virtuelle du homelab.
- À l'intérieur de chaque répertoire machine, un sous-répertoire est créé pour chaque service hébergé sur cette machine (ex: `portainer`, `traefik`).
- Chaque répertoire de service contient au minimum un fichier `docker-compose.yml` décrivant comment démarrer le service. Il peut également contenir un fichier `.env` pour les variables d'environnement spécifiques à ce service.

## Prérequis

Avant de commencer, assurez-vous d'avoir les outils suivants installés sur les machines hôtes :

- **Docker**: Pour la conteneurisation des services.
- **Docker Compose**: Pour orchestrer les conteneurs Docker.
- **Ansible** (Recommandé): Pour l'automatisation de la configuration des machines hôtes et le déploiement des services.
- **Git**: Pour la gestion de version de ce dépôt.

## Déploiement et Gestion des Services

### Déploiement Manuel (par service)

1.  **Naviguez vers le répertoire du service** que vous souhaitez déployer :
    ```bash
    cd chemin/vers/la/machine/nom_du_service
    # Exemple: cd srvDev/portainer
    ```

2.  **Créez ou modifiez le fichier `.env`** si des variables d'environnement spécifiques sont nécessaires pour ce service. Référez-vous à la section "Gestion de la Configuration" pour plus de détails.

3.  **Démarrez le service** avec Docker Compose :
    ```bash
    docker-compose up -d
    ```

### Gestion des Services

-   **Arrêter un service**:
    ```bash
    docker-compose down
    ```
-   **Voir les logs d'un service**:
    ```bash
    docker-compose logs -f
    ```
-   **Mettre à jour un service** (après avoir modifié `docker-compose.yml` ou les images Docker) :
    ```bash
    docker-compose pull  # Pour télécharger les dernières versions des images
    docker-compose up -d --force-recreate # Pour recréer les conteneurs avec les nouvelles images/configurations
    ```

### Déploiement Automatisé (avec Ansible - Exemple)

Des playbooks Ansible peuvent être développés pour automatiser le déploiement sur plusieurs machines et services. Un exemple de tâche pourrait être :

```yaml
# tasks/deploy_service.yml
- name: Create service directory on host
  file:
    path: "/opt/homelab/{{ machine_name }}/{{ service_name }}"
    state: directory
    owner: your_user
    group: your_group
    mode: '0755'

- name: Copy docker-compose.yml for {{ service_name }}
  template:
    src: "{{ inventory_hostname }}/{{ service_name }}/docker-compose.yml.j2" # Utiliser un template si des variables sont injectées
    dest: "/opt/homelab/{{ machine_name }}/{{ service_name }}/docker-compose.yml"

- name: Copy .env file for {{ service_name }} if it exists
  copy:
    src: "{{ inventory_hostname }}/{{ service_name }}/.env"
    dest: "/opt/homelab/{{ machine_name }}/{{ service_name }}/.env"
  ignore_errors: yes # Si le .env n'est pas toujours présent

- name: Deploy {{ service_name }} with Docker Compose
  community.docker.docker_compose:
    project_src: "/opt/homelab/{{ machine_name }}/{{ service_name }}"
    state: present
```

## Automatisation avec Ansible

Pour une gestion plus avancée et automatisée, notamment pour la configuration initiale des machines virtuelles (VMs) ou des déploiements complexes, ce projet utilise Ansible.

Le répertoire `ansible/` à la racine du projet contient tous les éléments nécessaires à cette automatisation :

```
ansible/
├── inventory.yml       # Fichier d'inventaire listant les machines à gérer
├── playbooks/
│   ├── xxx.yml         # Playbook exemple pour configurer une nouvelle VM
│   └── yyy.yml         # Playbook principal orchestrant diverses tâches
└── roles/              # (Optionnel) Répertoire pour les rôles Ansible réutilisables
    └── ...
```

-   **`ansible/inventory.yml`**: Contient le fichier d'inventaire. Il définit les groupes de machines et les variables spécifiques aux hôtes.
-   **`ansible/playbooks/`**: Contient les playbooks Ansible. Un playbook comme `setup_vm.yml` pourrait contenir les tâches pour installer Docker, configurer les utilisateurs, etc., sur une nouvelle machine. Le `main.yml` peut être utilisé pour orchestrer des déploiements plus larges.

Pour exécuter un playbook, utilisez une commande similaire à celle-ci depuis la racine du projet :

```bash
ansible-playbook -i inventory.yml playbooks/install_VM.yml --vault-password-file ~/.vault_pass.txt
```

Adaptez le nom du fichier d'inventaire et du playbook en fonction de vos besoins.
## Gestion de la Configuration (Variables)

La configuration des services est gérée principalement via des fichiers `.env` situés dans le répertoire de chaque service.

-   **Variables spécifiques au service**: Définies dans le fichier `.env` du service (ex: `srvDev/portainer/.env`). Ces variables sont chargées automatiquement par Docker Compose lors du démarrage du service.
    ```env
    # Exemple de contenu pour srvDev/portainer/.env
    PUID=1000
    PGID=1000
    TZ=Europe/Paris
    ```

-   **Variables globales ou par machine**: Pour des variables partagées entre plusieurs services d'une même machine ou à travers tout le homelab, plusieurs approches sont possibles :
    *   Utiliser des variables d'environnement globales sur la machine hôte (moins recommandé pour la portabilité).
    *   Injecter des variables via Ansible lors du déploiement (voir exemple ci-dessus avec les templates `.j2`).
    *   Utiliser un fichier `.env` à la racine du répertoire machine qui serait sourcé par des scripts de déploiement.

Il est crucial de **ne pas commiter de secrets** (mots de passe, clés API, etc.) directement dans les fichiers `.env` versionnés. Utilisez des placeholders et remplissez-les manuellement sur la machine hôte, ou utilisez un système de gestion de secrets comme HashiCorp Vault ou Ansible Vault.

**Exemple de `docker-compose.yml` utilisant des variables d'un fichier `.env`:**

```yaml
# srvDev/portainer/docker-compose.yml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "9000:9000"
      - "8000:8000" # Optionnel, pour l'agent Edge
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    environment:
      - PUID=${PUID} # Variable PUID lue depuis le .env
      - PGID=${PGID} # Variable PGID lue depuis le .env
      - TZ=${TZ}     # Variable TZ lue depuis le .env

volumes:
  portainer_data:
```

Ce `README.md` sert de guide pour comprendre, déployer et gérer les services de votre homelab de manière structurée et reproductible.