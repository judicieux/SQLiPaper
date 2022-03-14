# SQLiPaper

Ce white paper traîtera les différents vecteurs d'attaque dans le cas où une injection SQL est possible.<br/>
L'injection SQL est un vecteur d'attaque qui offre la possibilité d'insérer du code sur mesure dans une application et de le faire exécuter automatiquement sur des systèmes backend inaccessibles.<br/>
Dans ce white paper j'utilise ``MySQL`` comme système de gestion de base de données relationnelle.<br/>
La faille SQLi représente actuellement presque 2/3 des attaques application web. C'est la faille la plus commune selon OWASP.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158233875-8b440a6b-4f4d-4f1c-8b3e-28ebf0aa1fdd.png)
<br/>

## Background
Voici les différents types d'SQLi énumérés du plus soft au plus edgy (selon moi):

1. Union based -> ``IB``
2. Time based (inférentiel) -> ``OOB``
3. Blind based (inférentiel) -> ``OOB``
4. Routed based -> ``IB``

``IB``: In band<br/>
``OOB``: Out of band
<br/><br/>
Dans les 4 cas, nous utiliserons les mêmes méthodes d'approche.<br/>
J'introduirai aussi quelques techniques de bypass basiques pour avoid plusieurs filtres présents dans des ``WAF``.<br/>
J'évoquerai aussi la puissance des **jointures SQL** et à quoi ça sert de les utiliser dans le contexte d'une attaque.<br/>

# Setup
Avant de commencer, il est important de déployer un serveur web localement ainsi que ``PMA`` (phpMyAdmin).<br/>
Afin de comprendre l'essence de la faille il faut comprendre le code qui l'induit et sans mise en contexte c'est pas possible.<br/>
Pour ce faire je vais utiliser ``WampServer``, c'est une plateforme de développement Web de type ``WAMP``, permettant de faire fonctionner localement des scripts ``PHP``.<br/>
Si on veut exécuter du SQL il nous faudra une base de donnée avec laquelle on pourra interagir.<br/> 
J'utilise ``PMA`` qui est une application Web de gestion pour les systèmes de gestion de base de données ``MySQL``<br/><br/>

# Steps
### WampServer
• Une fois tout installé, lancez WampServer.<br/>
• Rendez vous dans la path ``/www`` en cliquant sur:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158238868-e9d82156-c272-440a-8870-2474de621a1e.png)
<br/><br/>
• Créez votre dossier projet avec dedans un fichier ``.php``, dans mon cas le dossier se nomme ``/sqlipaper`` et le fichier ``index.php``.<br/>
• Mettez votre adresse ``loopback`` avec le dossier correspondant dans n'importe quel navigateur web (``http://127.0.0.1/sqlipaper/index.php``).<br/>
• Si vous voyez la petite icone WampServer c'est que votre fichier PHP est bien hébergé, vous êtes donc prêt à écrire dedans.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158240258-d9cd88c3-33ae-4659-a3a5-0a38a5cfd94d.png)
<br/><br/>
### phpMyAdmin
• J'ai choisi WampServer car il nous permet aussi d'accéder à phpMyAdmin.<br/>
• Même étape vous lancez WampServer et vous cliquez sur:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158241274-b9f23547-1a1b-49c9-b915-306d45921b0c.png)
<br/><br/>
• Puis vous créez une nouvelle base de donnée en suivant les étapes figurées ci-dessous:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158242535-0b05fe6e-d5f4-4498-82c4-b09bad167bd9.png)
<br/><br/>
• Avant d'attaquer la suite, j'aimerais vous faire comprendre de quoi est structurée une base de donnée.<br/>
• Une base de données permet de stocker et de retrouver des données structurées, semi-structurées ou des données brutes ou de l'information (cf. ixa qui stocke tout sous txt ``¯\_(ツ)_/¯``).<br/>
• Je vous ai fait une petite représentation à l'aide de Lightshot:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158244976-9db7876b-a1c5-4598-a7b3-ca2cc81b535b.png)
<br/><br/>
Grossomodo, chaque base de données contient des tables, qui elles contiennent des colonnes où nous pouvons stocker des données.<br/>

## Union Based
Rentrons dans le vif du sujet en commençant par le plus simple.
