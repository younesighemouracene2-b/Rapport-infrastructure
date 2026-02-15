<document> 
<content> 
<p style="font-size:11pt; font-family:Calibri;"> 
<strong>RAPPORT TECHNIQUE : ARCHITECTURE DE PARTAGE DE FICHIERS SÉCURISÉE SUR GOOGLE CLOUD PLATFORM (GCP)</strong><br/>
<em>Younes</em><br/> 
</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>1. Introduction</strong></p> 
<p style="font-size:11pt;">Ce rapport présente la mise en place d'une infrastructure Cloud complète permettant l'upload, le stockage et le téléchargement de fichiers avec gestion d'expiration. Le projet repose sur une architecture multi-services utilisant Google Compute Engine (VM), Cloud SQL (Base de données managée) et Cloud Storage (Stockage d'objets).</p> 
<p style="font-size:11pt;">Ce rapport détaille l'ensemble des étapes réalisées, les commandes utilisées, les difficultés rencontrées et leurs résolutions. Il constitue une documentation complète du projet.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>2. Architecture Globale</strong></p> 
<p style="font-size:11pt;">L'architecture mise en place est illustrée dans le schéma ci-dessous. Tous les composants sont interconnectés de manière sécurisée : la VM accède à la base de données via son IP privée et au bucket via un compte de service avec les droits IAM appropriés.</p> 
<p style="font-size:11pt;"><em>Figure 1 – Schéma de l'infrastructure</em> (insérer ici une image du schéma)</p> 
<p style="font-size:11pt;">Le tableau suivant récapitule les services utilisés et leurs configurations principales :</p> 

<table style="border-collapse: collapse; width: 100%; font-size:11pt;" border="1"> 
<tr> 
<th style="border: 1px solid black; padding: 5px;">Composant</th> 
<th style="border: 1px solid black; padding: 5px;">Service GCP</th> 
<th style="border: 1px solid black; padding: 5px;">Rôle</th> 
<th style="border: 1px solid black; padding: 5px;">Configuration principale</th> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Machine virtuelle</td> 
<td style="border: 1px solid black; padding: 5px;">Compute Engine</td> 
<td style="border: 1px solid black; padding: 5px;">Héberge l'application web (Apache/PHP)</td> 
<td style="border: 1px solid black; padding: 5px;">Debian 12, europe-west9-a, IP publique 34.155.29.44, IP privée 10.0.0.2</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Base de données</td> 
<td style="border: 1px solid black; padding: 5px;">Cloud SQL (MySQL)</td> 
<td style="border: 1px solid black; padding: 5px;">Stocke les métadonnées des fichiers</td> 
<td style="border: 1px solid black; padding: 5px;">Instance MySQL 8.4, IP publique 34.155.58.4, IP privée 172.29.48.3, SSL uniquement</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Stockage objet</td> 
<td style="border: 1px solid black; padding: 5px;">Cloud Storage</td> 
<td style="border: 1px solid black; padding: 5px;">Stocke les fichiers uploadés</td> 
<td style="border: 1px solid black; padding: 5px;">Bucket <code>partage-fichiers-younes-2026</code>, région europe-west9, privé, suppression réversible activée</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Réseau</td> 
<td style="border: 1px solid black; padding: 5px;">VPC</td> 
<td style="border: 1px solid black; padding: 5px;">Isole les ressources</td> 
<td style="border: 1px solid black; padding: 5px;">Réseau par défaut avec règles de pare-feu (ports 80, 443, 3306)</td> 
</tr> 
</table> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>3. Prérequis et Configuration Initiale</strong></p> 
<p style="font-size:11pt;">La première étape a consisté à initialiser le projet sur la console GCP sous l'identifiant <code>My First Project</code>. Avant de créer les machines, il a fallu autoriser les flux de trafic :</p> 
<ul style="font-size:11pt;"> 
<li>Port 80 (HTTP) : Pour l'accès web au formulaire.</li> 
<li>Port 443 (HTTPS) : Pour les connexions sécurisées.</li> 
<li>Port 3306 (MySQL) : Pour la communication entre la VM et la base de données.</li> 
</ul> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>4. Création et Configuration de la Machine Virtuelle</strong></p> 
<p style="font-size:11pt;"><strong>4.1 Création de la VM</strong></p> 
<p style="font-size:11pt;">L'instance nommée <code>vm-web-serveur</code> a été déployée avec les caractéristiques suivantes :</p> 
<ul style="font-size:11pt;"> 
<li>Zone : europe-west9-a (Paris)</li> 
<li>OS : Debian 12</li> 
<li>IP Externe : 34.155.29.44</li> 
<li>IP Interne : 10.0.0.2</li> 
</ul> 

<p style="font-size:11pt;"><strong>4.2 Installation des Services</strong></p> 
<p style="font-size:11pt;">Une fois connecté en SSH, les commandes suivantes ont été exécutées pour préparer le serveur :</p> 
<pre style="background-color: #f0f0f0; padding: 5px; font-family: Consolas; font-size:10pt;"> 
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 php php-mysqli libapache2-mod-php composer -y
sudo systemctl status apache2
</pre> 

<p style="font-size:11pt;"><strong>5. Création et Configuration de la Base de Données Cloud SQL</strong></p> 
<p style="font-size:11pt;"><strong>5.1 Création de l'instance MySQL</strong></p> 
<p style="font-size:11pt;">Une instance MySQL 8.4 nommée <code>mysql-projet</code> a été créée :</p> 
<ul style="font-size:11pt;"> 
<li>IP Publique : 34.155.58.4</li> 
<li>IP Privée : 172.29.48.3</li> 
</ul> 

<p style="font-size:11pt;"><strong>5.2 Sécurisation et Connexion</strong></p> 
<p style="font-size:11pt;">L'accès a été restreint pour n'autoriser que les connexions sécurisées :</p> 
<ul style="font-size:11pt;"> 
<li>Forçage SSL : Activation de l'option "Autoriser uniquement les connexions SSL"</li> 
<li>Réseaux autorisés : Ajout de l'IP de la VM (34.155.29.44/32) dans les paramètres de connexion de Cloud SQL</li> 
</ul> 

<p style="font-size:11pt;"><strong>5.3 Structure de la base de données</strong></p> 
<p style="font-size:11pt;">Connexion via Cloud SQL Studio pour créer la table :</p> 
<pre style="background-color: #f0f0f0; padding: 5px; font-family: Consolas; font-size:10pt;"> 
CREATE DATABASE db_partage;
USE db_partage;
CREATE TABLE fichiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom_fichier VARCHAR(255),
    hash VARCHAR(64),
    type_mime VARCHAR(100),
    date_expiration DATETIME
);
</pre> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>6. Création et Configuration du Bucket Cloud Storage</strong></p> 
<p style="font-size:11pt;"><strong>6.1 Création du bucket</strong></p> 
<p style="font-size:11pt;">Un bucket nommé <code>partage-fichiers-younes-2026</code> a été créé dans la région europe-west9.</p> 

<p style="font-size:11pt;"><strong>6.2 Installation du SDK Google Cloud</strong></p> 
<p style="font-size:11pt;">Pour que PHP puisse interagir avec le stockage, l'installation de la bibliothèque cliente était nécessaire :</p> 
<pre style="background-color: #f0f0f0; padding: 5px; font-family: Consolas; font-size:10pt;"> 
cd /var/www/html
composer require google/cloud-storage
</pre> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>7. Développement de l'Application PHP</strong></p> 
<p style="font-size:11pt;"><strong>7.1 Page d'accueil (index.php)</strong></p> 
<p style="font-size:11pt;">Cette page affiche le formulaire et la liste des fichiers actifs en interrogeant la base SQL avec une clause WHERE date_expiration > NOW().</p> 

<p style="font-size:11pt;"><strong>7.2 Logique d'upload (upload.php)</strong></p> 
<p style="font-size:11pt;">Le script génère un hash unique pour chaque fichier afin d'éviter les doublons dans le bucket et calcule la date d'expiration.</p> 

<p style="font-size:11pt;"><strong>7.3 Gestion des téléchargements (download.php)</strong></p> 
<p style="font-size:11pt;">Ce script récupère le hash via l'URL, vérifie l'existence en base SQL, puis appelle l'API Google Cloud Storage pour servir le fichier au navigateur.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>8. Problèmes Rencontrés et Résolutions</strong></p> 

<p style="font-size:11pt;"><strong>8.1 Erreur 403 Forbidden (Storage)</strong></p> 
<p style="font-size:11pt;">Problème : Le compte de service <code>881081160154-compute@developer.gserviceaccount.com</code> n'avait pas les droits d'écriture sur le bucket.</p> 
<p style="font-size:11pt;">Résolution : Attribution du rôle Administrateur des objets Storage via la console IAM et passage de la VM en "Accès complet à toutes les API Cloud" après arrêt de l'instance.</p> 

<p style="font-size:11pt;"><strong>8.2 Erreur Access Denied SQL</strong></p> 
<p style="font-size:11pt;">Problème : Échec de la connexion à 10.0.0.2 car le SSL était exigé par Cloud SQL mais pas configuré dans PHP.</p> 
<p style="font-size:11pt;">Résolution : Utilisation de mysqli_ssl_set() et real_connect() avec le flag MYSQLI_CLIENT_SSL.</p> 

<p style="font-size:11pt;"><strong>8.3 Bug de la date d'expiration</strong></p> 
<p style="font-size:11pt;">Problème : L'heure d'expiration affichée était identique à l'heure d'upload.</p> 
<p style="font-size:11pt;">Résolution : Configuration du timezone <code>date_default_timezone_set('Europe/Paris')</code> et calcul dynamique <code>$date_exp = date('Y-m-d H:i:s', strtotime("+$secondes seconds"))</code>.</p> 

<p style="font-size:11pt;"><strong>8.4 Suppression accidentelle et Restauration</strong></p> 
<p style="font-size:11pt;">Problème : Les fichiers ont été supprimés du bucket par erreur, entraînant une erreur 500 au téléchargement.</p> 
<p style="font-size:11pt;">Résolution : Activation du filtre "Objets supprimés de façon réversible uniquement" dans la console Storage, puis clic sur le bouton Restaurer. La rétention est active jusqu'au 22 février 2026.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>9. Gestion Avancée des Identités et des Accès (IAM)</strong></p> 
<p style="font-size:11pt;"><strong>9.1 Concept du Moindre Privilège</strong></p> 
<p style="font-size:11pt;">Dans le cadre de ce projet, la sécurité ne repose pas uniquement sur des mots de passe, mais sur des rôles IAM. L'objectif a été d'appliquer le principe du "moindre privilège", consistant à ne donner à la machine virtuelle que les droits strictement nécessaires.</p> 

<p style="font-size:11pt;"><strong>9.2 Configuration du Compte de Service</strong></p> 
<p style="font-size:11pt;">L'instance de calcul utilise le compte de service <code>881081160154-compute@developer.gserviceaccount.com</code>. Le rôle Administrateur des objets Storage a été attribué pour permettre les opérations sur le bucket.</p> 

<p style="font-size:11pt;"><strong>9.3 Les Scopes d'Accès de l'API Cloud</strong></p> 
<p style="font-size:11pt;">Après arrêt de l'instance, la section Accès à l'API Cloud a été modifiée pour passer sur "Autoriser l'accès complet à toutes les API Cloud".</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>10. Architecture Réseau et Connectivité</strong></p> 
<p style="font-size:11pt;"><strong>10.1 Isolation par IP Privée</strong></p> 
<p style="font-size:11pt;">Cloud SQL a été configuré avec une IP privée (172.29.48.3) pour forcer le trafic à rester à l'intérieur du réseau VPC, réduisant la latence et augmentant la sécurité.</p> 

<p style="font-size:11pt;"><strong>10.2 Configuration des Réseaux Autorisés</strong></p> 
<p style="font-size:11pt;">L'adresse IP externe de la VM (34.155.29.44/32) a été ajoutée dans la liste des Réseaux autorisés de Cloud SQL pour permettre les tests.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>11. Sécurisation des Flux de Données (SSL/TLS)</strong></p> 
<p style="font-size:11pt;"><strong>11.1 Forçage du chiffrement</strong></p> 
<p style="font-size:11pt;">L'option "Autoriser uniquement les connexions SSL" a été activée sur l'instance mysql-projet, garantissant que toutes les communications sont chiffrées.</p> 

<p style="font-size:11pt;"><strong>11.2 Adaptation du code PHP au SSL</strong></p> 
<p style="font-size:11pt;">La fonction de connexion standard mysqli_connect ne gérant pas nativement le SSL, la méthode mysqli_ssl_set() a été utilisée avant d'appeler real_connect().</p> 
<pre style="background-color: #f0f0f0; padding: 5px; font-family: Consolas; font-size:10pt;"> 
mysqli_ssl_set($conn, null, null, null, null, null);
$conn->real_connect($db_host, $db_user, $db_pass, $db_name, 3306, null, MYSQLI_CLIENT_SSL);
</pre> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>12. Gestion de la Persistance et Suppression Réversible</strong></p> 
<p style="font-size:11pt;"><strong>12.1 Mécanisme de "Soft Delete"</strong></p> 
<p style="font-size:11pt;">La fonction de Suppression réversible était activée sur le bucket, conservant les objets supprimés pendant une durée de rétention jusqu'au 22 février 2026.</p> 

<p style="font-size:11pt;"><strong>12.2 Procédure de Restauration via Console</strong></p> 
<p style="font-size:11pt;">La restauration a nécessité de filtrer l'affichage du bucket pour inclure les "Objets supprimés de façon réversible uniquement", identifier les hashs correspondants, et utiliser l'outil de Restauration.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>13. Difficultés et Solutions</strong></p> 
<table style="border-collapse: collapse; width: 100%; font-size:11pt;" border="1"> 
<tr> 
<th style="border: 1px solid black; padding: 5px;">Problème</th> 
<th style="border: 1px solid black; padding: 5px;">Cause</th> 
<th style="border: 1px solid black; padding: 5px;">Solution</th> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">403 Forbidden</td> 
<td style="border: 1px solid black; padding: 5px;">Droits insuffisants sur le bucket</td> 
<td style="border: 1px solid black; padding: 5px;">Attribution rôle Administrateur des objets Storage</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Access Denied SQL</td> 
<td style="border: 1px solid black; padding: 5px;">SSL requis mais non configuré</td> 
<td style="border: 1px solid black; padding: 5px;">Utilisation de mysqli_ssl_set() avec flag MYSQLI_CLIENT_SSL</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Date d'expiration incorrecte</td> 
<td style="border: 1px solid black; padding: 5px;">Fuseau horaire non défini</td> 
<td style="border: 1px solid black; padding: 5px;">date_default_timezone_set('Europe/Paris')</td> 
</tr> 
<tr> 
<td style="border: 1px solid black; padding: 5px;">Erreur 500</td> 
<td style="border: 1px solid black; padding: 5px;">Fichiers supprimés accidentellement</td> 
<td style="border: 1px solid black; padding: 5px;">Restauration via "Objets supprimés réversiblement"</td> 
</tr> 
</table> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>14. Conclusion</strong></p> 
<p style="font-size:11pt;">L'infrastructure déployée répond à toutes les exigences du projet. L'application web permet l'upload et le téléchargement sécurisé de fichiers temporaires. La base de données Cloud SQL stocke les métadonnées, le bucket Cloud Storage contient les fichiers. La VM a été configurée avec les bonnes pratiques de sécurité (IAM, scopes, SSL) et le réseau isolé via IP privée.</p> 
<p style="font-size:11pt;">Les difficultés rencontrées ont permis d'approfondir la compréhension des mécanismes d'authentification et de sécurisation sur Google Cloud Platform.</p> 
<p style="font-size:11pt;"> </p> 

<p style="font-size:11pt;"><strong>15. Annexes</strong></p> 
<p style="font-size:11pt;"><strong>15.1 Commandes récapitulatives</strong></p> 
<pre style="background-color: #f0f0f0; padding: 5px; font-family: Consolas; font-size:10pt;"> 
# Installation LAMP
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 php php-mysqli libapache2-mod-php composer -y

# SDK Google Cloud
cd /var/www/html
composer require google/cloud-storage

# Connexion SSL MySQL
mysqli_ssl_set($conn, null, null, null, null, null);
$conn->real_connect($db_host, $db_user, $db_pass, $db_name, 3306, null, MYSQLI_CLIENT_SSL);

# Configuration fuseau horaire
date_default_timezone_set('Europe/Paris');
$date_exp = date('Y-m-d H:i:s', strtotime("+$secondes seconds"));
</pre> 
<p style="font-size:11pt;"> </p> 
<p style="font-size:11pt;">-------------------------- Younes --------------------------</p> 
</content> 
</document>
