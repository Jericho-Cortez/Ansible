---
title: Job 1 — Prise en main et premières automatisations Ansible
source_file: job1-ansible.md
type: notes
status: draft
created: 2026-07-15
updated: 2026-07-15
tags:
  - ansible
  - automation
  - sysadmin
  - obsidian
---
> [!info] Résumé rapide
> Document amélioré pour Obsidian avec frontmatter propre, structure normalisée et conservation intégrale du contenu.

# Job 1 — Prise en main et premières automatisations Ansible

## Résumé du besoin réel

Le Job 1 consiste à construire un socle Ansible propre et fonctionnel avant toute phase de hardening ou d'automatisation avancée. Le sujet demande d'installer Ansible sur un serveur de contrôle, de configurer l'accès SSH sans mot de passe vers les machines cibles, de créer un inventaire structuré, de valider la connectivité avec des commandes ad-hoc, puis d'écrire les premiers playbooks de base.

Dans la logique du parcours de référence, cette étape ne sert pas seulement à “faire marcher Ansible”, mais à valider toute la chaîne de fonctionnement : laboratoire, SSH, inventaire, exécution à distance et premier niveau d'idempotence.

## Objectifs obligatoires

Le sujet impose les éléments suivants pour le Job 1 :

- Installer Ansible sur le serveur de contrôle.
- Générer une paire de clés SSH sur le contrôleur et distribuer la clé publique sur toutes les machines cibles.
- Créer un fichier `inventory.ini` regroupant les machines cibles par catégories comme `webservers`, `dbservers` et `all_servers`.
- Vérifier la connectivité avec `ansible all -m ping` et avec des commandes simples à distance.
- Créer trois premiers playbooks : vérification de l'état des services, copie d'un fichier simple et mise à jour des paquets système selon la distribution Linux.

## Contraintes et critères d'évaluation

Le document de projet présente Ansible comme un outil d'automatisation sans agent destiné à renforcer la sécurité, gérer la configuration des systèmes et simuler des réponses à incident dans une infrastructure Linux variée. Même si le reste du projet va plus loin, le Job 1 est évalué implicitement sur la fiabilité du socle : si le contrôleur, le SSH ou l'inventaire sont mal préparés, les jobs suivants deviennent instables ou bloquants.

Le parcours de référence recommande un démarrage progressif : inventaire simple, test de la connexion, commandes ad-hoc, puis premier playbook. Cette méthode est particulièrement adaptée à un labo Debian sur VMware, car elle limite les variables d'erreur et facilite le débogage.

## Concepts à maîtriser

### Control node et managed nodes

Ansible fonctionne selon un modèle dans lequel un nœud de contrôle pilote des machines gérées à distance, sans agent logiciel permanent sur ces dernières. La communication repose principalement sur SSH, ce qui explique pourquoi la préparation de l'accès SSH est une étape structurante du Job 1.

### Inventaire Ansible

L'inventaire permet de décrire les machines cibles et de les regrouper logiquement. Le parcours recommande de commencer avec un inventaire au format INI, plus simple à lire et suffisant pour les premiers labs, avant de passer à des formes plus riches comme YAML ou les inventaires dynamiques.

### Commandes ad-hoc et playbooks

Les commandes ad-hoc servent à tester rapidement une action sur un ou plusieurs hôtes, par exemple la connectivité avec le module `ping` ou l'affichage d'informations système. Les playbooks servent ensuite à décrire des automatisations réutilisables, lisibles et idempotentes.

### Idempotence

Une automatisation Ansible doit idéalement être idempotente, c'est-à-dire produire un état stable lorsqu'elle est relancée plusieurs fois. Cette notion est centrale dès le Job 1, car elle distingue un simple script distant d'une vraie gestion de configuration.

## Pièges et erreurs fréquentes

Le premier piège est de vouloir écrire des playbooks avant d'avoir validé la chaîne réseau, SSH et inventaire. Le parcours recommande explicitement de corriger toute erreur de connexion ou d'authentification avant de passer à l'étape suivante.

Les erreurs classiques à éviter sont les suivantes :

- Oublier `python3` sur les machines cibles, ce qui peut empêcher le fonctionnement correct des modules Ansible.
- Ne pas tester la connexion SSH manuellement avant d'utiliser `ansible ... -m ping`.
- Utiliser `shell` ou `command` alors qu'un module dédié existe, ce qui réduit la qualité et l'idempotence des playbooks.
- Créer un inventaire sans groupes logiques ou sans le vérifier avec `ansible-inventory --graph`.
- Travailler directement avec `root` au lieu d'un utilisateur standard disposant de privilèges `sudo`, ce qui complique la suite du projet.

## Architecture recommandée

Pour un labo Debian sur VMware, une architecture simple à trois machines est suffisante pour le Job 1 : un contrôleur Ansible et deux machines cibles. Cette approche respecte l'esprit du parcours, même si les exemples du site utilisent parfois un lab plus large.

Exemple de répartition :

- `ctrl` : nœud de contrôle Ansible
- `web1` : cible du groupe `webservers`
- `db1` : cible du groupe `dbservers`

Une structure de projet minimale et propre peut être la suivante :

```text
ansible-starfleet/
├── inventory.ini
├── playbooks/
└── files/
```

Cette structure est suffisante pour le Job 1 tout en préparant correctement la suite du projet, notamment les rôles et l'organisation plus modulaire attendue dans les jobs suivants.
## Ce qui est indispensable, recommandé et optionnel

| Niveau        | Éléments                                                                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Indispensable | Installation d'Ansible, SSH par clé, `inventory.ini`, test `ping`, trois playbooks demandés par le sujet.[cite:1]                                             |
| Recommandé    | Vérification de l'inventaire avec `ansible-inventory --graph`, utilisation des modules dédiés, relecture de l'idempotence avec un second run.[cite:2][cite:3] |
| Optionnel     | Préparer un `ansible.cfg`, documenter les tests, structurer le dépôt dès maintenant pour faciliter le Job 2 et le Job 3.[cite:2][cite:3]                      |

## Comparaison des solutions possibles

| Sujet                       | Option 1                               | Option 2                                                     | Recommandation                                                       |
| --------------------------- | -------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------- |
| Format d'inventaire         | INI, simple pour débuter.[cite:2]      | YAML, plus riche mais plus verbeux.[cite:3]                  | INI pour le Job 1, conformément au parcours.[cite:2]                 |
| Validation initiale         | Aller directement au playbook.[cite:2] | Passer d'abord par des commandes ad-hoc.[cite:2]             | Commencer par l'ad-hoc pour isoler les problèmes rapidement.[cite:2] |
| Gestion des actions système | Utiliser `shell`/`command`.[cite:3]    | Utiliser un module dédié (`apt`, `copy`, `service`).[cite:3] | Préférer les modules dédiés pour garder l'idempotence.[cite:3]       |
| Vérification finale         | Un seul lancement.[cite:3]             | Deux lancements pour contrôler la stabilité.[cite:3]         | Deux lancements pour démontrer un comportement propre.[cite:3]       |

## Étapes opérationnelles
### Phase 1 — Préparer le laboratoire

**Objectif** : disposer de trois machines Debian joignables sur le réseau, avec SSH actif et Python disponible sur les cibles.

**Tâches prioritaires** :

- Vérifier la connectivité réseau entre les VMs.
- Installer et activer `openssh-server` sur les cibles.
- Vérifier la présence de `python3` sur les machines gérées.
- Choisir l'utilisateur Debian qui sera utilisé par Ansible.

**Difficulté** : facile.  
**Prérequis** : bases Linux et VMware.  
**Temps estimé** : 30 à 45 minutes.

Sur chaque machine cible Debian :

```bash
sudo apt update
sudo apt install -y openssh-server python3
sudo systemctl enable --now ssh
sudo systemctl status ssh
python3 --version
```

### Phase 2 — Installer Ansible sur le contrôleur

**Objectif** : rendre le serveur de contrôle capable d'exécuter des commandes Ansible et de piloter les hôtes distants.

**Tâches prioritaires** :

- Mettre à jour les paquets Debian.
- Installer Ansible.
- Vérifier la version avec `ansible --version`.

**Difficulté** : facile.  
**Prérequis** : accès Internet sur le contrôleur.  
**Temps estimé** : 10 minutes.

Sur le contrôleur :

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```
![[Ansible Installé.png]]
### Phase 3 — Configurer SSH par clé

**Objectif** : supprimer la dépendance au mot de passe pour que le contrôleur puisse exécuter les tâches automatiquement sur les machines cibles.

**Tâches prioritaires** :

- Générer une clé SSH sur le contrôleur.
- Copier la clé publique sur chaque cible.
- Tester la connexion SSH manuelle avant de passer à Ansible.

**Difficulté** : facile à moyen.  
**Prérequis** : service SSH fonctionnel sur les cibles.  
**Temps estimé** : 20 à 30 minutes.

Sur le contrôleur :

```bash
ssh-keygen -t ed25519 -C "ansible-controller"
```

Depuis le contrôleur :

```bash
ssh-copy-id ton_user@IP_WEB1
ssh-copy-id ton_user@IP_DB1
```

Puis tester :

```bash
ssh ton_user@IP_WEB1
ssh ton_user@IP_DB1
```

### Phase 4 — Créer l'inventaire

**Objectif** : décrire clairement les hôtes gérés et leurs groupes dans un fichier `inventory.ini`.

**Tâches prioritaires** :

- Définir les groupes `webservers` et `dbservers`.
- Ajouter les hôtes avec leur IP et l'utilisateur SSH.
- Vérifier la structure avec `ansible-inventory --graph`.

**Difficulté** : facile.  
**Prérequis** : IP connues et accès SSH valide.  
**Temps estimé** : 15 à 20 minutes.

Exemple minimal :

```ini
[webservers]
web1 ansible_host=IP_WEB1 ansible_user=ton_user ansible_python_interpreter=/usr/bin/python3

[dbservers]
db1 ansible_host=IP_DB1 ansible_user=ton_user ansible_python_interpreter=/usr/bin/python3

[all_servers:children]
webservers
dbservers
```
![[Création inventory.ini.png]]
#### Vérifier l'inventaire

```bash
ansible-inventory -i inventory.ini --graph
```
![[Verification inventory.ini avec un graph.png]]
### Phase 5 — Tester avec des commandes ad-hoc

**Objectif** : confirmer que la chaîne Ansible est opérationnelle avant d'écrire les playbooks.

**Tâches prioritaires** :

- Lancer `ansible all -m ping` ou l'équivalent sur le groupe choisi.
- Exécuter une commande de diagnostic simple à distance.
- Vérifier la collecte d'informations de base avec les facts si nécessaire.

**Difficulté** : facile.  
**Prérequis** : inventaire valide et SSH fonctionnel.  
**Temps estimé** : 15 à 20 minutes.

```bash
ansible all_servers -i inventory.ini -m ping
```
![[Tester la connectivité Ansible (ping).png]]
#### Exécuter une commande ad-hoc simple

```bash
ansible all_servers -i inventory.ini -a "hostname"
ansible all_servers -i inventory.ini -a "uptime"
```
![[Exécuter une commande ad-hoc simple.png]]
### Phase 6 — Écrire les premiers playbooks

**Objectif** : répondre au sujet avec trois automatisations de base réutilisables et lisibles.

**Tâches prioritaires** :

- Créer un playbook pour vérifier l'état d'un service sur toutes les cibles.
- Créer un playbook pour copier un fichier simple depuis le contrôleur.
- Créer un playbook de mise à jour des paquets avec prise en compte des familles de distributions Linux.

**Difficulté** : moyen.  
**Prérequis** : maîtrise du YAML et des modules de base.  
**Temps estimé** : 45 à 90 minutes.

#### Playbook 1 — Vérifier l'état d'un service

```yaml
---
- name: Vérifier l'état du service SSH
  hosts: all_servers
  tasks:
	# changed_when: false évite qu'Ansible marque la tâche comme modifiée
    # failed_when: false évite l'échec du playbook si le service n'est pas actif
    - name: Vérifier si le service ssh est actif
      ansible.builtin.command: systemctl is-active ssh
      register: ssh_status
      changed_when: false
      failed_when: false
      
    # stdout contient la sortie texte de la commande précédente
    - name: Afficher l'état du service ssh
      ansible.builtin.debug:
        msg: "Le service ssh est : {{ ssh_status.stdout }}"
```
#### Vérification Playbook 1

```bash
ansible.playbook -i inventory.ini fichier.yml --check
```
![[Vérification Playbook 1.png]]

```bash
ansible.playbook -i inventory.ini fichier.yml
```
![[Playbook 1 Ok.png]]
####  Playbook 2 — Copier un fichier simple

```yaml
---
- name: Copier un fichier de test
  hosts: all_servers
  
  tasks:
    - name: Copier le fichier hello.txt
      ansible.builtin.copy:
        src: ../files/hello.txt
        dest: /home/jericho/hello.txt
        owner: jericho
        group: jericho
        mode: '0644'
```

Créer le fichier source :

```bash
echo "Déployé par Ansible" > files/hello.txt
```
##### Vérification Playbook 2
```bash
ansible.playbook -i inventory.ini fichier.yml --check
```
![[Vérification Playbook 2.png]]
```bash
ansible.playbook -i inventory.ini fichier.yml
```
![[Playbook 2 Ok.png]]
#### Vérification Playbook 2 Cible
![[Vérification Playbook2 Cible.png]]

#### Playbook 3 — Mettre à jour les paquets

```yaml
---
- name: Mettre à jour les systèmes
  hosts: all_servers
  
  tasks:
    - name: Mettre à jour APT sur les systèmes Debian
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
      when: ansible_facts['os_family'] == "Debian"

    - name: Mettre à jour YUM sur les systèmes RedHat
      ansible.builtin.yum:
        name: "*"
        state: latest
      when: ansible_facts['os_family'] == "RedHat"
```

##### Vérification Playbook 3
```bash
ansible.playbook -i inventory.ini fichier.yml --check
```
![[Vérification Playbook 3.png]]
```bash
ansible.playbook -i inventory.ini fichier.yml
```
![[Playbook 3 Ok.png]]

## Validation attendue

Le Job 1 peut être considéré comme terminé lorsque les points suivants sont validés :

- Le contrôleur exécute correctement `ansible --version`.
- Les cibles sont accessibles en SSH sans mot de passe.
- L'inventaire est lisible et validé par `ansible-inventory --graph`.
- La commande `ansible all_servers -m ping` retourne `pong` pour chaque cible.
- Les trois playbooks du sujet s'exécutent sans erreur et restent cohérents lors d'un second lancement.

## Bilan de phase

### Ce qui est fait

Le Job 1 est cadré, documenté et structuré en phases, avec une méthode alignée sur le parcours de référence et sur les attentes du sujet.

### Ce qui reste à faire

Le laboratoire Debian sur VMware doit maintenant être configuré, puis les commandes et playbooks doivent être exécutés et testés.

### Points de vigilance

Il ne faut pas passer au Job 2 tant que la chaîne SSH, l'inventaire et le `ping` Ansible ne sont pas entièrement maîtrisés. C'est une étape bloquante, pas une formalité.

### Prochaine étape recommandée

Réaliser la phase 1 à la phase 4, puis valider les sorties de `ansible-inventory --graph` et `ansible all_servers -m ping` avant d'écrire ou d'adapter les playbooks.