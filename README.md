# Ansible - Projet Starfleet

Projet Ansible de cybersécurité et d’automatisation réalisé dans le cadre d’un parcours de mise en pratique sur Linux.  
L’objectif est de déployer, durcir, surveiller et présenter une infrastructure de serveurs de manière reproductible avec Ansible.

## Objectifs du projet

Ce projet illustre l’usage d’Ansible pour :
- automatiser l’administration système ;
- appliquer des mesures de hardening ;
- déployer des services web sécurisés ;
- intégrer la supervision et la journalisation ;
- simuler des scénarios de détection et de réponse à incident ;
- structurer un projet Ansible de manière claire et réutilisable.

## Environnement

Le projet repose sur un environnement de test composé de :
- une machine de contrôle Ansible ;
- une ou plusieurs machines cibles Linux ;
- un inventaire `inventory.ini` ;
- des variables sensibles stockées avec Ansible Vault ;
- des playbooks séparés par thème ;
- des rôles Ansible pour la réutilisation ;
- des templates Jinja2 pour la configuration dynamique.

## Arborescence

```text
ansible-startfleet/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   └── all/
│       └── vault.yml
├── playbooks/
│   ├── 01-1-verifie_service.yml
│   ├── 01-2-copie_fichier.yml
│   ├── 01-3-Maj_paquets.yml
│   ├── 02-1-users.yml
│   ├── 02-2-ssh-hardening.yml
│   ├── 02-3-firewall.yml
│   ├── 02-4-disable-services.yml
│   ├── 03-1-filebeat.yml
│   ├── 03-2-audit-conformite.yml
│   ├── 03-3-hardening-kernel-pam.yml
│   ├── 04-1-suricata.yml
│   ├── 04-2-incident-response.yml
│   ├── 04-3-vault-demo.yml
│   ├── 04-4-facts.yml
│   └── 05-1-secure-webapp.yml
├── roles/
│   ├── geerlingguy.filebeat/
│   └── hardening_base/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       ├── templates/
│       └── vars/
├── templates/
│   ├── index.html.j2
│   ├── nginx-secure-site.conf.j2
│   └── filebeat.yml.j2
└── files/
```

## Pré-requis

- Ansible installé sur la machine de contrôle.
- Accès SSH vers les machines cibles.
- Python présent sur les hôtes distants.
- Droits `sudo` ou `become` sur les machines gérées.
- Un inventaire correctement configuré.
- Ansible Vault si vous utilisez des variables sensibles.

## Utilisation

### Vérifier la connectivité

```bash
ansible all -i inventory.ini -m ping
```

### Lancer un playbook

```bash
ansible-playbook -i inventory.ini playbooks/nom-du-playbook.yml
```

### Lancer avec Vault

```bash
ansible-playbook -i inventory.ini playbooks/05-1-secure-webapp.yml --ask-vault-pass
```

## Jobs du projet

### Job 1 - Prise en main

Ce premier job sert à valider l’installation d’Ansible, la configuration SSH, la création de l’inventaire et les premiers playbooks de base.

### Job 2 - Hardening des systèmes

Ce job met en place les premières mesures de sécurité :
- gestion des utilisateurs ;
- durcissement SSH ;
- configuration du pare-feu ;
- désactivation de services inutiles.

### Job 3 - Surveillance, logs et conformité

Ce job ajoute :
- Filebeat ;
- audit de conformité ;
- durcissement noyau et politique de mot de passe ;
- début de structuration en rôles.

### Job 4 - Détection et réponse à incident

Ce job couvre :
- Suricata ;
- scénario de réponse à incident ;
- utilisation d’Ansible Vault ;
- récupération des facts système.

### Job 5 - Déploiement sécurisé et bilan

Ce job final regroupe :
- le déploiement sécurisé d’une application web simple avec Nginx ;
- HTTPS avec certificat auto-signé ;
- collecte des logs Nginx via Filebeat ;
- optimisation de la structure du projet ;
- rapport final et présentation.

## Job 5 - Déploiement web sécurisé

Le playbook `05-1-secure-webapp.yml` :
- applique le rôle de hardening ;
- installe Nginx et OpenSSL ;
- déploie une page HTML statique ;
- configure Nginx en HTTPS ;
- ajoute la collecte des logs Nginx dans Filebeat ;
- redémarre les services concernés ;
- permet de tester l’accès via HTTPS.

### Validation

```bash
ansible webservers -i inventory.ini -b -a "systemctl status nginx"
ansible webservers -i inventory.ini -b -a "systemctl status filebeat"
ansible webservers -i inventory.ini -b -a "nginx -t"
curl -k https://IP_DU_SERVEUR_WEB
```

## Structure et bonnes pratiques

Le projet utilise :
- des **playbooks** pour l’orchestration ;
- des **rôles** pour la réutilisation ;
- des **variables** pour généraliser les paramètres ;
- des **templates Jinja2** pour générer les fichiers de configuration ;
- des **handlers** pour ne redémarrer les services qu’en cas de changement ;
- des **blocs** pour regrouper les tâches logiquement.

## Sécurité

Les actions de sécurité intégrées au projet comprennent :
- la création d’un compte administrateur dédié ;
- le durcissement SSH ;
- la mise en place d’un pare-feu ;
- la désactivation de services non essentiels ;
- le durcissement noyau et mot de passe ;
- la supervision des logs ;
- l’usage d’un HTTPS auto-signé pour le service web ;
- l’utilisation d’Ansible Vault pour les variables sensibles.

## Résultats obtenus

Le projet démontre qu’Ansible permet :
- de standardiser les déploiements ;
- de réduire les erreurs manuelles ;
- de gagner du temps sur les opérations répétitives ;
- d’améliorer la sécurité des serveurs ;
- de rendre un environnement plus clair et plus maintenable.

## Bilan

Ce projet a permis de regrouper les acquis des différents jobs dans un scénario complet de cybersécurité et d’automatisation.  
Il montre qu’Ansible est particulièrement efficace pour le hardening, la supervision, la gestion des services et la mise en production d’un serveur web sécurisé.

## Auteur

Projet réalisé par **Jericho-Cortez**.