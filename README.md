# covid
statistiques diverses sur l'épidémie de covid

Les sources :
décés mensuels https://www.insee.fr/fr/information/4190491
décés annuels https://www.insee.fr/fr/information/4769950

```bash
wget https://www.insee.fr/fr/statistiques/fichier/4769950/deces-2010-2018-csv.zip
wget https://www.insee.fr/fr/statistiques/fichier/4769950/deces-2000-2009-csv.zip
wget https://www.insee.fr/fr/statistiques/fichier/4769950/deces-1990-1999-csv.zip
wget https://www.insee.fr/fr/statistiques/fichier/4769950/deces-1980-1989-csv.zip
wget https://www.insee.fr/fr/statistiques/fichier/4769950/deces-1970-1979-csv.zip

wget https://www.insee.fr/fr/statistiques/fichier/4190491/Deces_2019.zip
wget https://www.insee.fr/fr/statistiques/fichier/4190491/Deces_2018.zip
wget https://www.insee.fr/fr/statistiques/fichier/4190491/Deces_2020.zip
```

sur les fichiers mensuels et annuels, les entêtes sont différentes 
```bash
head -n1  mensuel/Deces_2020_M12.csv 
"nomprenom";"sexe";"datenaiss";"lieunaiss";"commnaiss";"paysnaiss";"datedeces";"lieudeces";"actedeces"

head -n1 annee/deces-2011.csv 
"nomprenom";"sexe";"datenaiss";"lieunaiss";"commnaiss";"paysnaiss";"datedeces";"lieudeces";"actedeces"

```
Comment rendre conforme les fichiers mensuels à la bdd visée

```bash
cat Deces_2020_M01.csv |sed 's/\*/";"/g'|sed 's/\/"/"/g' > temp01.csv
```

création de la base de donnée
```bash
mysql -uroot -Dinsee -e "DROP TABLE IF EXISTS deces;"
mysql -uroot -Dinsee -e "CREATE TABLE deces ( nom TEXT NOT NULL, prenom TEXT NOT NULL, \
sexe INTEGER, datenaiss TEXT NOT NULL, lieunaiss TEXT NOT NULL, commnaiss TEXT NOT NULL, \
paysnaiss TEXT NOT NULL, datedeces TEXT NOT NULL, lieudeces TEXT NOT NULL, actedeces TEXT NOT NULL , age INTEGER);"
```
la dernière colone est ajoutée par rapport aux colones des fichiers csv, elle sera calculée pour des recherches rapides sur l'âge du décés

import du fichier csv dans la base de données
```bash
mysql -uroot -Dinsee -e "LOAD DATA LOCAL INFILE 'temp01.csv' INTO TABLE deces \
CHARACTER SET latin1 FIELDS TERMINATED BY ';' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 ROWS;"
```

pour les fichier annuels, on a simplement besoin de supprimer le / en trop dans la colone prénom

```bash
cat annee/deces-2011.csv |sed 's/\/"/"/g' > temp2011.csv
```


calcul de l'age de décés
```bash
mysql -uroot -Dinsee -e  "update deces set age=left(datedeces,4)-left(datenaiss,4) ;"
```
on obtient une erreur
```
ERROR 1292 (22007) at line 1: Truncated incorrect DOUBLE value: 'MARO'
```
une erreur s'est glissée dans la ligne 
```bash
cat mensuel/Deces_2020_M*.csv|grep "*\""
"SEDDOURI*MOHAMMED/";"1";"20000710";"99350";"FES*";"MAROC";"20200720";"93039";"31"
```
il faut traiter ce point séparément
```bash
cat mensuel/Deces_2020_M09.csv |sed 's/*"/"/g'|sed 's/\*/";"/g'|sed 's/\/"/"/g' > temp09.csv
```

après réflexion, commençons par compter le nombre de lignes incohérentes
```bash
 mysql -uroot -Dinsee -e  ""select count(*) from deces where datedeces NOT LIKE '20%' AND datedeces NOT LIKE '19%' ;" 
 
 +-----------+
| count(* ) |
+-----------+
|        13 |
+-----------+
 ```
13 lignes incohérentes sur plus de 6 milions, autant les effacer, ça ne changera pas les pourcentages
```bash
mysql -uroot -Dinsee -e  "delete  from deces where datedeces NOT LIKE '20%' AND datedeces NOT LIKE '19%' ;" 
 ```
## tranches d'age ##
on pourrait faire une recherche par age
```bash 
mysql -uroot -Dinsee -e  "select  count(*) FROM deces WHERE datedeces LIKE '2020%' and age < 20;"
```

nous allons plutôt classer les deces par tranche d'âge de 10 ans
```bash
mysql -uroot -Dinsee -e  "update deces set age=ROUND((left(datedeces,4)-left(datenaiss,4))/10) ;"
```

## requetes finales ##
chaque année, on groupe les décés par tranches d'âge
```bash
mysql -uroot -Dinsee -e  "select age,count(*) FROM deces WHERE datedeces LIKE '2015%' group by age;"
+------+----------+
| age  | count(*) |
+------+----------+
|    0 |     3687 |
|    1 |      687 |
|    2 |     3066 |
|    3 |     4251 |
|    4 |    11486 |
|    5 |    24286 |
|    6 |    64091 |
|    7 |    74942 |
|    8 |   163963 |
|    9 |   200427 |
|   10 |    50455 |
|   11 |      609 |
|   12 |        2 |
+------+----------+
```


