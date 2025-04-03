# Rapport de Partitionnement et Durcissement du Système

## 1. **Installation du système d’exploitation**

### 1.1 **Choix du système d’exploitation**

Le système d'exploitation choisi pour cette installation est **Rocky Linux 9**, qui est une distribution stable et sécurisée, idéale pour un serveur. La version minimale a été installée, conformément aux consignes. Cette version ne comporte pas d'interface graphique afin de réduire la surface d'attaque et de minimiser les ressources système allouées.

### 1.2 **Installation sans interface graphique**

La décision de ne pas installer d'interface graphique est conforme aux recommandations de l'ANSSI. Cette mesure permet de :
- Réduire la surface d'attaque en éliminant les services graphiques inutiles qui peuvent être exploités par des attaquants.
- Limiter la consommation de ressources système, assurant une meilleure performance pour les services essentiels du serveur.
- Favoriser l'administration à distance via SSH, ce qui est une méthode plus sécurisée et efficace pour gérer un serveur en production.

### 1.3 **Installation de paquets utiles uniquement**

Seuls les paquets nécessaires à l'exécution des services ciblés ont été installés. Aucun paquet superflu n’a été ajouté, conformément aux bonnes pratiques de sécurité :
- **Moindre privilège** : limiter les paquets installés réduit la surface d'attaque.
- **Contrôle des dépendances** : en installant uniquement ce qui est nécessaire, nous assurons que les dépendances inutiles n’introduisent pas de failles potentielles.

### 1.4 **Mise à jour des correctifs de sécurité**

Avant de commencer le partitionnement et la configuration du serveur, tous les **derniers correctifs de sécurité** ont été appliqués pour garantir que le système est à jour et protégé contre les vulnérabilités connues. Cette étape est cruciale pour s’assurer que le système est protégé dès sa mise en service.

---

## 2. **Schéma de Partitionnement**

### 2.1 **Contexte et recommandations de l'ANSSI**

L'ANSSI recommande de configurer un partitionnement sécurisé avec des partitions dédiées pour chaque système important (par exemple, `/boot`, `/home`, `/var`, etc.). Ce partitionnement permet de mieux sécuriser les données en cas d'attaque, notamment en isolant les systèmes critiques et en facilitant la gestion des sauvegardes et des contrôles d’accès.

### 2.2 **Schéma de Partitionnement adopté**

Le schéma de partitionnement a été configuré comme suit :

| **Partition**   | **Taille** | **Type**   | **Système de fichiers** | **Justification**                                                                                                                                     |
|-----------------|------------|------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/boot`         | 1 Gio      | Partition standard | ext4                 | **Non LVM** : Il est recommandé par l'ANSSI de ne pas inclure la partition `/boot` dans un groupe de volumes LVM. Cela garantit que le chargeur de démarrage et les fichiers essentiels au démarrage du système sont indépendants du gestionnaire LVM. Cela permet également de faciliter la récupération du système en cas de problème. |
| `/` (root)      | 15 Gio     | LVM        | xfs                     | **LVM** : La partition racine est installée sous LVM pour permettre une gestion flexible de l’espace disque à l’avenir. Le système de fichiers XFS est utilisé pour sa robustesse et ses performances sur de grandes volumes de données. |
| `/home`         | 10 Gio     | LVM        | ext4                    | **LVM** : Séparer la partition `/home` permet d’isoler les données utilisateurs des autres systèmes, ce qui augmente la sécurité. Le système de fichiers ext4 est choisi pour sa stabilité et sa compatibilité avec LVM. |
| `/var`          | 5 Gio      | LVM        | ext4                    | **LVM** : La partition `/var` est dédiée aux données variables (logs, caches, etc.). La gestion séparée de cette partition permet de protéger le système contre les effets des attaques ou des erreurs liées aux logs, tout en facilitant la gestion des sauvegardes. |
| `swap`          | 2 Gio      | LVM        | swap                    | **LVM** : L'utilisation de LVM pour la partition swap permet une plus grande flexibilité dans la gestion de la mémoire virtuelle. La taille de 2 Gio est suffisante pour la gestion des besoins mémoire du serveur. |

### 2.3 **Justification des choix**

1. **Séparation des partitions** :
   - En séparant les partitions comme `/home`, `/var`, et `/`, on limite les risques en cas de corruption ou d'attaque sur l'une des partitions. Cela permet également d'améliorer la gestion des ressources et de faciliter les sauvegardes et restaurations spécifiques aux données critiques.
   
2. **Utilisation de LVM** :
   - LVM (Logical Volume Manager) a été utilisé pour les partitions `/`, `/home`, `/var`, et `swap` pour permettre une gestion dynamique de l’espace disque. LVM permet d'agrandir ou de réduire facilement les partitions en fonction des besoins, ce qui est particulièrement utile pour les serveurs en production.
   
3. **Choix des systèmes de fichiers** :
   - **ext4** a été choisi pour les partitions `/home`, `/var`, et `swap` en raison de sa stabilité et de ses bonnes performances pour les applications de serveur.
   - **xfs** a été choisi pour la partition `/` pour sa robustesse et ses performances sur les grandes tailles de fichiers, ce qui est adapté à un système de serveur.
   
---

## 3. **Protection du chargeur de démarrage**

### 3.1 **Mot de passe pour le chargeur de démarrage**

L'ANSSI recommande de protéger le chargeur de démarrage (GRUB) par un mot de passe pour éviter que des utilisateurs non autorisés ne modifient les options de démarrage, telles que les paramètres du noyau ou les entrées de démarrage en mode de récupération.

#### 3.2 **Est-ce adapté à ce cas ?**

Dans le contexte de ce serveur, **la protection du chargeur de démarrage par un mot de passe est pertinente et recommandée**. Voici pourquoi :
- **Protection contre les attaques physiques** : En cas d'accès physique non autorisé au serveur, cette protection empêche un attaquant de manipuler les options de démarrage pour exploiter des vulnérabilités ou obtenir un accès non autorisé au système.
- **Sécurisation du processus de démarrage** : Le mot de passe GRUB protège l'intégrité du processus de démarrage, assurant que seul un administrateur autorisé peut modifier la configuration du noyau ou démarrer en mode de récupération.
  
Cependant, dans un environnement de serveur où l'accès physique est restreint (par exemple, dans un datacenter sécurisé), cette mesure peut ne pas être aussi cruciale. Néanmoins, par principe de précaution et pour suivre les recommandations de l'ANSSI, il est conseillé de l'implémenter.

---

## Conclusion

Le partitionnement a été réalisé conformément aux recommandations de l'ANSSI en tenant compte des besoins spécifiques du serveur. Le système a été installé en version minimale, sans interface graphique, et seuls les paquets nécessaires ont été installés pour garantir la sécurité et la performance. La protection du chargeur de démarrage par un mot de passe est également une mesure pertinente pour sécuriser l'accès au serveur en cas de tentative d'attaque physique.

---

Cela répond-il à vos attentes ?
