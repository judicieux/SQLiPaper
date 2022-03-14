# SQLiPaper

Ce white paper traîtera les différents vecteurs d'attaque dans le cas où une injection SQL est possible.<br/>
L'injection SQL est un vecteur d'attaque qui offre la possibilité d'insérer du code sur mesure dans une application et de le faire exécuter automatiquement sur des systèmes backend inaccessibles.<br/>
Dans ce white paper j'utilise MySQL comme système de gestion de base de données relationnelle.<br/>
La faille SQLi représente actuellement presque 2/3 des attaques application web. C'est la faille la plus commune selon OWASP.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158233875-8b440a6b-4f4d-4f1c-8b3e-28ebf0aa1fdd.png)
<br/>

## Background
Voici les différents types d'SQLi énumérés du plus soft au plus edgy (selon moi):

1. Union based -> IB
2. Time based (inférentiel) -> OOB
3. Blind based (inférentiel) -> OOB
4. Routed based -> IB

IB: In band<br/>
OOB: Out of band
<br/><br/>
Dans les 4 cas, nous utiliserons les mêmes méthodes d'approche.<br/>
J'introduirai aussi quelques techniques de bypass basiques pour avoid plusieurs filtres présents dans des WAF.<br/>
J'évoquerai aussi la puissance des jointures SQL et à quoi ça sert de les utiliser dans le contexte d'une attaque.<br/>

# Setup
Avant de commencer, il est important de déployer un serveur web localement ainsi que PMA (phpMyAdmin).<br/>
Afin de comprendre l'essence de la faille il faut comprendre le code qui l'induit et sans mise en contexte c'est pas possible.<br/>
Pour ce faire je vais utiliser WampServer, c'est une plateforme de développement Web de type WAMP, permettant de faire fonctionner localement des scripts PHP.<br/>
Si on veut exécuter du SQL il nous faudra une base de donnée avec laquelle on pourra interagir.<br/> 
J'utilise phpMyAdmin qui est une application Web de gestion pour les systèmes de gestion de base de données MySQL<br/><br/>

# Steps
• Une fois tout installé, lancez WampServer.<br/>
• Rendez vous dans la path ``/www`` en appuyant sur:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158238868-e9d82156-c272-440a-8870-2474de621a1e.png)
<br/><br/>
• Créez votre dossier projet avec dedans fichier ``.php``, dans mon cas le dossier se nomme ``/sqlpaper`` et le fichier ``index.php``.

## Union Based
Rentrons dans le vif du sujet en commençant par le plus simple.
