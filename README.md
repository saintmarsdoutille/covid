# covid
statistiques diverses sur l'épidémie de covid

Les sources :
décés mensuels https://www.insee.fr/fr/information/4190491
décés annuels https://www.insee.fr/fr/information/4769950

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
 mysql -uroot -Dinsee -e  "select *  from deces where datedeces NOT LIKE '20%' AND datedeces NOT LIKE '19%' limit 5;" 
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

