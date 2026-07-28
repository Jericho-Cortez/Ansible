## Objectif du job

Le but de ce job est de mettre en place une surveillance de base, un audit de conformité et un premier durcissement système avec Ansible.
On doit aussi commencer à structurer le projet avec un rôle Ansible, afin de rendre l’ensemble plus propre et réutilisable.

## Architecture retenue

Pour garder un projet lisible, j’ai séparé les tâches en plusieurs playbooks et j’ai commencé à structurer le projet comme suit :

```text
ansible-startfleet/
├── inventory.ini
├── playbooks/
│   ├── 03-1-filebeat.yml
│   ├── 03-2-audit-conformite.yml
│   └── 03-3-hardening-kernel-pam.yml
├── roles/
│   └── hardening_base/
├── templates/
├── files/
└── group_vars/
```

J’ai choisi d’utiliser **Ansible Galaxy uniquement pour Filebeat**, afin de gagner du temps sur la partie standardisée tout en gardant la maîtrise des autres blocs du job.

### Installation du rôle Galaxy

```bash
ansible-galaxy role install geerlingguy.filebeat
```
![[Installation du rôle Galaxy.png]]
Ou via `requirements.yml` :

```bash
ansible-galaxy role install -r requirements.yml -p roles/
```

```yaml
roles:
  - name: geerlingguy.filebeat
```


***

## Déploiement de Filebeat

Le premier playbook du Job 3 installe et configure Filebeat sur toutes les machines cibles.
J’ai utilisé le rôle `geerlingguy.filebeat` pour déployer le service, installer le paquet, copier la configuration et le démarrer automatiquement.

```yaml
---
- name: Job 3 - Déploiement Filebeat
  hosts: all_servers
  become: true

  vars:
    filebeat_output_logstash_enabled: true
    filebeat_output_logstash_hosts:
      - "192.168.0.10:5044"
    filebeat_create_config: true
    filebeat_enable_logging: true
    filebeat_log_level: info
    filebeat_modules:
      - system

  roles:
    - geerlingguy.filebeat
```
![[Déploiement de Filebeat partie 1.png]]
![[Déploiement de Filebeat Partie 2.png]]


### Ce qu’il faut retenir

Filebeat collecte et envoie les logs, mais il ne les analyse pas.
Dans ce Job 3, Filebeat est configuré et testé dans un environnement de démonstration avec une destination de collecte de test. La mise en place d’un scénario web réel avec logs applicatifs et collecte complète sera réalisée dans le Job 5.

### Validation Filebeat

```bash
systemctl status filebeat
filebeat test config
filebeat test output
```
![[Validation Filebeat.png]]

***

## Audit de conformité

Le deuxième playbook réalise un audit de conformité non intrusif.
Il vérifie les fichiers système critiques, les permissions, les comptes avec mot de passe vide et les répertoires trop permissifs.

```yaml
---
- name: Job 3 - Audit de conformité
  hosts: all_servers
  become: true

  tasks:
    - name: Vérifier les fichiers critiques
      ansible.builtin.stat:
        path: "{{ item }}"
      loop:
        - /etc/passwd
        - /etc/shadow
        - /etc/sudoers
        - /etc/ssh/sshd_config
      register: critical_files

    - name: Afficher le statut des fichiers critiques
      ansible.builtin.debug:
        var: critical_files.results

    - name: Détecter les comptes avec mot de passe vide
      ansible.builtin.shell: "awk -F: '($2 == \"\") { print $1 }' /etc/shadow"
      register: empty_password_users
      changed_when: false
      failed_when: false

    - name: Afficher les comptes à mot de passe vide
      ansible.builtin.debug:
        var: empty_password_users.stdout_lines

    - name: Rechercher les répertoires trop permissifs
      ansible.builtin.shell: "find / -type d -perm 0777 2>/dev/null | head -n 50"
      register: permissive_dirs
      changed_when: false
      failed_when: false

    - name: Afficher les répertoires trouvés
      ansible.builtin.debug:
        var: permissive_dirs.stdout_lines
```

![[Validation Audit de conformité partie 1.png]]
![[Validation Audit de conformité partie 2.png]]
### Résultat

L’audit n’a remonté aucune anomalie sur les deux machines cibles : les fichiers critiques existent, leurs permissions sont correctes, aucun mot de passe vide n’a été trouvé et aucun dossier en `777` n’a été détecté.

***

## Durcissement noyau et mot de passe

Le troisième playbook applique un durcissement système sur deux axes : les paramètres noyau et la politique de mot de passe.  
J’ai choisi d’utiliser `ansible.posix.sysctl` au lieu de modifier directement `/etc/sysctl.conf`, car ce fichier n’existait pas sur mes machines cibles.  
Cette approche est plus propre et plus robuste.

Le playbook a ensuite été simplifié pour appeler un rôle dédié, `hardening_base`, qui contient maintenant l’ensemble des tâches de durcissement.  
Cela permet de séparer clairement la logique du playbook et le contenu technique du durcissement, tout en rendant le projet plus réutilisable.

```yaml
---
- name: Job 3 - Durcissement du noyau et des politiques de mot de passe
  hosts: all_servers
  become: true
  gather_facts: true

  roles:
    - hardening_base
```
## Lignes importantes

- `hosts: all_servers` : le playbook s’applique à toutes les machines cibles.
    
- `become: true` : indispensable pour modifier les réglages système.
    
- `gather_facts: true` : permet de récupérer les informations système avant l’exécution.
    
- `roles:` : appelle le rôle `hardening_base` qui contient les tâches de durcissement.
    
- `hardening_base` : centralise les réglages noyau et mot de passe dans un seul rôle réutilisable.
    

## Contenu du rôle `hardening_base`

Les tâches suivantes sont désormais déplacées dans `roles/hardening_base/tasks/main.yml` :

- activer `tcp_syncookies`,
    
- désactiver `ip_forward`,
    
- forcer une longueur minimale de mot de passe,
    
- limiter la durée maximale des mots de passe,
    
- imposer un délai minimum entre deux changements.
![[tasks main.yml.png]]
pour exécuter les tâches de durcissement.
![[defaults main.yml.png]]
si tes tâches utilisent des variables par défaut.
![[handlers main.yml.png]]
uniquement si une tâche déclenche un handler.
### Validation

```bash
ansible-playbook -i ../inventory.ini 03-3-hardening-kernel-pam.yml
ansible all_servers -i ../inventory.ini -b -a "sysctl net.ipv4.tcp_syncookies"
ansible all_servers -i ../inventory.ini -b -a "sysctl net.ipv4.ip_forward"
ansible all_servers -i ../inventory.ini -b -a "grep -E 'PASS_MIN_LEN|PASS_MAX_DAYS|PASS_MIN_DAYS' /etc/login.defs"
```
![[Check Validation 03-3-hardening-kernel-pam.yml.png]]
![[Validation 03-3-hardening-kernel-pam.yml.png]]
![[Validation 03-3-hardening-kernel-pam.yml Cible.png]]
## Résultat

	Cette refactorisation rend le playbook plus lisible, plus court et plus facile à maintenir.  
	Elle montre aussi une bonne séparation entre le niveau “orchestration” du playbook et le niveau “implémentation” du rôle.

***

## Rôle `hardening_base`

Le job 3 est aussi l’occasion de commencer à structurer le projet avec un rôle Ansible.
L’objectif est de préparer une base réutilisable pour les tâches de durcissement.
### Structure du rôle

```text
roles/hardening_base/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
└── vars/
    └── main.yml
```


### Ce que je peux y mettre

- les paramètres `sysctl`,
- des paramètres de sécurité système,
- des tâches communes de durcissement,
- éventuellement des variables réutilisables.


### Pourquoi c’est important

Un rôle permet d’encapsuler une responsabilité claire, testable et réutilisable.
C’est une étape importante vers un projet Ansible plus professionnel.

***

## Vault et secrets

Le Job 3 est aussi un bon endroit pour introduire **Ansible Vault** si je veux chiffrer des variables sensibles dans le projet.

### Commandes utiles

```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
```
![[Contenue du Vault.png]]
Le fichier `vault.yml` a été créé pour stocker les variables sensibles du projet de manière chiffrée.
![[Contenue Vault clair.png]]
Le fichier Vault contient les secrets utilisés dans le job, notamment ceux liés à Filebeat et au hash administrateur.
![[Modification 03-1 pour le vault.png]]
La variable chiffrée est réutilisée dans le playbook Filebeat pour éviter de laisser un secret en clair dans le code.
![[Cat Vault.png]]
Un mot de passe fictif peut être stocké dans Vault pour simuler une configuration sensible sans exposer la valeur dans le dépôt.
Cette variable sensible montre comment protéger une information de configuration qui ne doit pas apparaître en clair.
La commande `ansible-vault view` permet de vérifier le contenu du fichier chiffré après saisie du mot de passe Vault.

![[Verification du fichier modifier avec le vault.png]]
### Cas d’usage

Je peux y stocker :

- un mot de passe fictif,
- une variable sensible,
- un secret lié à Filebeat,
- une clé d’API de test.

***

## Bilan du Job 3

Ce job m’a permis de mettre en place :

- un déploiement de Filebeat avec Galaxy,
- un audit de conformité propre,
- un durcissement noyau et mot de passe validé,
- et un début de structuration avec un rôle Ansible.

Le principal apprentissage est qu’il faut séparer clairement la **collecte de logs**, l’**audit** et la **remédiation**, afin de garder un projet lisible et maîtrisable.
J’ai aussi appris qu’un playbook de durcissement doit parfois être adapté à la distribution cible, notamment pour `sysctl` et les fichiers de configuration système.

***

## Ce qu’il faut retenir

- Filebeat est déployé et fonctionnel sur les cibles.
- L’audit de conformité est propre.
- Le durcissement noyau et mot de passe est validé.
- Le projet commence à être organisé en rôles.
- Le `connection refused` sur Filebeat est attendu dans le cadre d’une destination fictive.

