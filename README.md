# Debian 13 CIS Hardening (Ansible)

Ce projet propose un rôle Ansible pour appliquer les recommandations du
**CIS Benchmark Debian 13 (v1.0.0)**.

L'objectif est de fournir une base de durcissement. Contrairement à
certains scripts qui appliquent 100% des règles (et cassent la prod),
celui-ci est modulaire et pensé pour s'adapter aussi bien à des serveurs
qu'à des postes d'administration.

-   **Socle :** Debian 13 (Trixie).
-   **Approche :** "Whitelist" (tout est interdit sauf ce qui est
    autorisé).
-   **Stack Technique :**
    -   🔥 **Firewall :** Nftables (Pas d'UFW, syntaxe native).
    -   ⏱️ **Temps :** Chrony (Pas de ntp/systemd-timesyncd).
    -   🚫 **IPv6 :** Désactivé par défaut (Kernel + Grub).

------------------------------------------------------------------------

## 📂 Structure du projet

L'arborescence est découpée par sections logiques du CIS pour faciliter
la maintenance et la lecture.

``` text
roles/cis_hardening/
├── tasks/
│   ├── main.yml                 # Point d'entrée (Chef d'orchestre)
│   ├── section1_filesystem.yml  # Partitionnement, /tmp, noexec, modules
│   ├── section1_desktop.yml     # Gestion intelligente du GDM (GUI)
│   ├── section2_services.yml    # Nettoyage des services inutiles
│   ├── section2_time.yml        # Configuration Chrony
│   ├── section3_network.yml     # Sysctl, IPv6 kill-switch
│   ├── section4_nftables.yml    # Firewalling (Table Inet)
│   └── ...
└── defaults/
    └── main.yml                 # TOUTES les variables configurables sont ici
```

------------------------------------------------------------------------

## ⚙️ Variables & Configuration

Toute la configuration se passe dans
`roles/cis_hardening/defaults/main.yml`.

Voici les variables qui modifient drastiquement le comportement du
playbook.

### 🖥️ Mode Serveur vs Workstation

  -------------------------------------------------------------------------------------------------------
  Variable             Valeur par défaut                    Effet
  -------------------- ------------------------------------ ---------------------------------------------
  `cis_enable_gui`     `false`                              **False** : Supprime le paquet `gdm3` (Mode
                                                            Serveur).`<br>`{=html}`<br>`{=html}**True** :
                                                            Garde l'interface graphique mais sécurise la
                                                            bannière et masque la liste des utilisateurs.

  -------------------------------------------------------------------------------------------------------

### 🌐 Réseau & Firewall (Nftables)

Nous utilisons **nftables** avec une politique par défaut à `DROP`.

  -----------------------------------------------------------------------------------------
  Variable                           Valeur par défaut               Description
  ---------------------------------- ------------------------------- ----------------------
  `cis_disable_ipv6`                 `true`                          Désactive l'IPv6 au
                                                                     niveau noyau (sysctl)
                                                                     **ET** chargeur de
                                                                     démarrage (Grub).

  `cis_nftables_allowed_tcp_ports`   `[22]`                          Liste des ports TCP
                                                                     ouverts en entrée.
                                                                     **⚠️ Ne retirez pas le
                                                                     22 sous peine de vous
                                                                     enfermer dehors.**

  `cis_nftables_allowed_udp_ports`   `[]`                            Ports UDP ouverts (ex:
                                                                     `[123]` pour NTP
                                                                     serveur, `[53]` pour
                                                                     DNS).

  `cis_nftables_allow_icmp`          `true`                          Autorise le ping
                                                                     (echo-request). Mettre
                                                                     à `false` pour être
                                                                     furtif (mais pénible à
                                                                     debug).
  -----------------------------------------------------------------------------------------

### 🔒 Bootloader (GRUB)

Le CIS exige un mot de passe pour modifier les paramètres de boot
(empêche le `init=/bin/bash`).

  -----------------------------------------------------------------------
  Variable                       Description
  ------------------------------ ----------------------------------------
  `cis_grub_password_hash`       **CRITIQUE.** Vous devez générer ce hash
                                 avec `grub-mkpasswd-pbkdf2`. Si cette
                                 variable n'est pas définie, la
                                 sécurisation du GRUB est ignorée pour ne
                                 pas casser le boot.

  -----------------------------------------------------------------------

### 🛠️ Services (Whitelisting)

Par défaut, le script **désactive/supprime** une longue liste de
services (Avahi, CUPS, LDAP, RPC, HTTP...).

Pour activer un service, passez sa variable à `true` :

Exemple :

``` yaml
cis_allow_web_server: true   # Apache/Nginx
cis_allow_docker: true
```

------------------------------------------------------------------------

## 🚀 Utilisation

### 1. Lancer le durcissement complet

``` bash
ansible-playbook -i inventory/hosts site.yml
```

### 2. Lancer par modules (Recommandé)

``` bash
# Appliquer uniquement la couche réseau (IPv6, Sysctl)
ansible-playbook site.yml --tags "network"

# Appliquer uniquement le firewall (Nftables)
ansible-playbook site.yml --tags "firewall"

# Appliquer uniquement la conf SSH
ansible-playbook site.yml --tags "ssh"

# Juste la désactivation des services inutiles
ansible-playbook site.yml --tags "services"
```

------------------------------------------------------------------------

## ⚠️ Notes Techniques

1.  **Routage Interdit :** Le paramètre `net.ipv4.ip_forward` est forcé
    à `0`. Le serveur ne peut pas agir comme routeur/gateway.
2.  **Chrony :** Le paquet `ntp` est supprimé et `systemd-timesyncd` est
    masqué au profit de `chrony`.
3.  **Audit :** De nombreuses règles `auditd` sont activées. Surveillez
    `/var/log/audit/audit.log` si une application se comporte
    anormalement.
4.  **Partitionnement :** Le playbook vérifie les options de montage
    (`nodev`, `noexec`, `nosuid`) sur `/tmp` et `/var`, mais ne modifie
    pas le partitionnement disque physique.
