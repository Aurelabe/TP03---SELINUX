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
