# SQLiPaper

Ce white paper traîtera les différents vecteurs d'attaque dans le cas où une injection SQL est possible.<br/>
L'injection SQL est un vecteur d'attaque qui offre la possibilité d'insérer du code sur mesure dans une application et de le faire exécuter automatiquement sur des systèmes backend inaccessibles.<br/>
Dans ce white paper j'utilise MySQL comme système de gestion de base de données relationnelle.<br/>
La faille SQLi représente actuellement presque 2/3 des attaques application web. C'est la faille la plus commune selon OWASP.<br/><br/>
![image](https://user-images.githubusercontent.com/74382279/158233875-8b440a6b-4f4d-4f1c-8b3e-28ebf0aa1fdd.png)
<br/>

## Background
Voici les différents types d'SQLi énumérés du plus edgy au plus soft (selon moi):

1. Routed based -> IB
2. Blind based (inférentiel) -> OOB
3. Time based (inférentiel) -> OOB
4. Union based -> IB

IB: In band<br/>
OOB: Out of band
<br/><br/>
Dans les 4 cas, nous utiliserons les mêmes méthodes d'approche.<br/>
J'introduirai aussi quelques techniques de bypass basiques pour avoid plusieurs filtres présents dans des WAF.<br/>
J'évoquerai aussi la puissance des jointures SQL.
