## Introduction
Le Job 5 constitue la phase d'intégration finale du parcours : il demande de réutiliser les compétences acquises dans les jobs précédents pour réaliser un déploiement web sécurisé, améliorer l'organisation du projet Ansible et produire un bilan synthétique avec support de présentation. Le cahier des charges impose en particulier le déploiement d'une page HTML statique via Nginx, l'usage de HTTPS avec certificat auto-signé, la collecte des logs Nginx par Filebeat, l'application des mesures de hardening existantes et une revue globale de l'architecture des playbooks et des rôles.

## Environnement de TP

L'environnement de TP repose sur un projet Ansible nommé `ansible-starfleet`, organisé autour d'un fichier `inventory.ini`, d'un dossier `playbooks`, d'un dossier `roles`, de templates Jinja2 et d'un usage progressif d'Ansible Vault pour les variables sensibles.Les jobs précédents ont été séparés en plusieurs playbooks spécialisés, ce qui a permis de garder une logique modulaire et de limiter les risques lors des opérations sensibles comme le durcissement SSH, l'activation du pare-feu ou les tests de conformité.

L'architecture déjà documentée contient notamment les playbooks du Job 2 pour les utilisateurs, SSH, UFW et la désactivation de services, ceux du Job 3 pour Filebeat, l'audit de conformité et le durcissement noyau/mots de passe, ainsi que ceux du Job 4 pour Suricata, la réponse à incident, Vault et les facts système. Cette progression montre que le Job 5 ne doit pas être conçu comme un bloc isolé, mais comme une couche finale de consolidation et de démonstration.

## Architecture Ansible retenue

L'organisation actuelle du projet est cohérente avec la recommandation de séparer les responsabilités par playbook et de préparer une réutilisation via des rôles Ansible. Le Job 3 mentionne explicitement l'introduction d'un rôle `hardeningbase` destiné à centraliser les tâches communes de durcissement, tandis que Filebeat a déjà été déployé via un rôle Galaxy afin de conserver une structure propre et évolutive.

Une arborescence cohérente pour le Job 5, alignée avec le reste du dépôt, est la suivante :

```text
ansible-starfleet/
├── inventory.ini
├── group_vars/
│   └── all/
│       └── vault.yml
├── playbooks/
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
│   └── hardeningbase/
│       ├── defaults/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
├── templates/
│   ├── filebeat.yml.j2
│   ├── index.html.j2
│   └── nginx-secure-site.conf.j2
└── files/
```

Cette structure respecte l'objectif du sujet qui demande un projet bien organisé, avec des rôles bien appliqués, des templates Jinja2, des variables réutilisables, des blocs pour regrouper les tâches et des handlers pour les services.

## Déploiement sécurisé demandé

Le scénario de déploiement sécurisé demandé dans le Job 5 consiste à publier une page HTML statique sur un serveur web via Nginx, en activant HTTPS avec un certificat auto-signé, puis en s'assurant que les journaux générés par Nginx sont bien collectés par Filebeat. Le serveur ciblé doit aussi recevoir les mesures de hardening élaborées précédemment, ce qui justifie l'appel du rôle `hardeningbase` avant l'installation ou la configuration des composants web.

Le playbook principal du Job 5 peut être placé dans `playbooks/05-1-secure-webapp.yml` et structuré autour d'un bloc principal, avec handlers pour Nginx et Filebeat :

```yaml
---
- name: Job 5 - Déploiement sécurisé d'une application web simple
  hosts: webservers
  become: true
  gather_facts: true

  vars:
    web_root: /var/www/secure-site
    nginx_site_name: secure-site
    nginx_site_conf: /etc/nginx/sites-available/secure-site
    nginx_site_link: /etc/nginx/sites-enabled/secure-site
    tls_dir: /etc/nginx/ssl
    tls_cert: /etc/nginx/ssl/selfsigned.crt
    tls_key: /etc/nginx/ssl/selfsigned.key
    nginx_service_name: nginx
    page_title: "Starfleet Secure Web"
    page_message: "Application web statique déployée et sécurisée avec Ansible"
    server_name: "_"

  pre_tasks:
    - name: Appliquer le rôle de hardening de base
      ansible.builtin.include_role:
        name: hardeningbase

  tasks:
    - name: Bloc de déploiement web sécurisé
      block:
        - name: Installer Nginx et OpenSSL
          ansible.builtin.apt:
            name:
              - nginx
              - openssl
            state: present
            update_cache: true

        - name: Créer le répertoire racine du site
          ansible.builtin.file:
            path: "{{ web_root }}"
            state: directory
            owner: root
            group: root
            mode: '0755'

        - name: Créer le répertoire des certificats TLS
          ansible.builtin.file:
            path: "{{ tls_dir }}"
            state: directory
            owner: root
            group: root
            mode: '0755'

        - name: Déployer la page HTML statique
          ansible.builtin.template:
            src: ../templates/index.html.j2
            dest: "{{ web_root }}/index.html"
            owner: root
            group: root
            mode: '0644'
          notify: restart nginx

        - name: Générer le certificat auto-signé
          ansible.builtin.command:
            cmd: >
              openssl req -x509 -nodes -days 365
              -newkey rsa:2048
              -keyout {{ tls_key }}
              -out {{ tls_cert }}
              -subj "/C=FR/ST=IDF/L=Paris/O=Starfleet/OU=Security/CN={{ ansible_fqdn | default(inventory_hostname) }}"
          args:
            creates: "{{ tls_cert }}"
          notify: restart nginx

        - name: Déployer la configuration Nginx
          ansible.builtin.template:
            src: ../templates/nginx-secure-site.conf.j2
            dest: "{{ nginx_site_conf }}"
            owner: root
            group: root
            mode: '0644'
          notify: restart nginx

        - name: Activer le site Nginx
          ansible.builtin.file:
            src: "{{ nginx_site_conf }}"
            dest: "{{ nginx_site_link }}"
            state: link
          notify: restart nginx

        - name: Désactiver le site par défaut
          ansible.builtin.file:
            path: /etc/nginx/sites-enabled/default
            state: absent
          notify: restart nginx

        - name: Vérifier la configuration Nginx
          ansible.builtin.command: nginx -t
          changed_when: false

        - name: Ajouter la collecte des logs Nginx dans Filebeat
          ansible.builtin.blockinfile:
            path: /etc/filebeat/filebeat.yml
            marker: "# {mark} ANSIBLE NGINX LOGS"
            block: |
              filebeat.inputs:
                - type: filestream
                  id: nginx-access
                  enabled: true
                  paths:
                    - /var/log/nginx/access.log
                - type: filestream
                  id: nginx-error
                  enabled: true
                  paths:
                    - /var/log/nginx/error.log
          notify: restart filebeat

        - name: S'assurer que Nginx est démarré et activé
          ansible.builtin.service:
            name: "{{ nginx_service_name }}"
            state: started
            enabled: true

      rescue:
        - name: Afficher un message en cas d'échec
          ansible.builtin.debug:
            msg: "Le déploiement sécurisé a échoué sur {{ inventory_hostname }}"

      always:
        - name: Fin du playbook Job 5
          ansible.builtin.debug:
            msg: "Fin du déploiement sécurisé sur {{ inventory_hostname }}"

  handlers:
    - name: restart nginx
      ansible.builtin.service:
        name: "{{ nginx_service_name }}"
        state: restarted

    - name: restart filebeat
      ansible.builtin.service:
        name: filebeat
        state: restarted
```

Cette implémentation répond aux exigences du sujet sur les blocs, les handlers, l'usage de variables et l'intégration du hardening existant dans un déploiement final. Elle reste aussi compatible avec l'évolution progressive décrite dans les jobs précédents, où le projet a été construit sous forme de briques indépendantes puis réutilisables.

## Templates Jinja2 utilisés

Le Job 5 doit réutiliser le dossier `templates/` existant afin de rendre les fichiers de configuration plus génériques et plus faciles à maintenir. Le premier template correspond à la page HTML statique servie par Nginx :

index.html.j2

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{ page_title }}</title>
  <style>
    body {
      margin: 0;
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      background: #0f172a;
      color: #f8fafc;
      font-family: Arial, sans-serif;
    }
    .card {
      background: #1e293b;
      padding: 2rem;
      border-radius: 12px;
      box-shadow: 0 10px 30px rgba(0,0,0,0.35);
      max-width: 720px;
      text-align: center;
    }
    h1 {
      color: #38bdf8;
      margin-bottom: 1rem;
    }
    p {
      line-height: 1.6;
    }
    .meta {
      margin-top: 1rem;
      font-size: 0.95rem;
      color: #cbd5e1;
    }
  </style>
</head>
<body>
  <div class="card">
    <h1>{{ page_title }}</h1>
    <p>{{ page_message }}</p>
    <p class="meta">Hôte : {{ inventory_hostname }}</p>
    <p class="meta">OS : {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
  </div>
</body>
</html>
```

Le second template correspond à la configuration Nginx sécurisée :
nginx-secure-site.conf.j2

```nginx
server {
    listen 80;
    server_name {{ server_name }};
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name {{ server_name }};

    ssl_certificate {{ tls_cert }};
    ssl_certificate_key {{ tls_key }};

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    root {{ web_root }};
    index index.html;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

L'utilisation de templates est justifiée par les recommandations Ansible sur les variables et sur la substitution Jinja2, qui permettent de réutiliser un même modèle sur plusieurs systèmes tout en injectant des paramètres spécifiques à l'environnement.

## Réutilisation du rôle hardeningbase

Le Job 3 indiquait que le rôle `hardeningbase` devait commencer à regrouper les tâches communes de durcissement, notamment les paramètres `sysctl`, certaines politiques de sécurité système et des variables réutilisables. Dans le Job 5, ce rôle peut servir de point d'entrée unique pour appliquer automatiquement le durcissement minimal avant le déploiement de l'application web.

Un contenu simple et cohérent pour `roles/hardeningbase/tasks/main.yml` est le suivant :

```yaml
---
- name: Activer TCP SYN cookies
  ansible.posix.sysctl:
    name: net.ipv4.tcp_syncookies
    value: '1'
    state: present
    sysctl_set: true
    reload: true

- name: Désactiver IP forwarding
  ansible.posix.sysctl:
    name: net.ipv4.ip_forward
    value: '0'
    state: present
    sysctl_set: true
    reload: true

- name: Forcer une longueur minimale de mot de passe
  ansible.builtin.lineinfile:
    path: /etc/login.defs
    regexp: '^PASS_MIN_LEN'
    line: 'PASS_MIN_LEN 12'
    create: false

- name: Définir la durée maximale de mot de passe
  ansible.builtin.lineinfile:
    path: /etc/login.defs
    regexp: '^PASS_MAX_DAYS'
    line: 'PASS_MAX_DAYS 90'
    create: false

- name: Définir la durée minimale entre changements
  ansible.builtin.lineinfile:
    path: /etc/login.defs
    regexp: '^PASS_MIN_DAYS'
    line: 'PASS_MIN_DAYS 1'
    create: false
```

Cette version reprend directement des éléments déjà validés au Job 3, ce qui évite de dupliquer ces réglages dans le playbook du Job 5 et renforce la cohérence du projet.

## Mesures de sécurité mises en place

Le Job 5 permet de démontrer une chaîne de sécurité complète allant du durcissement de l'hôte jusqu'au service exposé.Les principales mesures à mettre en avant sont les suivantes :

- compte d'administration dédié, durcissement SSH, changement de port et désactivation de l'authentification par mot de passe issus du Job 2 ;
- pare-feu UFW avec politique restrictive et ouverture contrôlée des ports 2002, 80 et 443 ;
- durcissement du noyau et de la politique de mots de passe via le Job 3 ;
- déploiement de Nginx avec redirection HTTP vers HTTPS et certificat auto-signé ;
- collecte des journaux applicatifs Nginx par Filebeat afin d'étendre la visibilité de supervision au service web ;
- validation de configuration avant mise en production avec `nginx -t`, afin de limiter le risque de rupture de service lors du redémarrage.

Le sujet attend aussi une justification des choix techniques. Le changement de port SSH, la restriction du pare-feu et la suppression ou désactivation de services inutiles réduisent la surface d'exposition, tandis que le recours à des handlers évite les redémarrages inutiles et améliore l'idempotence globale des exécutions. L'ajout de HTTPS, même avec un certificat auto-signé, permet de démontrer la mise en place d'un canal chiffré et prépare une transition ultérieure vers un certificat signé par une autorité reconnue.

## Commandes d'exécution et validation

La commande principale pour exécuter le Job 5 reste alignée avec les autres jobs du projet :

```bash
ansible-playbook -i inventory.ini playbooks/05-1-secure-webapp.yml --ask-vault-pass
```
![[05-1-secure-webapp.yml.png]]
Une fois le playbook exécuté, plusieurs validations permettent de démontrer le bon fonctionnement du scénario demandé :

```bash
ansible webservers -i inventory.ini -b -a "systemctl status nginx"
ansible webservers -i inventory.ini -b -a "systemctl status filebeat"
ansible webservers -i inventory.ini -b -a "nginx -t"
ansible webservers -i inventory.ini -b -a "curl -k https://localhost"
curl -k https://IP_DU_SERVEUR_WEB
```
ansible webservers -i inventory.ini -b -a "systemctl status nginx"
![[systemctl status nginx.png]]
ansible webservers -i inventory.ini -b -a "systemctl status filebeat"
![[systemctl status filebeat.png]]
ansible webservers -i inventory.ini -b -a "nginx -t"
![[nginx -t.png]]
ansible webservers -i inventory.ini -b -a "curl -k https://localhost"
![[curl -k localhost.png]]
curl -k https://IP_DU_SERVEUR_WEB
![[Curl IP Web.png]]

Ces tests permettent de vérifier à la fois l'état des services, la validité de la configuration Nginx et l'accès réel à la page web via HTTPS, comme demandé dans le cahier des charges. L'usage de `curl -k` est normal ici, car le certificat est volontairement auto-signé et ne peut donc pas être validé par une autorité de confiance publique.

## Défis rencontrés et solutions apportées

Les difficultés les plus probables dans ce projet sont cohérentes avec celles déjà observées dans les jobs précédents. Le premier risque important reste la perte d'accès SSH lors du changement de port ou de la désactivation de l'authentification par mot de passe ; ce point a déjà été identifié comme critique dans le Job 2, avec la nécessité de valider l'accès du compte d'administration avant tout verrouillage supplémentaire.

Le second point de vigilance concerne Filebeat. Le Job 3 précise que Filebeat a été installé dans un contexte de destination fictive et que le scénario applicatif complet avec logs web devait justement être finalisé au Job 5. La solution retenue consiste à conserver Filebeat déjà présent, puis à enrichir sa configuration pour intégrer les journaux Nginx sans redéployer complètement l'agent.

Le troisième défi concerne la lisibilité du projet. Le parcours montre une montée en complexité progressive, avec un passage de playbooks simples à une architecture plus propre basée sur rôles, variables et templates. Le Job 5 sert précisément à rationaliser cette évolution, en montrant que le projet n'est plus une succession de scripts isolés mais un ensemble organisé, réutilisable et compréhensible.

## Bilan sur l'efficacité d'Ansible

Le principal apport d'Ansible dans ce projet est sa capacité à transformer des opérations sensibles et répétitives en procédures automatisées, relisibles et rejouables. Cela est particulièrement visible dans les tâches de hardening, la configuration de services, le déploiement de fichiers, l'usage de handlers pour limiter les redémarrages, et la possibilité de structurer le tout avec des rôles et des variables.

Le projet Starfleet montre aussi qu'Ansible est utile non seulement pour configurer des serveurs, mais également pour améliorer la méthode de travail : séparation des responsabilités, validation progressive, meilleure traçabilité et meilleure capacité à expliquer ce qui a été fait sur chaque machine. Pour un usage cybersécurité, cette approche est précieuse car elle réduit l'improvisation, renforce la cohérence et facilite les démonstrations de conformité ou de sécurisation.

## Support de présentation courte

Une présentation courte de soutenance peut tenir sur six diapositives : 

| Slide | Contenu attendu |
|------|------------------|
| 1 | Contexte du projet Starfleet, objectif du Job 5, intérêt de l'automatisation en cybersécurité [cite:4] |
| 2 | Environnement de TP et architecture Ansible du dépôt : inventaire, playbooks, rôles, templates [cite:3][cite:4] |
| 3 | Mesures de hardening déjà réalisées et réutilisées dans le déploiement final [cite:2][cite:3] |
| 4 | Déploiement de Nginx, HTTPS auto-signé, page HTML statique, intégration Filebeat pour les logs Nginx [cite:4][cite:3] |
| 5 | Défis rencontrés : SSH, firewall, Filebeat, structuration du projet ; solutions retenues [cite:2][cite:3] |
| 6 | Résultats obtenus, intérêt d'Ansible, ouverture possible vers des certificats réels, supervision plus complète ou CI/CD [cite:4] |

## Version synthétique pour la conclusion

Le Job 5 a permis d'intégrer les acquis des jobs précédents dans un scénario cohérent de mise en production sécurisée d'un service web simple. Le projet final combine hardening système, exposition contrôlée d'un service Nginx en HTTPS, journalisation via Filebeat et structuration plus propre du dépôt Ansible autour de playbooks spécialisés, de templates et d'un rôle réutilisable.
Au-delà du résultat technique, ce job montre qu'Ansible est particulièrement efficace pour standardiser les tâches d'administration et de sécurité, limiter les erreurs de manipulation et rendre un environnement de lab plus facile à maintenir, à expliquer et à faire évoluer.
