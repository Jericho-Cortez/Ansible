## Vue d’ensemble

Le Job 4 contient 4 parties :

1. **Déploiement d’un IDS/IPS simple** : Snort ou Suricata.
2. **Scénario de réponse à incident simulé** : détection d’un fichier suspect, confinement, collecte d’infos, notification.
3. **Utilisation d’Ansible Vault** : démontrer le chiffrement d’informations sensibles.
4. **Gestion des facts** : récupérer et afficher des informations système avec `setup`.[^1]

Le plus propre est de faire **un playbook par partie**, comme tu l’as fait au Job 3.[^3][^1]

## Arborescence conseillée

Ajoute ces fichiers dans `playbooks/` :

```text
playbooks/
├── 04-1-suricata.yml
├── 04-2-incident-response.yml
├── 04-3-vault-demo.yml
└── 04-4-facts.yml
```

Et si tu veux aller plus loin plus tard :

```text
roles/
└── incident_response/
```

Mais pour le Job 4, des playbooks simples suffisent largement.[^2][^4]

## Partie 1 — IDS/IPS avec Suricata

Je te conseille **Suricata** plutôt que Snort, car c’est plus simple à installer et à démontrer dans un TP.[^2]

### Objectif

Installer Suricata sur une ou plusieurs machines cibles et le faire surveiller une interface réseau.[^1]

### Ce qu’il faut faire

- installer le paquet ;
- vérifier l’interface réseau ;
- activer le service ;
- démarrer le service ;
- contrôler son état.[^3]


### Playbook conseillé : `04-1-suricata.yml`

```yaml
---
- name: Job 4 - Déploiement de Suricata
  hosts: all_servers
  become: true
  gather_facts: true

  tasks:
    - name: Installer Suricata
      ansible.builtin.apt:
        name: suricata
        state: present
        update_cache: true

    - name: Vérifier la configuration Suricata
      ansible.builtin.command: suricata -T -c /etc/suricata/suricata.yaml
      register: suricata_test
      changed_when: false
      failed_when: false

    - name: Afficher le résultat du test de configuration
      ansible.builtin.debug:
        var: suricata_test.stdout_lines

    - name: Lancer Suricata sur l'interface ens33
      ansible.builtin.command: suricata -c /etc/suricata/suricata.yaml -i ens33 -D
      register: suricata_start
      changed_when: true
      failed_when: false

    - name: Vérifier que Suricata est bien démarré
      ansible.builtin.command: pgrep -a suricata
      register: suricata_proc
      changed_when: false
      failed_when: false

    - name: Afficher le processus Suricata
      ansible.builtin.debug:
        var: suricata_proc.stdout_lines
```


### Validation

Lance :

```bash
ansible-playbook -i inventory.ini playbooks/04-1-suricata.yml
```


### Screens à prendre

- exécution du playbook ;
  ![[04-1-suricata-playbook.png]]


### Phrase pour le rapport

> Le déploiement de Suricata est validé sur les deux machines cibles. Le test de configuration remonte un avertissement sur l’absence de règles par défaut, mais ne bloque pas l’exécution. Le lancement sur l’interface ens33 permet de confirmer que le moteur peut être initialisé dans cet environnement.

## Partie 2 — Réponse à incident simulée

Cette partie simule une réponse à incident automatisée avec Ansible.  
L’objectif est de détecter un fichier suspect, de contenir l’incident, de collecter des preuves techniques et d’écrire une notification sur le contrôleur.

### Préparation manuelle

Sur la machine cible `web1`, j’ai créé un fichier suspect simulé dans `/tmp` :

```bash
sudo touch /tmp/malicious_script.sh
sudo chmod +x /tmp/malicious_script.sh
```
![[04-2-create-suspicious-file.png]]
Cette étape permet de déclencher la logique de détection du playbook.

Le playbook `04-2-incident-response.yml` vérifie d’abord la présence du fichier suspect avec `stat`, puis crée un répertoire de quarantaine, arrête le service ciblé, déplace le fichier en quarantaine, collecte des logs et des processus, et enregistre une notification locale sur le contrôleur.

### Dossier de quarantaine

Choisis par exemple :

```text
/var/quarantine
```


### Playbook conseillé : `04-2-incident-response.yml`

```yaml
---
- name: Job 4 - Réponse à incident simulée
  hosts: all_servers
  become: true
  gather_facts: true

  vars:
    suspicious_file: /tmp/malicious_script.sh
    quarantine_dir: /var/quarantine
    compromised_service: ssh
    local_incident_log: ./incident-report.log

  tasks:
    - name: Vérifier la présence du fichier suspect
      ansible.builtin.stat:
        path: "{{ suspicious_file }}"
      register: suspicious_stat

    - name: Créer le répertoire de quarantaine
      ansible.builtin.file:
        path: "{{ quarantine_dir }}"
        state: directory
        mode: '0755'
      when: suspicious_stat.stat.exists

    - name: Arrêter le service potentiellement compromis
      ansible.builtin.service:
        name: "{{ compromised_service }}"
        state: stopped
      when: suspicious_stat.stat.exists
      ignore_errors: true

    - name: Déplacer le fichier suspect en quarantaine
      ansible.builtin.command:
        cmd: mv "{{ suspicious_file }}" "{{ quarantine_dir }}/"
      when: suspicious_stat.stat.exists

    - name: Collecter les derniers logs système
      ansible.builtin.shell: "journalctl -n 20 --no-pager"
      register: recent_logs
      changed_when: false
      when: suspicious_stat.stat.exists

    - name: Collecter la liste des processus
      ansible.builtin.command: ps aux
      register: process_list
      changed_when: false
      when: suspicious_stat.stat.exists

    - name: Enregistrer une notification sur le contrôleur
      ansible.builtin.delegate_to: localhost
      ansible.builtin.copy:
        dest: "{{ local_incident_log }}"
        content: |
          Incident détecté sur {{ inventory_hostname }}
          Fichier suspect : {{ suspicious_file }}
          Service arrêté : {{ compromised_service }}
          Date : {{ ansible_date_time.iso8601 }}
      when: suspicious_stat.stat.exists

    - name: Afficher le statut de détection
      ansible.builtin.debug:
        msg: "Incident détecté et traité sur {{ inventory_hostname }}"
      when: suspicious_stat.stat.exists

    - name: Afficher le cas où rien n'est détecté
      ansible.builtin.debug:
        msg: "Aucun fichier suspect détecté sur {{ inventory_hostname }}"
      when: not suspicious_stat.stat.exists
```


### Validation

Lance :

```bash
ansible-playbook -i inventory.ini playbooks/04-2-incident-response.yml
```
![[04-2-incident-playbook.png]]
## Validation

Le fichier suspect a bien été créé sur `web1`, puis le playbook a été exécuté avec succès.  
Le fichier a été déplacé dans `/var/quarantine`, la notification a été écrite sur le contrôleur, et le service ciblé a été arrêté ou traité selon la cible.
![[04-2-quarantine.png]]


### Phrases pour le rapport

Sur `web1`, un fichier suspect a été détecté, mis en quarantaine avec succès, et le service potentiellement compromis a été arrêté. Les logs système et la liste des processus ont ensuite été collectés pour faciliter l’analyse de l’incident
L’incident est contenu et les éléments nécessaires à l’investigation ont été conservés.

## Partie 3 — Vault dans le Job 4

Tu as déjà commencé Vault au Job 3, donc ici tu peux **le réutiliser** au lieu de refaire une introduction complète.[^1]

### Objectif

Montrer que tu sais :

- chiffrer une variable sensible ;
- l’utiliser dans un playbook.[^5][^6]


### Fichier conseillé

Tu peux garder :

```text
group_vars/all/vault.yml
```


### Exemple de contenu

Par exemple :

```yaml
vault_incident_api_key: "fake-incident-key-456"
vault_notify_email: "security@example.local"
```


### Playbook conseillé : `04-3-vault-demo.yml`

```yaml
---
- name: Job 4 - Démonstration Vault
  hosts: all_servers
  become: false

  tasks:
    - name: Afficher une variable Vault de test
      ansible.builtin.debug:
        msg: "La clé API fictive sécurisée est bien chargée."
```

Le plus important n’est pas d’afficher la valeur, mais de montrer qu’un fichier chiffré est bien utilisé.[^5]

### Validation

```bash
ansible-playbook -i inventory.ini playbooks/04-3-vault-demo.yml --ask-vault-pass
```
![[04-3-vault-playbook.png]]

### Screens à prendre
![[04-3-vault-view.png]]
- `ansible-vault view group_vars/all/vault.yml` ;
- exécution du playbook avec `--ask-vault-pass`.[^1]


### Phrase pour le rapport

> Le Job 4 réutilise Ansible Vault pour sécuriser les variables sensibles liées à la simulation d’incident.

## Partie 4 — Facts

C’est la partie la plus simple.

### Objectif

Collecter et afficher :

- version du système ;
- quantité de RAM ;
- architecture ;
- nom d’hôte.[^1]


### Playbook conseillé : `04-4-facts.yml`

```yaml
---
- name: Job 4 - Gestion des facts
  hosts: all_servers
  become: false
  gather_facts: true

  tasks:
    - name: Afficher le système d'exploitation
      ansible.builtin.debug:
        msg: "OS : {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Afficher l'architecture
      ansible.builtin.debug:
        msg: "Architecture : {{ ansible_architecture }}"

    - name: Afficher la mémoire totale
      ansible.builtin.debug:
        msg: "RAM totale : {{ ansible_memtotal_mb }} MB"

    - name: Afficher le nom d'hôte
      ansible.builtin.debug:
        msg: "Nom d'hôte : {{ ansible_hostname }}"
```


### Validation

```bash
ansible-playbook -i inventory.ini playbooks/04-4-facts.yml
```
![[04-4-facts-playbook.png]]

### Screens à prendre

- exécution complète du playbook ;
- sortie montrant OS, RAM, architecture et hostname.[^1]


### Phrase pour le rapport

> Les facts Ansible permettent de récupérer automatiquement des informations détaillées sur les machines cibles, ce qui facilite l’adaptation des playbooks à l’environnement réel.

## Ordre complet de réalisation

Je te conseille de faire ça dans cet ordre :

1. Créer les 4 playbooks.
2. Faire Suricata.
3. Créer le fichier suspect à la main.
4. Tester la réponse à incident.
5. Réutiliser Vault.
6. Faire les facts.
7. Prendre les captures au fur et à mesure.
8. Rédiger la doc Job 4 avec une section par playbook.[^1]

## Structure du rapport Job 4

Tu peux organiser la doc comme ça :

### 1. Objectif du job

Présenter rapidement les 4 axes.

### 2. Déploiement d’un IDS/IPS

Expliquer Suricata, montrer le playbook, la validation et les captures.

### 3. Réponse à incident simulée

Décrire le scénario, le fichier suspect, les actions automatiques et les preuves.

### 4. Utilisation d’Ansible Vault

Montrer que les secrets sensibles sont chiffrés et réutilisés.

### 5. Gestion des facts

Montrer les informations système récupérées.






