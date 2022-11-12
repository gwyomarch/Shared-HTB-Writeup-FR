# Shared HTB

![Shared HTB](img/Shared.png)

Une box Linux de difficulté Medium créée par [**Nauten**](https://app.hackthebox.com/users/27582) qui nous amènera à récupérer, grâce à une injection SQL, le hash du mot de passe d'un utilisateur dans la base de données derrière un site Prestashop. Puis elle nous permettra de compromettre un deuxième utilisateur en exploitant une vulnérabilité sur un package python ([**iPython Command Execution (CVE-2022-21699)**](https://github.com/ipython/ipython/security/advisories/GHSA-pq7m-3gw7-gq5x)). Ensuite, l'exploitation d'un binaire nous donnera un mot de passe qui nous donnera la possibilité de nous échapper d'une sandbox Redis ([**Redis LUA Sandbox Escape (CVE-2022-0543)**](https://thesecmaster.com/how-to-fix-cve-2022-0543-a-critical-lua-sandbox-escape-vulnerability-in-redis/)) avec les droits root.


## Enumeration

### NMAP

Premièrement, on lance un scan Nmap de la machine hôte avec les scripts de détection par défaut pour trouver des ports TCP ouverts.

![Nmap Scan](img/01-Nmap.JPG)

3 services tournent dessus:
- SSH sur le port 22
- HTTP sur le port 80 avec un nom de domaine associé: **shared.htb**
- HTTPS sur le port 443 avec un nom de domaine associé: **shared.htb**

On est rediriger vers **http://shared.htb**, qui redirige lui-même vers **https://shared.htb**.

![Curl Check](img/02-Curl.JPG)

Ajoutons-le à notre fichier **/etc/hosts** et allons voir à quoi ressemble ce site web.

On accepte le certificat et on arrive sur un site Prestashop.

![Browser](img/03-Browser.JPG)

![Prestashop](img/04a-Prestashop.JPG)

### GOBUSTER

On lance une attaque par dictionnaire avec [**Gobuster**](https://github.com/OJ/gobuster) pour énumérer les repertoires et fichiers à la racine du site en HTTP.

![Gobuster Directories](img/05-Gobuster-Dir.JPG)

Une api avec quelques endpoints et un Makefile. 
Jetons un œil au fichier **robots.txt**.

![Robots](img/04b-Robots-txt.JPG)

On a la confirmation que Prestashop a été installé et partiellement fonctionnel.
Voyons si il y a des sous-domaine.

![Gobuster Subdomain](img/06-Gobuster-VHosts.JPG)

On trouve **checkout.shared.htb** mais rien n'est possible directement.

Reprenons les choses dans l'ordre, allons faire un peu de shopping.

![Cart Shopping](img/07-Shopping.JPG)

Remplissons notre panier et dirigeons-nous vers la confirmation.

![Checkout](img/08-Checkout.JPG)

Nous revoila sur la page **checkout** mais avec des données à manipuler.

## EXPLOIT (SQL Injection)

Un champ **custom_cart** dans les cookies qui s'avère être vulnérable à une Injection SQL.

### BURPSUITE

![Database](img/09-Database.JPG)

On récupère le nom de la base de données, puis des tables... La table **user** peut contenir des informations intéressantes.

![Tables](img/10-Tables.JPG)

On récupère ses colonnes.

![Columns](img/11-Columns.JPG)

Un seul utilisateur enregistré **james_mason** et le hash de son mot de passe, en MD5 à première vue.

![Users](img/12-User.JPG)

Essayons de cracker son mot de passe.

![Crack Password](img/13-Cracked.JPG)

Mais on aurait aussi très bien pu enregistrer la requête dans un fichier depuis BurpSuite et utiliser SQLMap.

![BurpSuite File](img/13d-BurpSuite-File.png)

![SQLMap Injection](img/13a-SQLmap-Injection.JPG)

![SQLMap Tables](img/13b-SQLmap-Tables.JPG)

![SQLMap Dump](img/13c-SQLmap-Dump.JPG)




## FOOTHOLD

Avec son mot de passe, on peut se connecter en SSH à la machine cible.

![SSH Foothold](img/14-SSH-Foothold.JPG)

Cet utilisateur fait partie du groupe **developer**, ce qui lui donne tout les droit sur le dossier **/opt/scripts_review**.

Mais pas encore de flag...

![Developer Group](img/15-Developer-Group.JPG)

On sait qu'il y a un autre utilisateur, qui a lui aussi une configuration SSH dans son répertoire HOMEDIR.

(C'est surement lui qui détient le flag)

![Users](img/16-Users.JPG)

Uploadons [PsPy](https://github.com/DominicBreuker/pspy) sur la machine cible pour énumérer les processus en temps réel.

![Pspy](img/17-Pspy.JPG)

Il nous révèle un processus qui lance **iPython** (interactive python) depuis **/opt/scripts_review**.


### CVE-2022-21699

Après quelques recherches sur ce package, il comporte une [**vulnérabilité d'exécution de code arbitraire**](https://github.com/ipython/ipython/security/advisories/GHSA-pq7m-3gw7-gq5x) à travers un modèle de système de dossiers spécifique, à partir du répertoire courant.

On sait que **iPython** est lancé depuis **/opt/scripts_review** sur lequel nous avons les droits en écriture, et que la clé SSH **id_rsa** de l'utilisateur **dan_smith** est possiblement à notre portée durant l'exécution.

![iPython](img/18-Exploit.JPG)

On créer un petit script **foo.py** pour récupérer la clé SSH. On le place dans le répertoire **profile_default/startup** à partir de celui dans lequel iPython est lancé.

![Get Key](img/19-Get-Key.JPG)


## Lateral Movement 

### SSH

![SSH Lateral Movement](img/20-SSH-Lateral.JPG)

Si la clé n'a pas de mot de passe, on peut maintenant se connecter à la machine via le service SSH en tant que **dan_smith**.

![Groups](img/21-Group.JPG)

Cette fois, cet utilisateur fait parti du groupe **sysadmin** avec un executable associé **redis_connector_dev** qui, d'après son nom, permet au developpeur de se connecter au système de gestion de base de données Redis, je suppose.

![Redis Connector](img/22-Redis-Program.JPG)

Il se connecte sur le port 6379 par défaut et utilise un mot de passe. 

On va le télécharger sur notre machine pour l'examiner.

![Setup Download](img/23-Setup-Download.JPG)

![Download Redis Connector](img/24-Download.JPG)

Lançons le binaire en écoutant sur le port 6379 avec netcat pour intercepter l'authentification.

![Run Redis Connector](img/25-Run-Redis.JPG)

Il nous donne le mot de passe pour accéder au service Redis.

![Get Redis Password](img/26a-Get-Password.png)

## Privilege Escalation

### CVE-2022-0543 (Lua Sandbox Escape)

On s'y connecte sans préciser d'utilisateur, ça doit être **default**, l'utilisateur par défaut de Redis.
Puis, on lance un [**script Lua**](https://thesecmaster.com/how-to-fix-cve-2022-0543-a-critical-lua-sandbox-escape-vulnerability-in-redis/) pour échapper au bac à sable Lua et lancer un reverse shell en bash.

![LUA Exploit](img/26b-LUA-Exploit.JPG)

Ca fonctionne pour lancer quelques commandes mais ce n'est pas stable, la connexion coupe au bout d'une minute environ.

![Get Root With POC](img/26c-Get-Root.JPG)

J'ai continué en quête d'une meilleur méthode. 

### CHISEL

On va utiliser [**Chisel**](https://github.com/jpillora/chisel) pour faire du port forwarding, rediriger le port 6379 de la cible vers le port 6379 de notre machine en la faisant écouter sur le port 9999.

![Tunneling Schema](img/Port-Forwarding.png)

Pour mettre en place notre tunnel, il faut l'exécutable **chisel** sur les 2 machines.

![Upload Chisel](img/27a-Upload-Chisel.JPG)

Et on le télécharge sur la cible.

![Download Chisel](img/27b-Download-Chisel.JPG)

On met en place **chisel** en mode serveur en écoute sur le port 9999 de notre machine. 

![Chisel Server](img/28-Chisel-Server.JPG)

Puis en mode client sur la cible sur le port 6379 en entrée et en sortie.

![Chisel Client](img/29-Chisel-Client.JPG)

Nous pouvons maintenant utiliser le service Redis depuis notre machine comme si il y était.
Cette fois-ci, pour l'exploiter, nous allons utliser un [**script python**](https://github.com/aodsec/CVE-2022-0543/) trouver pendant mes recherches sur cette CVE.

![Modified Script](img/30-Modified-Script.JPG)

Une fois le script modifié et ajusté pour notre situation, nous pouvons enfin récupérer le flag **root**.


![Get Root](img/31-Get-Root.JPG)

Merci [**Nauten**](https://app.hackthebox.com/users/27582) à pour cette Box super intéressantes avec des CVEs récentes.

N'hésitez pas à aller lui donner du "respect" sur son profil si elle vous a plu.

### Liens:
- [**Gobuster**](https://github.com/OJ/gobuster)
- [**PsPy - unprivileged Linux process snooping**](https://github.com/DominicBreuker/pspy)
- [**iPython Command Execution (CVE-2022-21699)**](https://github.com/ipython/ipython/security/advisories/GHSA-pq7m-3gw7-gq5x)
- [**Chisel - TCP/UDP tunneling**](https://github.com/jpillora/chisel)
- [**Redis LUA Sandbox Escape POC (CVE-2022-0543)**](https://thesecmaster.com/how-to-fix-cve-2022-0543-a-critical-lua-sandbox-escape-vulnerability-in-redis/)
- [**Redis LUA Sandbox Escape original script (CVE-2022-0543)**](https://github.com/aodsec/CVE-2022-0543/)
- [**Profil HackTheBox de Nauten**](https://app.hackthebox.com/users/27582)


