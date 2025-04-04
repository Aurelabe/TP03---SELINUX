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

### Mise à jour
Avant de commencer quoi que ce soit, il est important de lancer une mise à jour afin d'avoir les derniers correctifs de sécurité

**Commande exécutée :**

   ```bash   
   sudo dnf update
   ``` 

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



## 3.2 Configuration et Durcissement du Rôle Serveur de Fichiers

### 3.2.1 Installation de Samba et mise en place du stockage dédié

Nous avons installés la dernière version de Samba pour gérer les partages de fichiers sur le serveur. Pour respecter les bonnes pratiques de sécurité et de maintenance, les fichiers partagés seront stockés sur une partition dédiée (`/srv/fileserver`) configurée avec **LVM** et **le système de fichiers ext4**, ce qui facilite un redimensionnement ultérieur.

#### 1. Création de la partition physique `/dev/sda6`

La partition a été créée avec `fdisk` :
```bash
sudo fdisk /dev/sda
```
Dans `fdisk`, nous avons :
- Créé une nouvelle partition (`n`) ;
- Choisi la taille de la partition ;
- Sauvegardé avec `w`.

Puis on a rechargé la table :
```bash
sudo partprobe
```

#### 2. Ajout au groupe de volumes et création du volume logique

```bash
sudo vgextend vg_data /dev/sda6
sudo lvcreate -L 9G -n fileserver vg_data
```

#### 3. Formatage et montage de la partition

```bash
sudo mkfs.ext4 /dev/vg_data/fileserver
sudo mkdir -p /srv/fileserver
sudo mount /dev/vg_data/fileserver /srv/fileserver
```

Pour rendre le montage permanent :
```bash
echo "/dev/vg_data/fileserver /srv/fileserver ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

#### Chiffrement de la partition ?

Le volume `/srv/fileserver` **n’a pas été chiffré**.

**Justification** :
- La partition ne contient **pas de données sensibles**.
- Le serveur est restreint dans une zone réseau fermée (zone `restricted` de `firewalld`).
- L’accès est contrôlé uniquement via Samba avec authentification utilisateur.
- Le chiffrement ajouterait de la complexité (notamment pour le redémarrage automatique du service) sans bénéfice significatif ici.

En cas d’évolution des exigences (stockage de données confidentielles), une solution de chiffrement comme **LUKS2** pourrait être envisagée ultérieurement.

---

#### 4. Vérification de la structure des disques

On utilise `lsblk` pour vérifier la hiérarchie des volumes :

![image](https://github.com/user-attachments/assets/933b1da3-183e-4ced-b65b-6e8f2deb4a32)


Merci pour la précision, voici la suite complète de la section **3.2.2 Installation et Configuration de Samba**, avec l'intégration du changement de port pour Samba :

---

### 3.2.2 Installation et Configuration de Samba

#### 1. **Installation de Samba**

Nous avons installé la dernière version de Samba pour permettre la gestion des partages de fichiers sur le serveur. Samba est essentiel pour le partage de fichiers entre systèmes Windows et Linux, en utilisant le protocole SMB/CIFS.

**Commande pour installer Samba :**

```bash
sudo dnf install samba
```

#### 2. **Configuration de Samba**

Nous avons ensuite configuré le fichier **`/etc/samba/smb.conf`** afin de définir le partage et les paramètres nécessaires.

**Lignes ajoutées à `smb.conf`** pour le partage du répertoire **`/srv/fileserver`** :

```ini
[fileserver]
   path = /srv/fileserver
   read only = no
   browsable = yes
   guest ok = no
   valid users = @users
```

Cette configuration définit un partage **`fileserver`** pointant sur la partition montée à **`/srv/fileserver`**, avec un accès en lecture et écriture.

#### 3. **Changement du port Samba**

Le port par défaut de Samba pour les services SMB (445/tcp) peut être modifié pour renforcer la sécurité, en évitant les tentatives d'attaque sur des ports bien connus. Nous avons donc changé le port d’écoute de Samba.

**Modification du port dans `smb.conf`** :

Nous avons ajouté la ligne suivante à la section **[global]** du fichier **`/etc/samba/smb.conf`** pour utiliser un autre port (dans notre cas, **7512**):

```ini
[global]
    smb ports = 7512
```

Cela permet à Samba d’écouter sur le port **7512** au lieu du port par défaut **445/tcp**.

#### 4. **Redémarrage et activation de Samba**

Une fois la configuration modifiée, nous avons redémarré puis activé le service Samba pour appliquer les nouvelles modifications de port et de partage.

**Commande pour redémarrer Samba** :

```bash
sudo systemctl restart smb
sudo systemctl enable smb
```

#### 5. **Vérification du service Samba**

Enfin, nous avons vérifié que le service Samba fonctionne correctement en utilisant la commande **testparm**, qui permet de tester la syntaxe de la configuration de Samba :

```bash
testparm
```

![image](https://github.com/user-attachments/assets/1309eee4-2db9-4b90-932f-290d8eeeb1cd)

Le résultat de la commande testparm montre que la configuration de Samba a été chargée correctement et que le fichier /etc/samba/smb.conf ne présente pas d'erreurs majeures. Cependant, il indique également que la crypto faible est autorisée par GnuTLS, ce qui permet des connexions moins sécurisées, comme celles utilisant NTLM, principalement pour des raisons de compatibilité.

Alors, nous avons modifier le fichier `smb.conf` en ajoutant ces lignes à **[global]** :

```ini
[global]
   server min protocol = SMB2
   smb encrypt = required
   ntlm auth = no
```

- **`server min protocol = SMB2`** : Cette directive s'assure que la version minimale de SMB utilisée est SMB2, évitant ainsi les connexions utilisant SMB1.
- **`smb encrypt = required`** : Cette option oblige l'utilisation de chiffrement pour toutes les connexions.
- **`ntlm auth = no`** : Cette directive désactive l'authentification NTLM, la rendant obsolète au profit des méthodes plus sécurisées comme Kerberos.

Une fois ces modifications effectuées, nous avons redémarré Samba avec la commande suivante pour appliquer les nouvelles règles :

```bash
sudo systemctl restart smb
```

#### 6. **Vérification du port Samba**

Pour vérifier que Samba écoute sur le port que nous avons configuré, nous utilisons la commande suivante pour observer les ports d'écoute du service Samba :

```bash
sudo ss -tnlp | grep 7512
```

![image](https://github.com/user-attachments/assets/6d329026-0fd7-4224-9df8-934190f45345)


### 3.2.3 Préparation des Comptes Utilisateurs

#### Objectif
Le but ici est de créer un script shell qui générera automatiquement des comptes et des groupes d'utilisateurs pour le serveur Samba. Ce script devra également créer des mots de passe forts, tout en respectant les bonnes pratiques de sécurité et en permettant une exécution sécurisée par les membres du service informatique.

### Détails du processus

1. **Création du fichier script pour gérer les utilisateurs** :
   
   Le script doit être capable de créer les comptes et groupes d'utilisateurs de manière sécurisée. Il prendra en entrée un fichier structuré contenant la liste des utilisateurs et des groupes à créer.

   - **Le fichier structuré** devra être un fichier CSV, où chaque ligne contient un utilisateur et son groupe correspondant.
   
   - **Les comptes d’utilisateurs** doivent être créés avec des mots de passe forts, qui répondent à la politique de sécurité (minimum 10 caractères, avec des majuscules, minuscules, chiffres et caractères spéciaux).
   
   - **Les groupes d’utilisateurs** doivent également être créés si nécessaire.

2. **Script pour la création des utilisateurs** :

   Ce script **create_users** qui sera enregistré dans **/opt/scripts/maintenance/** va créer des utilisateurs et des groupes tout en générant un mot de passe fort pour chaque utilisateur. Il s’assure également que le script est exécuté par un utilisateur ayant des privilèges élevés (root). Pour s'assurer que les noms des groupes ne contiennent pas d'espaces, nous pouvons écrire le script de manière à remplacer les espaces par des underscores (`_`).


   ```bash
   #/opt/scripts/maintenance/create_users.sh
   
   # Vérifier que le script est exécuté en tant que root
   if [[ $(id -u) -ne 0 ]]; then
       echo "Ce script doit être exécuté avec des privilèges root."
       exit 1
   fi
   
   # Vérifier que l'argument (fichier CSV) est passé et que le fichier existe
   if [[ -z "$1" ]] || [[ ! -f "$1" ]]; then
       echo "Veuillez fournir un fichier CSV valide."
       exit 1
   fi
   
   # Vérifier que le fichier est bien au format CSV
   if [[ ! "$1" =~ \.csv$ ]]; then
       echo "Le fichier doit être au format CSV."
       exit 1
   fi
   
   # Fonction pour créer un groupe
   create_group() {
       # Remplacer les espaces par des tirets (ou underscores si tu préfères)
       group_name=$(echo "$1" | tr ' ' '_')
       
       if ! getent group "$group_name" > /dev/null; then
           groupadd "$group_name"
           echo "Groupe $group_name créé."
       else
           echo "Le groupe $group_name existe déjà."
       fi
   }
   
   # Fonction pour créer un utilisateur avec mot de passe fort
   create_user() {
       if ! id "$1" &>/dev/null; then
           # Créer l'utilisateur sans le groupe
           useradd -m -s /bin/bash "$1" --badname
           PASSWORD=$(openssl rand -base64 12)  # Générer un mot de passe fort
           echo "$1:$PASSWORD" | chpasswd
           echo "Utilisateur $1 créé avec mot de passe : $PASSWORD"
           
           # Ajouter l'utilisateur au groupe
           usermod -aG "$2" "$1"
           echo "Utilisateur $1 ajouté au groupe $2"
       else
           echo "L'utilisateur $1 existe déjà."
       fi
   }
   
   # Lecture du fichier CSV et création des utilisateurs et groupes
   while IFS=, read -r username group; do
       # Ignorer les lignes vides et les commentaires
       if [[ -z "$username" || "$username" =~ ^# ]]; then
           continue
       fi
   
       create_group "$group"
       create_user "$username" "$group"
   done < "$1"
   ```

3. **Utilisation du script** :
   
   Le script peut être exécuté de la manière suivante :

   - Le fichier d'entrée doit être un fichier CSV (dans notre cas **users.csv**).
   
   - Contenu du fichier **/opt/scripts/maintenance/users.csv** pour notre cas :
     ```csv
      Edouard-Franck Wagner,Direction
      Édouard-Franck Wagner,Pilotage
      Alexandra Met,Pilotage
      Claude Muller,Pilotage
      Gabriel Benard,Pilotage
      Alexandra Met,Service_Comptable
      Tristan Maurice,Service_Comptable
      Alfred Laroche,Service_Comptable
      Claude Muller,Service_Informatique
      Henriette Julien,Service_Informatique
      Jeanne Pasquier,Service_Informatique
      Monique Pires,Service_Informatique
      Gabriel Benard,Service_Logistique
      Cerise Lapresse,Service_Logistique
      Richard Pirouet,Service_Logistique
      Jacques Bonnet,Service_Logistique
      Louis Lamoureux,Service_Logistique
      Honore Adler,Service_Logistique
     ```
   
   - Commande pour exécuter le script avec un fichier CSV :
     ```bash
     sudo /opt/scripts/maintenance/create_users.sh /opt/scripts/maintenance/users.csv
     ```
![Capture d'écran 2025-04-04 093826](https://github.com/user-attachments/assets/e6de6a7c-17c9-4d62-8758-51c0d6260817)


4. **Sécurisation du script** :

   Le script doit être sécurisé, de sorte qu’il ne puisse être exécuté que par des utilisateurs ayant des privilèges élevés (root). De plus, seuls les membres du **Service Informatique** auront le droit d'exécuter ce script. On peut donc restreindre l'accès à ce fichier en modifiant ses permissions :

```bash
sudo chown root:Service_Informatique /opt/scripts/maintenance/create_users.sh
sudo chmod 750 /opt/scripts/maintenance/create_users.sh
```

Les membres du groupe **Service Informatique** peuvent exécuter ce script avec `sudo`, mais aucun autre utilisateur ne pourra l'exécuter.


5. **Exécution du script avec privilèges élevés** :

   - Nous devons nous assurer que le script peut être exécuté avec des privilèges root par les utilisateurs du service informatique sans avoir besoin d'entrer le mot de passe root. Pour cela, nous devons ajouter une règle dans le fichier `sudoers` :

   ```bash
   sudo visudo
   ```

   ```
   %Service_Informatique ALL=(ALL) NOPASSWD: /opt/scripts/maintenance/create_users.sh
   ```



### 3.3 Préparation de la structure des répertoires

   Nous créerons la structure des répertoires partagés sur `/srv/fileserver`, selon les besoins de l'entreprise. Nous allons donc créer les dossiers nécessaires et y appliquer les permissions via les ACLs POSIX, tout en garantissant que **seul le propriétaire d'un fichier peut le supprimer**.

Voici la structure des répertoires telle que définie dans l'annexe :

```
/srv/fileserver
    ├── D1 - Direction
    │   ├── D1-1 – Stratégie
    │   ├── D1-2 – Achats
    ├── C1 - Comptabilité
    │   ├── C1-1 - Comptabilité
    │   ├── C1-2 - TVA
    │   ├── C1-3 - Facturation
    ├── A1 - Administratif
    │   ├── A1-1 - Gestion des congés
    │   ├── A1-2 - Gestion des formations internes
    │   ├── A1-3 - Qualité
    │   ├── A1-4 - Achats
    ├── D2 - DSI
    │   ├── D2-1 - Procédures
    │   ├── D2-2 - Divers scripts
    │   ├── D2-3 - Documentation du SI
    ├── D3 - Dépôt
    ├── C2 - Commun
        ├── C2-1 - Direction
        ├── C2-2 - Administratif
        ├── C2-3 - Informatique
        ├── C2-4 - Production
```


#### 1. Création des répertoires
Voici les commandes exécutées pour créer cette structure de répertoires sans les lettres (D1, C1, etc.) :

```bash
sudo mkdir -p /srv/fileserver/Direction/{Strategie,Achats} \
             /srv/fileserver/Comptabilite/{Comptabilite,TVA,Facturation} \
             /srv/fileserver/Administratif/{Gestion_des_conges,Gestion_des_formations_internes,Qualite,Achats} \
             /srv/fileserver/DSI/{Procedures,Divers_scripts,Documentation_du_SI} \
             /srv/fileserver/Commun/{Direction,Administratif,Informatique,Production} \
             /srv/fileserver/Depot
```

#### 2. Application des ACLs POSIX
   Maintenant que la structure des répertoires est en place, nous appliquons les ACLs POSIX pour gérer les permissions de manière fine. Nous allons garantir que chaque groupe associé à un répertoire donné aura les bonnes permissions d'accès.

##### Pour le répertoire `/srv/fileserver/Direction` :
```bash
sudo setfacl -m g:Direction:rwx /srv/fileserver/Direction
sudo setfacl -d -m g:Direction:rwx /srv/fileserver/Direction

sudo setfacl -m g:Direction:rwx /srv/fileserver/Direction/Strategie
sudo setfacl -d -m g:Direction:rwx /srv/fileserver/Direction/Strategie

sudo setfacl -m g:Direction:rwx /srv/fileserver/Direction/Achats
sudo setfacl -d -m g:Direction:rwx /srv/fileserver/Direction/Achats
```

Cette méthode sera répétée pour chaque répertoire spécifique, en appliquant les permissions en fonction des groupes comme indiqué dans l'énoncé du TP.

##### Pour `/srv/fileserver/Comptabilite` :
```bash
sudo setfacl -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite
sudo setfacl -d -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite

sudo setfacl -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/Comptabilite
sudo setfacl -d -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/Comptabilite

sudo setfacl -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/TVA
sudo setfacl -d -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/TVA

sudo setfacl -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/Facturation
sudo setfacl -d -m g:Service_Comptable:rwx /srv/fileserver/Comptabilite/Facturation
```

##### Pour `/srv/fileserver/Administratif` :
```bash
sudo setfacl -m g:Service_Logistique:rwx /srv/fileserver/Administratif
sudo setfacl -d -m g:Service_Logistique:rwx /srv/fileserver/Administratif

sudo setfacl -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Gestion_des_conges
sudo setfacl -d -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Gestion_des_conges

sudo setfacl -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Gestion_des_formations_internes
sudo setfacl -d -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Gestion_des_formations_internes

sudo setfacl -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Qualite
sudo setfacl -d -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Qualite

sudo setfacl -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Achats
sudo setfacl -d -m g:Service_Logistique:rwx /srv/fileserver/Administratif/Achats

```

##### Pour `/srv/fileserver/DSI` :
```bash
sudo setfacl -m g:Service_Informatique:rwx /srv/fileserver/DSI
sudo setfacl -d -m g:Service_Informatique:rwx /srv/fileserver/DSI

sudo setfacl -m g:Service_Informatique:rwx /srv/fileserver/DSI/Procedures
sudo setfacl -d -m g:Service_Informatique:rwx /srv/fileserver/DSI/Procedures

sudo setfacl -m g:Service_Informatique:rwx /srv/fileserver/DSI/Divers_scripts
sudo setfacl -d -m g:Service_Informatique:rwx /srv/fileserver/DSI/Divers_scripts

sudo setfacl -m g:Service_Informatique:rwx /srv/fileserver/DSI/Documentation_du_SI
sudo setfacl -d -m g:Service_Informatique:rwx /srv/fileserver/DSI/Documentation_du_SI

```

##### Pour `/srv/fileserver/Commun` :
```bash
sudo setfacl -m g:Pilotage:rwx /srv/fileserver/Commun
sudo setfacl -d -m g:Pilotage:rwx /srv/fileserver/Commun

sudo setfacl -m g:Pilotage:rwx /srv/fileserver/Commun/Direction
sudo setfacl -d -m g:Pilotage:rwx /srv/fileserver/Commun/Direction

sudo setfacl -m g:Pilotage:rwx /srv/fileserver/Commun/Administratif
sudo setfacl -d -m g:Pilotage:rwx /srv/fileserver/Commun/Administratif

sudo setfacl -m g:Pilotage:rwx /srv/fileserver/Commun/Informatique
sudo setfacl -d -m g:Pilotage:rwx /srv/fileserver/Commun/Informatique

sudo setfacl -m g:Pilotage:rwx /srv/fileserver/Commun/Production
sudo setfacl -d -m g:Pilotage:rwx /srv/fileserver/Commun/Production

```

##### Pour `/srv/fileserver/Depôt` :
```bash
sudo setfacl -m g:Direction:rwx /srv/fileserver/Depot
sudo setfacl -d -m g:Direction:rwx /srv/fileserver/Depot
```

#### 3. Vérification des ACLs
Pour vérifier les ACLs appliquées sur les répertoires et s’assurer qu’elles respectent les règles définies, nous avons utilisés la commande suivante :

```bash
sudo getfacl -R /srv/fileserver
```
Voici un petit aperçu de la sortie :

![image](https://github.com/user-attachments/assets/a3ae6c2c-c061-468a-a382-e4f1660ff06a)


### 3.4 Sauvegarde du serveur

   Nous allons maintenant créer un script shell pour effectuer une sauvegarde de la structure de répertoires sur `/srv/fileserver`, avec un chiffrement AES-256-GCM et stockage de l'archive sur un espace en ligne.

#### 1. Création du script

Le script devra être placé dans `/opt/scripts/maintenance/`, et devra effectuer les actions suivantes :

- **Créer une archive tar.gz chiffrée** du répertoire `/srv/fileserver`.
- **Générer une clé de chiffrement AES-256-GCM** lors de la sauvegarde.
- **Envoyer la clé de chiffrement par email à l'administrateur.**
- **Déposer l'archive sur un espace en ligne dédié.**


```bash
#/opt/scripts/maintenance/backup.sh

# Variables
backup_dir="/srv/fileserver"
backup_file="/opt/scripts/maintenance/backup-$(date +%F).tar.gz"
encryption_key=$(openssl rand -base64 32)   # Générer une clé de chiffrement aléatoire
encryption_key_file="/opt/scripts/maintenance/encryption_key-$(date +%F).key"  # Fichier de la clé
email_admin="admin@company.com"  # Email de l'administrateur
remote_server="user@online-space"  # Serveur distant pour la sauvegarde
remote_dir="/backup/"  # Répertoire sur le serveur distant pour stocker l'archive

# Création de l'archive tar.gz
echo "Création de l'archive tar.gz..."
tar -czf $backup_file $backup_dir
if [ $? -ne 0 ]; then
    echo "Erreur lors de la création de l'archive. Script arrêté."
    exit 1
fi

# Chiffrement de l'archive avec AES-256-GCM
echo "Chiffrement de l'archive..."
openssl enc -aes-256-gcm -salt -in $backup_file -out ${backup_file}.enc -k $encryption_key
if [ $? -ne 0 ]; then
    echo "Erreur lors du chiffrement de l'archive. Script arrêté."
    exit 1
fi

# Sauvegarder la clé de chiffrement dans un fichier sécurisé
echo $encryption_key > $encryption_key_file
chmod 600 $encryption_key_file  # Sécuriser le fichier clé

# Envoyer la clé de chiffrement par email
echo "La clé de chiffrement est : $encryption_key" | mail -s "Clé de chiffrement de la sauvegarde" $email_admin

# Télécharger l'archive chiffrée sur l'espace en ligne
echo "Déploiement de l'archive sur l'espace en ligne..."
scp ${backup_file}.enc $remote_server:$remote_dir
if [ $? -ne 0 ]; then
    echo "Erreur lors du transfert de l'archive. Script arrêté."
    exit 1
fi

# Suppression de l'archive non chiffrée pour des raisons de sécurité
rm $backup_file

echo "Sauvegarde effectuée avec succès."
```

#### 2. Permissions du script

Le script doit être exécuté par le compte `SauvegardeServeur`. Il doit être stocké dans `/opt/scripts/maintenance/` avec les bonnes permissions. Voici les commandes pour attribuer les permissions :

1. **Création du compte et du groupe `SauvegardeServeur`** :

    ```bash
     sudo useradd -m -s /bin/bash SauvegardeServeur
     sudo groupadd SauvegardeServeur
     sudo usermod -a -G SauvegardeServeur SauvegardeServeur
   ```

3. **Attribution des droits d'accès** au script :

    ```bash
      sudo chown root:SauvegardeServeur /opt/scripts/maintenance/backup.sh
      sudo chmod 750 /opt/scripts/maintenance/backup.sh
   ```

5. **Sudoers pour l'exécution sans mot de passe** :

    La ligne suivante a été ajoutée au fichier `/etc/sudoers` pour permettre à `Service_Informatique` d'exécuter certaines commandes avec `sudo` sans mot de passe :
   
   ```bash
   Service_Informatique ALL=(ALL) NOPASSWD: /usr/bin/openssl, /bin/tar, /usr/bin/mail, /usr/bin/scp
   ```

Cela garantit que seul `SauvegardeServeur` et les membres du groupe `Service_Informatique` peuvent exécuter le script.



### 3.5 Revue des accès et contrôle de conformité

### 1. Objectifs du script

- Vérifier que les fichiers et répertoires ont les bonnes permissions (en fonction de la matrice de droits de référence).
- Désactiver les comptes qui ne correspondent pas à la matrice de référence.
- Créer un rapport détaillé des écarts.

### 2. Structure du script


```bash
# /opt/scripts/maintenance/controle_acces.sh

# Paramètres
reference_file="$1"
report_file="/opt/scripts/maintenance/rapport_acces_$(date +%F).txt"
date=$(date "+%Y-%m-%d %H:%M:%S")
email_admin="admin@company.com"

# Vérification si le fichier de référence existe
if [ ! -f "$reference_file" ]; then
    echo "Le fichier de référence $reference_file est introuvable." >> $report_file
    exit 1
fi

# En-tête du rapport
echo "Rapport de revue des accès - $date" > $report_file
echo "--------------------------------------------------" >> $report_file

# Fonction de comparaison des privilèges
check_permissions() {
    local file_path="$1"
    local expected_permissions="$2"
    actual_permissions=$(stat -c %a "$file_path")
    
    if [ "$actual_permissions" != "$expected_permissions" ]; then
        echo "Erreur sur les permissions de $file_path : attendu $expected_permissions, trouvé $actual_permissions." >> $report_file
    fi
}

# Fonction pour vérifier les utilisateurs
check_user() {
    local user="$1"
    user_exists=$(id -u "$user" &>/dev/null && echo "yes" || echo "no")

    if [ "$user_exists" == "no" ]; then
        echo "L'utilisateur $user est absent sur le système, il sera désactivé." >> $report_file
        sudo usermod -L "$user" # Désactivation de l'utilisateur
    fi
}

# Lecture de la matrice de référence et vérification
while IFS=, read -r file_path expected_permissions
do
    # Vérifier les permissions sur chaque fichier/dossier
    check_permissions "$file_path" "$expected_permissions"
done < "$reference_file"

# Vérification des utilisateurs et groupes
while IFS=, read -r user
do
    # Vérifier si l'utilisateur existe et, sinon, le désactiver
    check_user "$user"
done < "$reference_file"

# Finaliser le rapport
echo "--------------------------------------------------" >> $report_file
echo "Revue terminée - $date" >> $report_file

# Envoyer un email à l'administrateur avec le rapport
mail -s "Rapport de revue des accès" $email_admin < $report_file

echo "Revue des accès terminée. Le rapport a été envoyé à $email_admin."
```

### 3. Explications des parties principales du script

- **Lecture du fichier de référence** : Le fichier de référence contient la liste des chemins des fichiers/dossiers et les permissions attendues, ainsi que les utilisateurs à vérifier.
- **Vérification des permissions** : Pour chaque fichier/dossier, le script compare les permissions actuelles avec celles spécifiées dans la matrice de référence. Si elles ne correspondent pas, une erreur est enregistrée dans le rapport.
- **Vérification des utilisateurs** : Le script s'assure que les utilisateurs mentionnés dans le fichier de référence existent. Si un utilisateur n'existe pas, il sera désactivé avec `usermod -L`.
- **Rapport et envoi par email** : Le script génère un rapport détaillant les erreurs et les utilisateurs désactivés, puis envoie ce rapport par email à l'administrateur.

### 4. Permissions du script

Comme pour le script précédent, il est important que le script ait les bonnes permissions et soit accessible uniquement par le groupe `Service_Informatique`.

#### 1. **Attribution des accès au script dans `/opt/scripts/maintenance/`** :

```bash
sudo chown root:Service_Informatique /opt/scripts/maintenance/controle_acces.sh
sudo chmod 750 /opt/scripts/maintenance/controle_acces.sh
```

#### 2. **Configuration des privilèges `sudo`** :

Pour permettre aux membres de `Service_Informatique` d'exécuter certaines commandes avec `sudo` sans mot de passe, onajoute cette ligne dans le fichier sudoers via `visudo` :

```bash
Service_Informatique ALL=(ALL) NOPASSWD: /usr/bin/stat, /usr/sbin/usermod, /usr/bin/mail
```
