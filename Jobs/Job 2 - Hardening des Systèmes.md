---
title: Job 2 - Hardening des Systèmes
aliases:
  - Job 2 Ansible hardening détaillé
  - Hardening Debian Ansible Job 2
tags:
  - ansible
  - hardening
  - debian
  - ssh
  - ufw
  - systemd
  - security
  - obsidian
category: sysadmin
status: à retravailler
language: fr
created: 2026-07-20
updated: 2026-07-20
---
# Job 2 — Hardening des systèmes avec Ansible

> [!abstract] Résumé rapide
> Ce document détaille le **Job 2** autour du hardening de base sur Debian avec **Ansible** : gestion des utilisateurs, durcissement SSH, configuration UFW et désactivation de services inutiles.
> 
> L’objectif principal est de **sécuriser sans casser l’exploitation**, avec un ordre d’exécution sûr, des playbooks lisibles, idempotents et validables.

## Synthèse

> [!info] Objectif global
> Le Job 2 consiste à appliquer un **hardening système de base** sur des VMs Debian avec Ansible : sécuriser les comptes d’administration, verrouiller SSH, configurer le pare-feu et réduire les services inutiles.

> [!warning] Enjeu critique
> Le vrai besoin n’est pas seulement technique : il faut montrer que le durcissement est fait **sans perte d’accès**, avec un ordre d’action sûr, des validations intermédiaires et une logique d’exécution progressive.

## Résumé du besoin réel

Oui. Je vais te montrer **tout le Job 2 en détail**, avec l’objectif, les concepts, la stratégie, le squelette rempli des playbooks, l’ordre d’exécution, les validations et les points de vigilance.

Je reste aligné avec ton sujet et avec la logique du blog de Stéphane Robert sur la structure des playbooks, les handlers, l’idempotence et l’exécution progressive.

## Objectifs obligatoires

Le sujet impose 4 blocs :

- gestion sécurisée des utilisateurs et groupes
- durcissement de la configuration SSH
- configuration d’UFW
- désactivation de services inutiles

Il impose aussi des actions précises :

- création de `ansible_admin`
- mot de passe haché
- appartenance à `sudo` ou `wheel`
- `PermitRootLogin no`
- `PasswordAuthentication no`
- `Port 2002`
- `X11Forwarding no`
- ouverture UFW des ports `2002`, `80`, `443`
- désactivation de services comme `cups` ou `avahi-daemon`

## Contraintes et critères d’évaluation

La contrainte la plus critique est la **sécurité d’exécution**.

Si les changements SSH ou UFW sont appliqués dans le mauvais ordre, l’accès aux VMs peut être perdu. Pour ce type de configuration sensible, le blog de référence met en avant :

- les **handlers** pour le redémarrage conditionnel
- le **check mode**
- le **diff mode**
- une structure de play claire avec `hosts`, `become`, `vars`, `tasks` et `handlers`

Le critère implicite de qualité est donc :

- lisibilité
- idempotence
- sécurité de déploiement
- capacité à expliquer pourquoi chaque mesure existe

## Concepts que tu dois maîtriser

### 1. `become: true`

Tu modifies des fichiers et services système, donc toutes les tâches sensibles devront s’exécuter avec élévation de privilèges.

Le blog classe `become` parmi les mots-clés structurants d’un play.

### 2. Handler

Un handler est une tâche réactive qui ne s’exécute que si une tâche a réellement changé quelque chose.

C’est exactement le pattern recommandé pour `sshd` : on ne redémarre pas SSH à chaque exécution, seulement si `sshd_config` a changé.

### 3. Idempotence

Au deuxième run, si rien n’a changé, les tâches doivent majoritairement être en `ok`, pas en `changed`.

C’est un point central de la logique Ansible du parcours.

### 4. `lineinfile` vs `template`

Pour le Job 2, il est possible de rester sur `lineinfile` pour quelques directives SSH ciblées.

C’est plus simple à comprendre à ce stade. Plus tard, une bascule vers `template` permettra une structuration plus propre.

### 5. Validation avant redémarrage

Le blog insiste sur un point très important pour `sshd` : **valider la configuration avant de la rendre active**, afin d’éviter de casser l’accès au serveur.

Même sans utiliser immédiatement `template + validate`, cette logique doit rester en tête.

## Pièges et erreurs fréquentes

> [!danger] Risque majeur
> Le point le plus important est simple : **tu ne dois jamais couper l’accès admin avant d’avoir validé le nouvel accès admin**.

- désactiver l’authentification par mot de passe avant d’avoir validé la clé SSH de `ansible_admin`
- changer le port SSH avant d’avoir ouvert ce port dans le firewall
- activer UFW avant d’avoir ajouté les règles nécessaires
- supprimer des comptes Debian système au hasard
- modifier `sshd_config` sans handler
- écrire un gros playbook unique impossible à déboguer

## Architecture ou stratégie recommandée

Je recommande un découpage en **4 playbooks distincts**, un par sous-partie du Job 2.

C’est plus propre, plus testable et plus conforme à l’esprit “playbook bien structuré” du blog.

```text
playbooks/
├── 02-1-users.yml
├── 02-2-ssh-hardening.yml
├── 02-3-firewall.yml
└── 02-4-disable-services.yml
```

### Pourquoi cette stratégie

- tu peux exécuter, corriger et valider chaque brique séparément
- tu limites le risque de lockout
- tu expliques mieux ton travail en entretien
- tu prépares déjà la future transition vers un rôle `hardening_base`

## Ce qui est indispensable, recommandé et optionnel

| Niveau | Éléments |
|---|---|
| Indispensable | `ansible_admin` créé et admin ; les 4 paramètres SSH demandés ; UFW avec règles demandées ; désactivation d’au moins quelques services non essentiels |
| Recommandé | clé SSH pour `ansible_admin` ; handler SSH ; tests après chaque playbook ; `--check` et `--diff` sur les parties sensibles |
| Optionnel | tags par bloc ; regroupement final dans un playbook global ; migration future vers rôle Ansible |

## Compare les solutions possibles

| Sujet | Option simple | Option plus propre | Recommandation |
|---|---|---|---|
| SSH config | `lineinfile` | `template` | `lineinfile` pour Job 2 |
| Structure | 1 gros playbook | 4 playbooks | 4 playbooks |
| Redémarrage SSH | tâche normale | handler | handler |
| Comptes à supprimer | suppression agressive | suppression ciblée et justifiée | ciblée et prudente |

---

#  détaillé du Job 2

## 1. Playbook `02-1-users.yml`

### Objectif

Créer l’utilisateur `ansible_admin`, lui donner les droits sudo et préparer une gestion prudente des comptes à supprimer.

### Concepts clés

- module `user`
- mot de passe haché
- groupe `sudo` sur Debian
- optionnel mais conseillé : clé publique SSH

### Stratégie

On ne supprime pas des comptes système Debian “par défaut” sans justification.

On crée d’abord le compte admin propre, puis on traite les comptes candidats à suppression avec une liste explicite.

### Squelette rempli

```yaml
---
- name: Job 2 - Gestion sécurisée des utilisateurs et groupes
  hosts: all_servers
  become: true
  gather_facts: true

  vars:
    admin_user: ansible_admin
    admin_group: sudo
    admin_shell: /bin/bash
    admin_password_hash: "$6$CHANGE_ME_HASH"
    users_to_remove:
      - tempuser
      - testuser

  tasks:
    - name: Créer le groupe admin si nécessaire
      ansible.builtin.group:
        name: "{{ admin_group }}"
        state: present
      tags: [users]

    - name: Créer l'utilisateur administrateur dédié
      ansible.builtin.user:
        name: "{{ admin_user }}"
        password: "{{ admin_password_hash }}"
        shell: "{{ admin_shell }}"
        groups: "{{ admin_group }}"
        append: true
        create_home: true
        state: present
      tags: [users]

    - name: Supprimer les utilisateurs non souhaités
      ansible.builtin.user:
        name: "{{ item }}"
        state: absent
        remove: true
      loop: "{{ users_to_remove }}"
      tags: [users]
```

### Explication

- `gather_facts: true` est utile ici car tu gardes un playbook standard et prêt à évoluer.
- `admin_password_hash` doit être un hash SHA-512, pas un mot de passe en clair.
- `users_to_remove` doit contenir **uniquement des comptes sûrs à supprimer**, idéalement des comptes de lab créés par toi.

### Ce qu’il manque pour être vraiment propre

- le déploiement de la clé publique de `ansible_admin`
- une liste plus intelligente des utilisateurs candidats
- éventuellement un `assert` pour éviter de supprimer un compte critique

### Comment faire le mot de passe haché

Sur le contrôleur :

```bash
openssl passwd -6
```

![[Hash mdp.png]]

![[Check 02-1-user.yml.png]]
![[02-1-users.yml OK.png]]
### Validation

```bash
ansible all_servers -i inventory.ini -a "id ansible_admin" -b
ansible all_servers -i inventory.ini -a "getent passwd ansible_admin" -b
```
![[Validation 02-1-users.yml.png]]
## 2. Playbook `02-2-ssh-hardening.yml`

### Objectif

Appliquer les 4 paramètres demandés dans `sshd_config` et redémarrer SSH uniquement si nécessaire.

### Concepts clés

- `lineinfile`
- `notify`
- handler
- test de reconnexion
- ordre de sécurité

### Stratégie

On modifie uniquement les lignes nécessaires, on notifie un handler, puis on teste.

On ne coupe **jamais** le mot de passe avant d’avoir validé que la clé SSH fonctionne pour `ansible_admin`.

### Squelette rempli

```yaml
---
- name: Job 2 - Durcissement SSH
  hosts: all_servers
  become: true
  gather_facts: true

  vars:
    ssh_port: 2002
    ssh_service_name: ssh

  tasks:
    - name: Désactiver la connexion root
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
        backup: true
      notify: Restart SSH
      tags: [ssh]

    - name: Désactiver l'authentification par mot de passe
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
        backup: true
      notify: Restart SSH
      tags: [ssh]

    - name: Changer le port SSH
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?Port'
        line: 'Port {{ ssh_port }}'
        state: present
        backup: true
      notify: Restart SSH
      tags: [ssh]

    - name: Désactiver X11Forwarding
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?X11Forwarding'
        line: 'X11Forwarding no'
        state: present
        backup: true
      notify: Restart SSH
      tags: [ssh]

  handlers:
    - name: Restart SSH
      ansible.builtin.service:
        name: "{{ ssh_service_name }}"
        state: restarted
```

### Explication

- `backup: true` est très utile sur un fichier critique.
- `notify: Restart SSH` suit la logique des handlers : le service ne redémarre que si une ligne a réellement changé.
- le handler évite de redémarrer SSH 4 fois.

### Point critique

Avant de lancer ce playbook, tu dois être capable de te connecter en clé SSH avec le bon utilisateur.

Sinon `PasswordAuthentication no` est dangereux.

![[Check 02-2-ssh-hardening.yml.png]]
![[02-2-ssh-hardening.yml OK.png]]
### Validation après exécution

Depuis le contrôleur :

```bash
ssh -p 2002 ansible_admin@IP_DE_LA_VM
```
![[Validation 02-2-users.hardening.yml.png]]
Sur la VM :

```bash
sudo ss -tlnp | grep 2002
sudo systemctl status ssh
```
![[Validation 02-2-users.hardening.yml Cible.png]]
### Variante plus propre à terme

Plus tard, on pourra faire un `template` avec validation `sshd -t`, ce que le blog recommande pour les services critiques.

## 3. Playbook `02-3-firewall.yml`

### Objectif

Installer UFW et appliquer la politique de filtrage demandée par le sujet.

### Concepts clés

- module `community.general.ufw` ou `ufw` selon ton environnement
- ordre des règles
- activation en dernier

### Stratégie

On ajoute d’abord les autorisations nécessaires, puis les politiques, puis on active UFW.

### Squelette rempli

```yaml
---
- name: Job 2 - Configuration du pare-feu UFW
  hosts: all_servers
  become: true
  gather_facts: true

  vars:
    ssh_port: 2002

  tasks:
    - name: Installer UFW
      ansible.builtin.apt:
        name: ufw
        state: present
        update_cache: true
      tags: [firewall]

    - name: Autoriser le port SSH personnalisé
      community.general.ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp
      tags: [firewall]

    - name: Autoriser HTTP
      community.general.ufw:
        rule: allow
        port: '80'
        proto: tcp
      tags: [firewall]

    - name: Autoriser HTTPS
      community.general.ufw:
        rule: allow
        port: '443'
        proto: tcp
      tags: [firewall]

    - name: Définir la politique par défaut pour les entrées
      community.general.ufw:
        direction: incoming
        policy: deny
      tags: [firewall]

    - name: Définir la politique par défaut pour les sorties
      community.general.ufw:
        direction: outgoing
        policy: allow
      tags: [firewall]

    - name: Activer UFW
      community.general.ufw:
        state: enabled
      tags: [firewall]
```

MAJ Inventory
![[MAJ inventory.ini.png]]
### Explication

- on installe d’abord UFW
- on autorise les ports
- on applique les politiques
- on active à la fin

### Point de vigilance

Si tu actives UFW avant d’avoir autorisé `2002/tcp`, tu risques le lockout.

![[02-3-firewall.yml OK.png]]
### Validation

```bash
sudo ufw status verbose
```

Ou via Ansible :

```bash
ansible all_servers -i inventory.ini -a "ufw status verbose" -b
```
![[Validation 02-3-firwall.yml.png]]
## 4. Playbook `02-4-disable-services.yml`

### Objectif

Désactiver et masquer les services non essentiels au rôle serveur.

### Concepts clés

- `service`
- `enabled: false`
- `masked: true`
- tolérance si service absent

### Stratégie

On travaille avec une liste de services candidats, mais on reste prudent.

Sur Debian minimal, certains services ne seront pas installés.

### Squelette rempli

```yaml
---
- name: Job 2 - Désactivation des services inutiles
  hosts: all_servers
  become: true
  gather_facts: true

  vars:
    unwanted_services:
      - cups
      - avahi-daemon

  tasks:
    - name: Désactiver et masquer les services non essentiels
      ansible.builtin.systemd:
        name: "{{ item }}"
        state: stopped
        enabled: false
        masked: true
      loop: "{{ unwanted_services }}"
      ignore_errors: true
      tags: [services]
```
![[02-4-disable-services.yml OK.png]]

### Explication

- `systemd` est plus adapté ici que `service`
- `masked: true` empêche un redémarrage involontaire
- `ignore_errors: true` évite l’échec si le service n’existe pas, mais ce n’est pas la solution la plus élégante

### Variante plus propre

Plus tard, on pourra faire un contrôle préalable avec `service_facts`, puis agir uniquement sur les services présents.

### Validation

```bash
systemctl status cups
systemctl status avahi-daemon
systemctl is-enabled cups
```
![[Validation 02-4-disable-services.yml.png]]

## Structure finale du dossier

```text
ansible-starfleet/
├── inventory.ini
├── playbooks/
│   ├── 02-1-users.yml
│   ├── 02-2-ssh-hardening.yml
│   ├── 02-3-firewall.yml
│   └── 02-4-disable-services.yml
└── files/
```

## Notes de vigilance

> [!warning] Séquence critique SSH/UFW
> Tu ne coupes jamais l’authentification par mot de passe avant d’avoir validé l’accès par clé avec l’utilisateur admin sur le nouveau port.[^1]

> [!tip] Bonne pratique
> Séparer les actions en plusieurs playbooks rend le debug, les tests et la démonstration en entretien beaucoup plus simples.

## Ce qui est fait

Le Job 2 est maintenant entièrement cadré, détaillé, découpé et illustré avec des squelettes remplis.

## Ce qui reste à faire

Il faut maintenant passer de la stratégie au **vrai YAML adapté à ton lab**, avec tes IP, ton utilisateur actuel, ton hash de mot de passe et éventuellement l’ajout de la clé SSH pour `ansible_admin`.

## Points de vigilance

Le point le plus dangereux reste la séquence SSH/UFW.

Tu ne coupes jamais l’authentification par mot de passe avant d’avoir validé l’accès par clé avec l’utilisateur admin sur le nouveau port.
