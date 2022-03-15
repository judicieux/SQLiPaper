# SQLiPaper

Ce white paper traîtera les différents vecteurs d'attaque dans le cas où une ``injection SQL`` est possible.<br/>
L'injection SQL est un vecteur d'attaque qui offre la possibilité d'insérer du code sur mesure dans une application et de le faire exécuter automatiquement sur des systèmes backend inaccessibles.<br/>
Dans ce white paper j'utilise ``MySQL`` comme système de gestion de base de données relationnelle.<br/>
La faille SQLi représente actuellement presque ``2/3`` des attaques application web. C'est la faille la plus commune selon OWASP.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158233875-8b440a6b-4f4d-4f1c-8b3e-28ebf0aa1fdd.png)
<br/>

## Background
Voici les différents types d'SQLis énumérés du plus soft au plus edgy (selon moi):

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
• Une base de données permet de stocker et de retrouver des données structurées, semi-structurées ou des données brutes ou de l'information (cf. ixa qui stocke tout sous txt à coup de ``fopen()`` et ``fgets()`` ``¯\_(ツ)_/¯``).<br/>
• Je vous ai fait une petite représentation à l'aide de Lightshot:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158244976-9db7876b-a1c5-4598-a7b3-ca2cc81b535b.png)
<br/><br/>
Grossomodo, chaque base de données contient des tables, qui elles contiennent des colonnes où nous pouvons stocker des données.<br/>
Pour la création de ces derniers on va exécuter du code SQL, la création peut aussi se faire de façon interfacée.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158255790-7115a9e6-0db9-4fea-a6d7-7dcb43aece00.png)
<br/><br/>
```sql
CREATE TABLE users (
	id int(10) AUTO_INCREMENT NOT NULL,
	username VARCHAR(100) NOT NULL,
	PRIMARY KEY(id)
) ENGINE=innodb AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```
• Comme son nom l'indique, l'instruction ``CREATE TABLE`` permet de créer une table. Ici nous créons la table ``users``.<br/>
• On assigne deux colonnes à la table ``users``, notamment ``id`` et ``username``. On passe la valeur ``PRIMARY KEY`` à la colonne ``id``.<br/><br/>
• La contrainte ``PRIMARY KEY`` identifie de manière unique chaque enregistrement dans une table. Les clés primaires doivent contenir des valeurs uniques et ne peuvent pas contenir de valeurs ``NULL`` ou de doublons. Une table ne peut avoir qu'une clé primaire ; et dans le tableau, cette clé primaire peut consister en une ou plusieurs colonnes (champs).<br/><br/>
• La commande ``AUTO_INCREMENT`` est utilisée afin de spécifier qu’une colonne ``int`` avec une ``PRIMARY KEY`` sera incrémentée automatiquement à chaque ajout d’enregistrement dans celle-ci.<br/><br/>
• ``int(10)`` signifie qu'on a défini ``id`` comme ``INT UNSIGNED``. Ainsi, on pourra stocker des nombres allant de ``0`` jusqu'à ``4294967295`` (à noter que la valeur maximale est composée 10 chiffres, donc MySQL ajoute automatiquement le (10) dans la définition de la colonne qui (10) n'est qu'un indice de format et rien de plus. Ça n'a aucun effet sur la taille du nombre que vous pouvez stocker).<br/><br/>
• Ici j'utilise ``VARCHAR``. Mais je peux aussi utiliser ``CHAR``. La différence est que ``CHAR`` est un type de données de longueur fixe, la taille de stockage de la valeur char est égale à la taille maximale de cette colonne. Étant donné que ``VARCHAR`` est un type de données de longueur variable, la taille de stockage de la valeur ``VARCHAR`` correspond à la longueur réelle des données saisies, et non à la taille maximale de cette colonne. Imaginons que j'insère la valeur ``zyuomo``, si ``CHAR`` était utilisé. Il aurait rempli le nombre de caractères restants par des espaces. Donc la valeur serait stockée avec 96 espaces. Tandis que si j'utilise ``VARCHAR``, la taille de la valeur correspondra à la taille de la valeur inscrite et non à la taille prédifinie (``100``).<br/><br/>
• ``ENGINE=innodb`` permet de définir le moteur de stockage par défaut utilisé par MySQL. Son principal avantage par rapport aux autres moteurs de stockage de MySQL est qu'il permet des transactions ``ACID`` (atomiques, cohérentes, isolées et durables), ainsi que la gestion des clés étrangères avec vérification de la cohérence.<br/><br/>
• Utilisez toujours ``DEFAULT CHARSET=utf8`` lors de la création de nouvelles tables. A ce stade, votre client et votre serveur MySQL doivent être en ``UTF-8`` (cf. https://my.cnf). n'oubliez pas que tous les langages que vous utilisez (tels que ``PHP``) doivent également être sous ``UTF-8``. Certaines versions de PHP utilisent leur propre lib client MySQL, qui peut ne pas être compatible avec ``UTF-8``.<br/>
Il nous reste qu'à exécuter:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158256871-88e2a443-5c71-435e-9238-b9c3537e6bd9.png)
<br/><br/>
On peut constater que la table ``users`` et les colonnes ``id`` & ``username`` ont bien été créé dans la base de donnée ``sqlipaper`` avec les paramètres qu'on leur a assigné.<br/>
Maintenant, quoi faire d'une base de donnée qui ne contient aucune donnée? Eh bien rien, c'est pour ça qu'on va directement insérer une donnée dans la colonne ``id`` et ``username`` de la manière suivante:<br/>
```sql
INSERT INTO users VALUES(1, "zyuomo");
```
Vous l'aurez deviné, l'instruction ``INSERT INTO`` est utilisé pour insérer de nouveaux enregistrements dans une table.<br/>
J'insère la valeur ``1`` qui correspond à la colonne ``id`` et ``zyuomo`` à la colonne ``username``.<br/>
Tout est prêt, il nous manque plus qu'à scripter le code SQL vulnérable.<br/>
On se rend au fichier ``PHP`` qu'on a créé tout à l'heure.<br/>
On a plus qu'à provoquer la faille, 

## Union Based
Rentrons dans le vif du sujet en commençant par le plus simple.<br/>

L'opérateur SQL ``UNION`` est utilisé pour combiner les résultats de deux déclarations ``SELECT`` ou plus.<br/>
Voici une instruction typique qui utilise l'opérateur UNION:
```sql
UNION SELECT id, username FROM users;
```
Cette instruction demande à la ``BDD`` de retourner la colonne ``id`` et ``username`` de la table ``users``.<br/>

### Rerences
- https://stackoverflow.com/questions/10879345/what-is-the-maximum-size-of-int10-in-mysql
- https://stackoverflow.com/questions/202205/how-to-make-mysql-handle-utf-8-properly
- https://dev.mysql.com/
- https://www.cisecurity.org/wp-content/uploads/2017/05/SQL-Injection-White-Paper2.pdf
- https://www.w3schools.com/sql/
