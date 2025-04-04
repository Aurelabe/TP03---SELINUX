# Rapport : Préparation et durcissement du système d'exploitation

## 1. Installation du système d’exploitation

### 1.1 Installation de la version minimale de Linux

Pour garantir un serveur sécurisé et optimisé, nous avons choisi d’installer une version minimale de **Rocky Linux 9**, sans interface graphique. Cela permet de réduire la surface d’attaque en supprimant tous les paquets inutiles et en évitant les services non essentiels qui pourraient présenter des vulnérabilités.

### 1.2 Choix des paquets installés

Seuls les paquets nécessaires à l’utilisation du serveur ont été installés, conformément aux bonnes pratiques de sécurité. En optant pour une installation minimale, nous avons évité l’installation de paquets inutiles qui pourraient augmenter la complexité du système et potentiellement introduire des vulnérabilités.

### 1.3 Sélection de l’image système

Nous avons choisi d’utiliser la **dernière image officielle de Rocky Linux 9** pour garantir que le système soit à jour, avec les derniers correctifs de sécurité appliqués dès le départ. Cela permet de bénéficier des dernières mises à jour de sécurité et de stabilité proposées par la communauté de distribution.

### 1.4 Sécurisation du démarrage

L'ANSSI recommande de protéger le chargeur de démarrage avec un mot de passe pour prévenir des attaques physiques ou des modifications malveillantes au niveau du démarrage. Cette mesure est adaptée dans des environnements où l'accès physique aux serveurs n'est pas entièrement sécurisé. Dans notre cas, comme le serveur sera principalement accessible via SSH et dans un environnement contrôlé, cette mesure de protection du chargeur de démarrage ne semble pas indispensable. Toutefois, elle peut être mise en place si le serveur est physiquement exposé à des risques d'accès non autorisé.

## 2. Partitionnement du disque

### 2.1 Structure des partitions

Conformément aux recommandations de l'ANSSI et en prenant en compte les spécificités du serveur, nous avons partitionné le disque en respectant les bonnes pratiques. Voici la configuration retenue pour le partitionnement et les groupes de volumes (LVM) :

| Partition   | Taille  | Type  | Système de fichiers | Description |
|-------------|---------|-------|---------------------|-------------|
| **/boot**   | 1024 MiB | Partition standard | ext4 | Partition de démarrage, nécessaire pour charger le système d’exploitation. |
| **/**       | 15 Gio   | LVM   | XFS                 | Volume racine contenant tous les fichiers système et logiciels. |
| **/home**   | 10 Gio   | LVM   | ext4                | Volume dédié aux données utilisateurs. |
| **/var**    | 5 Gio    | LVM   | ext4                | Volume séparé pour les fichiers logs et autres données variables du système. |
| **swap**    | 2 Gio    | LVM   | swap                | Partition swap pour l’espace d’échange. |

### 2.2 Justification des choix

Le choix des volumes LVM permet une gestion flexible des partitions, facilitant les ajustements en cas de besoin. De plus, le partitionnement séparé pour **/home**, **/var**, et **/** permet de limiter les risques en cas de corruption de l’une de ces partitions, notamment pour les fichiers système ou les données utilisateurs. Le chiffrement des volumes sensibles, notamment pour **/home** et **/var**, est recommandé pour garantir la confidentialité des données. Ce chiffrement a été effectué à l’aide de LUKS2 pour protéger les informations en cas de vol ou de perte du serveur.

- **/boot** : Cette partition est nécessaire pour permettre le démarrage du système et ne peut pas être incluse dans un volume LVM. Elle est donc configurée indépendamment, avec un système de fichiers ext4 pour la stabilité.
- **/ (racine)** et **/var** : Ces partitions contiennent des fichiers essentiels au bon fonctionnement du système et des applications. Elles sont configurées en LVM pour une gestion plus flexible, et **/var** est séparée pour améliorer la sécurité et la performance.
- **swap** : La partition swap est configurée en LVM pour faciliter la gestion des ressources mémoire, et elle a été définie à 2 Gio, comme recommandé.

### 2.3 Respect des recommandations de l'ANSSI

Les recommandations de l'ANSSI stipulent qu'il est important de séparer les différentes partitions pour limiter les risques d'attaque. Cette configuration permet de séparer les volumes essentiels et d’assurer une meilleure gestion des permissions et de la sécurité, en particulier pour les volumes sensibles contenant des données utilisateurs et des fichiers logs. En outre, le chiffrement des volumes importants et la séparation des partitions garantissent une sécurité renforcée.

---
## 3. Sécurisation et configuration du serveur

## 3.1 Sécurisation de l’administration du serveur

### Configuration de SSH et Renforcement de la Sécurité
Afin de respecter les bonnes pratiques de sécurisation de l’administration, nous avons mis en place un serveur SSH renforcé, permettant uniquement aux utilisateurs autorisés d'accéder au serveur à l’aide d’une clé SSH sécurisée.

Voici les étapes entreprises pour sécuriser l'accès SSH :

1. **Changement du port SSH :**
   Le port par défaut de SSH (22/tcp) a été modifié en **1025/tcp** pour éviter les tentatives d'attaques automatisées sur le port standard. Cette ligne de `/etc/ssh/sshd_config` a été modifiée

   ```bash   
   Port 1025
   ``` 

2. **Configuration des clés SSH :**
   Pour renforcer la sécurité, nous avons configuré l’accès de sorte à ce qu'il soit fait uniquement via des clés SSH, ce qui empêche l'utilisation de mots de passe pour la connexion SSH. Cela élimine les risques associés aux attaques par brute force.

   Configuration dans `/etc/ssh/sshd_config` :
   ```bash
   PasswordAuthentication no
   PubkeyAuthentication yes
   ```

3. **Restriction de l’accès SSH :**
   Nous avons également mis en place deux configurations supplémentaires :

   - **Limitation des utilisateurs autorisés** : Nous avons ajouté la directive `AllowUsers` pour restreindre l'accès SSH uniquement à l'utilisateur `admin`. Cela empêche tout autre utilisateur non autorisé de se connecter via SSH.

   - **Désactivation de l'accès root** : La directive `PermitRootLogin` a été définie sur `no` pour interdire les connexions directes à l'utilisateur root. Cela empêche toute tentative de connexion en utilisant le compte root, renforçant ainsi la sécurité en obligeant l'utilisation d'un compte utilisateur spécifique pour toute connexion SSH.

   Pour ce faire, les ajouts suivants ont été effectués dans le fichier de configuration `/etc/ssh/sshd_config` :

   ```bash
   AllowUsers admin
   PermitRootLogin no
   ```

4. **Filtrage du trafic réseau via firewalld :**
   Nous avons retiré tous les services de la zone "public" et mis en place une zone **restricted** afin de n’autoriser que les connexions réseau nécessaires, notamment celles sur le port **1025** pour SSH et  nous avons réservé **7512/tcp** pour SMB qui sera installé dans la suite du TP.

   **Commandes exécutées pour la création et la configuration de zone restricted:**
   ```bash
   # Création de la zone restricted
   sudo firewall-cmd --permanent --new-zone=restricted
   # Ajout des interfaces enp0s3 et enp0s8 à la zone restricted
   sudo firewall-cmd --permanent --zone=restricted --change-zone=enp0s3
   sudo firewall-cmd --permanent --zone=restricted --change-zone=enp0s8
   # Ouverture des ports nécessaires
   sudo firewall-cmd --permanent --zone=restricted --add-port=1025/tcp
   sudo firewall-cmd --permanent --zone=restricted --add-port=7512/tcp
   # Rechargement de la configuration
   sudo firewall-cmd --reload
   ```

   **Vérification de la zone :**
   ```bash
   sudo firewall-cmd --zone=restricted --list-all
   ```
  ![image](https://github.com/user-attachments/assets/764b4690-0448-4025-9d23-3a23604ce554)

   
---

## 3.1.3 Politique de sécurité

### Configuration des mots de passe forts via PAM
Nous avons configuré **PAM** (Pluggable Authentication Modules) afin de renforcer la politique de sécurité des mots de passe. La politique impose une longueur minimale de 10 caractères, ainsi qu’une complexité incluant au moins une majuscule, une minuscule, un chiffre et un caractère spécial.

**Modification du fichier `/etc/pam.d/system-auth` pour appliquer cette politique :**

```bash
sudo vim /etc/pam.d/system-auth
# Ajout ou modification de la ligne suivante :
password requisite pam_pwquality.so retry=3 minlen=10 minclass=4
```

---

## 3.2 Configuration et Durcissement du Rôle Serveur de Fichiers

### Installation de Samba
Nous avons installé la dernière version de Samba pour gérer les partages de fichiers sur le serveur. Samba permet de créer des partages réseau compatibles avec les systèmes Windows.

1. **Installation de Samba :**
   ```bash
   sudo dnf install samba
   ```

2. **Configuration des fichiers de partage :**
   La configuration de Samba a été effectuée dans le fichier `/etc/samba/smb.conf` pour définir les partages et les autorisations d'accès.

   Exemple d’un partage défini pour **/srv/fileserver** :
   ```ini
   [fileserver]
   path = /srv/fileserver
   read only = no
   ```

---

## 3.2.2 Préparation des Comptes Utilisateurs

### Création des utilisateurs et groupes
Un script shell a été créé pour automatiser la création des comptes utilisateurs et des groupes pour Samba, conformément aux spécifications du TP.

1. **Script pour créer des utilisateurs et des groupes** :
   Le script prend en entrée un fichier structuré contenant la liste des utilisateurs et des groupes à créer. Le mot de passe pour chaque utilisateur est généré conformément à la politique de sécurité de l’entreprise.

   Exemple de script :

   ```bash
   #!/bin/bash
   create_group() {
       groupname=$1
       if ! getent group $groupname > /dev/null; then
           groupadd $groupname
           echo "Groupe $groupname créé"
       else
           echo "Le groupe $groupname existe déjà"
       fi
   }

   # Liste des utilisateurs et groupes à créer
   create_group "Service_Informatique"
   create_group "Service_Comptable"
   create_group "Pilotage"
   create_group "Direction"

   # Création des utilisateurs et assignation des groupes
   useradd -m -G Service_Informatique utilisateur1
   passwd utilisateur1
   ```

---

## 3.2.3 Structure des Répertoires
