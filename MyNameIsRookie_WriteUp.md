# CTF  ESGI
**Challenge : My Name is Rookie**
# Contexte :
**Nom** : My Name Is Rookie

**Description** M0th3r > Quelque chose me perturbe. Comment un Androïde a pu passer le test des pirates cybernétique. Duke le premier de son genre n’a été crée par personne du gouvernement. Aujourd’hui disparu je veux retrouver son core. Si tu veux m’aider, tu dois passer le test des pirate Cybernétique. C’est le test que Duke-083 a passé haut la main. Récupère tout ce que tu sais sur Zedcorp.

http = ctf.hacklab-esgi.org:5008 ssh = ctf.hacklab-esgi.org:5007

**Points** : 350

# Résolutions :
## Première partie :

Nous avons différentes informations données dans la description du challenge. Le ssh ne nous sert pas pour le moment etant donné que l'on a pas d'utilisateur, on décide de se diriger sur le site web (ctf.hacklab-esgi.org:5008).
Les premières informations intéressantes que l'on retrouve sur le site sont dans le robots.txt avec notamment le dossiers /logs

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web1.png "robots.png")

En accédant au dossier /logs, on remarque que l'on a accès à différents fichiers logs intéressants, comme access.log, access-details.logs
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/logs.PNG "logs.png")

On accéder au fichier access.log, on remarque dedans l'existence d'une page login.php :

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web2.png "web2.png")

Sans surprise on arrive sur une page d'authentification, on décide alors de regarder le access-details.logs, et on va retrouver une requête avec des credentials passés en paramètre :

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web3.png "web3.png")

On arrive sur une page d'administration, normalement si il n'y avait pas de soucis, on devrait avoir les logs affichés, mais là ce n'est pas le cas :

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/log2.PNG "log2.png")

On lance Burp Suite, pour voir s'il fait une requête spécifique, on remarque qu'il donne le nom du fichier en paramètre, on va donc tenter de combiner ça avec une autre commande pour voir si on a un retour
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web4.png "web4.png")
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web5.png "web5.png")

Bingo on a réussi à avoir un retour, donc on a accès à une RCE, on va l'utiliser pour chercher ce qui nous intéresse. 
En faisant un petit listing des fichiers dans le home, on récupère la clef privée d'un utilisateur :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web8.png "web8.png")
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web10.png "web10.png")

Avec la clef privée précédemment récupérée, on essaye de s'authentifier sur la machine via le port SSH donné en début de challenge :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web11.png "web11.png")

Malheureusement, on n'obtient pas de shell, mais un hint nous est donné "Do you know proxychains ?".
En l'occurrence proxychains est utilisé pour créer des tunnels, on peut donc en déduire qu'il se trouve d'autres machines derrières.
Mais nous n'avons jusqu'à maintenant aucune information sur le LAN derrière, et pour ne pas y aller en guessing on va essayer de récupérer des informations via notre RCE. 

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web13.png "web13.png")
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web15.png "web15.png")
Nous récupérons des informations dans le fichier /etc/hosts, mais également via la commande ip addr, nous savons donc que nous sommes sur la machines dev-server, et qu'il y a potentiellement deux autres machines :
10.0.0.2 project-server et 10.0.0.3 admin-server.

Nous allons donc nous créer un tunnel SOCKS via ssh, puis nous passerons par ce tunnel à l'aide de proxychains pour executer nos commandes :
```shell:
ssh -D 1337 -i key test@ctf.hacklab-esgi.org -p 5007 -N
```
-D : Nous créons notre tunnel sur le port défini par la suite
-N : pour ne pas exécuter de commande


## Deuxième partie :

On configure donc notre proxychains.conf pour utiliser le port 1337 sur notre localhost, puis on balance les nmaps pour voir ce qu'il tourne sur les deux autres serveurs :

![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/nmap.png "nmap.png")

On voit différents services sur les deux machines, on commence par la machine 10.0.0.2, on tombe sur une interface web de gestion de tâche où on ne pas voir grand chose hormis des pistes pour la suite
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web21.png "web21.png")

Sur ce serveur, nous ne trouvons pas grand choses, hormis que l'on a pu récupérer la version de l'Apache Tomcat :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web22.png "web22.png")

En cherchant un peu des vulnérabilités liées à cette version, on remarque une CVE qui nous permet d'upload un fichier JSP via la méthode PUT :
https://nvd.nist.gov/vuln/detail/CVE-2017-12617

On tente alors d'exploiter cette vulnérabilité, et d'upload un webshell au format .jsp via curl sans certitude.
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web24.png "web24.png")

On va ensuite se diriger sur la page, et on essaye d'exécuter des commandes sur notre webshell voir si cela a bien fonctionné.
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web25.png "web25.png")
Bingo, on arrive à exécuter des commandes, on voit qu'on a l'utilisateur tomcat, donc assez limité, mais on va chercher des choses qui peuvent nous intéresser :)

En reprenant les indices que l'on avait eu précédemment, on se rappelle que des credentials avait été trouvé sur le project-server, donc comme à mon habitude, je fais un listing de tous les fichiers des répertoires des utilisateurs, on remarque beaucoup de fichiers, pas de credentials, mais cette tâche était marqué en "Done".
La 2eme information c'est qu'un serveur ftp a été ajouté pour sauvegarder des fichiers, peut-être les fichiers d'identifiants...
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web27.png "web27.png")

On remarque des fichiers .bash_history accessible en lecture, on décide d'aller regarder un peu tout ça, surtout celui de l'utilisateur dcloutier qui semble un peu volumineux par rapport aux autres.
On retrouve énormément d'informations dedans qui pourront peut-être nous servir par la suite :
Une connexion ftp sur le serveur admin avec les credentials dans la ligne de commande, donc le 10.0.0.3.
username backup
password : 46t5r2e5t&2z

On a aussi la compression d'une archive, ensuite chiffré avec openssl avec le mot de passe pass:daniel2019
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web28.png "web28.png")

On essaye donc de se connecter sur le serveur ftp du serveur admin avec les credentials précédemment récupérés, et petite surprise...
Ca fonctionne :), on arrive même à récupérer l'archive en question :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web29.png "web29.png")

On va donc faire l'opération inverse pour déchiffrer l'archive :
```shell:
openssl enc -d -aes256 -in credentials.tar.gz -out credentials2.tar.gz -pass pass:daniel2019
tar -xvf credentials2.tar.gz
```
On récupère dans cette archive un fichier credentials.txt avec identifiants pour le serveur admin :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web30.png "web30.png")
Dont les identifiants pour l'auth basic, et les identifiants de la plateforme qui se trouve derrière.

On s'authentifie avec tous les comptes et on accède à une page sans pouvoir rien faire, ni d'upload etc...
Avec l'utilisation de burp, on remarque qu'en fonction du compte sur lequel on se connecte, un cookie "status" est présent et semble encodé en base64, on décide de le décoder pour voir ce qu'on récupère :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/cookie1.png "cookie1")
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/cookie2.png "cookie2")
Pour **user** --> user
Pour **Admin** --> admin

Si on suit la logique
Pour l'utilisateur **CEO** --> ceo qui donne Y2Vv encodé en base64

On se connecte et on donne le cookie status=**Y2Vv** et bingo on accède à l'espace du CEO.
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web36.png "web36.png")

Puis on récupère le flag dans les fichier PDF :
![alt text](https://github.com/ishusoka/CTF_ESGI/blob/master/Images/web37.png "web37.png")

Le flag final est **ESGI{W3_H0p3_t3_S33_y0u_N3xT_Y34R:)}**

Merci encore à Th1b4ud pour ce challenge Web :)
