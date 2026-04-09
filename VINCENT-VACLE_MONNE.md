# Markdown CC2 - Léon VINCENT VACLE et Nolann MONNE

GitHub : https://github.com/LeonVV74000/CC2_Nolann_MONNE_Leon_VINCENT-VACLE

---

## Préparation de l'environnement

### Connexion SSH et passage en root

```
ssh maria_dev@localhost -p 2222
sudo su root
```

### Installation de Nano

Alternative à `yum`(qui ne fonctionne pas sur mon PC) en téléchargeant le RPM directement :

```
wget --no-check-certificate https://vault.centos.org/7.9.2009/os/x86_64/Packages/nano-2.3.1-10.el7.x86_64.rpm
rpm -ivh nano-2.3.1-10.el7.x86_64.rpm
```

### Installation de pip et MRJob

On passe par curl car je n'arrive pas à faire fonctionner 'yum' :

```
curl -k https://bootstrap.pypa.io/pip/2.7/get-pip.py -o /tmp/get-pip.py
python /tmp/get-pip.py
pip install pathlib PyYAML==5.4.1 mrjob==0.7.4
```

### Import des données

```
su maria_dev
cd ~
wget --no-check-certificate https://files.grouplens.org/datasets/movielens/ml-25m.zip
unzip ml-25m.zip
hdfs dfs -mkdir datasets
hdfs dfs -put ml-25m/tags.csv datasets/
```

---

## 1. Combien de tags chaque film possède-t-il ?

### Script Python : `tags_par_film.py`

```
from mrjob.job import MRJob
from mrjob.step import MRStep

class TagsParFilm(MRJob):
    def steps(self):
        return [MRStep(mapper=self.mapper_get_tags, reducer=self.reducer_count_tags)]

    def mapper_get_tags(self, _, line):
        try:
            (userId, movieId, tag, timestamp) = line.split(',', 3)
            if userId != 'userId':
                yield movieId, 1
        except Exception:
            pass

    def reducer_count_tags(self, movieId, counts):
        yield movieId, sum(counts)

if __name__ == '__main__':
    TagsParFilm.run()
```

### Commande Hadoop :

```
#Créer fichier python
nano tags_par_film.py
#Exécuter python
python tags_par_film.py -r hadoop --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar hdfs:///user/maria_dev/datasets/tags.csv -o hdfs:///user/maria_dev/output_par_film
```

### Echantillon du résultat :

```
hdfs dfs -cat /user/maria_dev/output_par_film/part-00000 | head -n 15

"1"     697
"10"    137
"100"   18
"1000"  10
"100001"        1
"100003"        3
"100008"        9
"100017"        9
"100032"        2
"100034"        19
"100036"        1
"100038"        4
"100042"        2
"100044"        12
"100046"        3
```

Le mapper émet `(movieId, 1)` pour chaque ligne valide. Le header est ignoré via la condition `userId != 'userId'`. Les instructions sont encapsulées dans un bloc `try ... except` pour gérer les lignes malformées. Le reducer agrège les compteurs par `movieId`. On obtient 45 251 films distincts ayant au moins un tag.

---

## 2. Combien de tags chaque utilisateur a-t-il ajoutés ?

### Script Python : `tags_par_utilisateur.py`

```
from mrjob.job import MRJob
from mrjob.step import MRStep

class TagsParUtilisateur(MRJob):
    def steps(self):
        return [MRStep(mapper=self.mapper_get_tags, reducer=self.reducer_count_tags)]

    def mapper_get_tags(self, _, line):
        try:
            (userId, movieId, tag, timestamp) = line.split(',', 3)
            if userId != 'userId':
                yield userId, 1
        except Exception:
            pass

    def reducer_count_tags(self, userId, counts):
        yield userId, sum(counts)

if __name__ == '__main__':
    TagsParUtilisateur.run()
```

### Commande Hadoop :

```
#Créer fichier python
nano tags_par_utilisateur.py
#Exécuter python
python tags_par_utilisateur.py -r hadoop --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar hdfs:///user/maria_dev/datasets/tags.csv -o hdfs:///user/maria_dev/output_par_utilisateur
```

### Echantillon du résultat :

```
hdfs dfs -cat /user/maria_dev/output_par_utilisateur/part-00000 | head -n 15

"100001"        9
"100016"        50
"100028"        4
"100029"        1
"100033"        1
"100046"        133
"100051"        19
"100058"        5
"100065"        2
"100068"        19
"100076"        4
"100085"        3
"100087"        8
"100088"        13
"100091"        29
```

Même logique que la question 1, mais la clé émise par le mapper est `userId` au lieu de `movieId`. Soit 14 592 utilisateurs distincts ayant posé au moins un tag.

---

## 3. Combien de blocs le fichier occupe-t-il dans HDFS dans chacune des configurations ?

Pour déterminer le nombre de blocs, on utilise la commande `hdfs fsck`.

### Avec bloc de 128 Mo :

```
hdfs fsck /user/maria_dev/datasets/tags.csv -files -blocks


Status: HEALTHY
 Total size:    38810332 B
 Total dirs:    0
 Total files:   1
 Total symlinks:                0
 Total blocks (validated):      1 (avg. block size 38810332 B)
 Minimally replicated blocks:   1 (100.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       0 (0.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    1
 Average block replication:     1.0
 Corrupt blocks:                0
 Missing replicas:              0 (0.0 %)
 Number of data-nodes:          1
 Number of racks:               1
FSCK ended at Thu Apr 09 14:20:41 UTC 2026 in 23 milliseconds
```

Le fichier fait environ 38 Mo (38 810 332 bytes). Avec une taille de bloc par défaut de 128 Mo, il est logique qu'un seul bloc soit utilisé pour stocker le fichier dans HDFS, puisque celui-ci est inférieur à la taille d'un bloc.

### Avec bloc de 64 Mo :

```
hdfs dfs -rm -f /user/maria_dev/datasets/tags.csv
hdfs dfs -D dfs.block.size=67108864 -put ml-25m/tags.csv /user/maria_dev/datasets/
hdfs fsck /user/maria_dev/datasets/tags.csv -files -blocks


Status: HEALTHY
 Total size:    38810332 B
 Total dirs:    0
 Total files:   1
 Total symlinks:                0
 Total blocks (validated):      1 (avg. block size 38810332 B)
 Minimally replicated blocks:   1 (100.0 %)
 Over-replicated blocks:        0 (0.0 %)
 Under-replicated blocks:       0 (0.0 %)
 Mis-replicated blocks:         0 (0.0 %)
 Default replication factor:    1
 Average block replication:     1.0
 Corrupt blocks:                0
 Missing replicas:              0 (0.0 %)
 Number of data-nodes:          1
 Number of racks:               1
FSCK ended at Thu Apr 09 14:21:24 UTC 2026 in 5 milliseconds
```

Avec une taille de bloc de 64 Mo, le fichier occupant environ 38 Mo reste inférieur à la taille d'un seul bloc, ce qui explique encore une fois qu'un seul bloc suffit pour le stocker dans HDFS.

---

## 4. Combien de fois chaque tag a-t-il été utilisé pour taguer un film ?

### Script Python : `comptage_par_tag.py`

```
from mrjob.job import MRJob
from mrjob.step import MRStep

class ComptageParTag(MRJob):
    def steps(self):
        return [MRStep(mapper=self.mapper_get_tags, reducer=self.reducer_count_tags)]

    def mapper_get_tags(self, _, line):
        try:
            (userId, movieId, tag, timestamp) = line.split(',', 3)
            if userId != 'userId':
                yield tag.strip(), 1
        except Exception:
            pass

    def reducer_count_tags(self, tag, counts):
        yield tag, sum(counts)

if __name__ == '__main__':
    ComptageParTag.run()
```

### Commande Hadoop :

```
#Créer fichier python
nano comptage_par_tag.py
#Exécuter python
python comptage_par_tag.py -r hadoop --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar hdfs:///user/maria_dev/datasets/tags.csv -o hdfs:///user/maria_dev/output_par_tag
```

### Echantillon du résultat :

```
hdfs dfs -cat /user/maria_dev/output_par_tag/part-00000 | head -n 15

"!950's Superman TV show"       1
""      9
"#1 prediction" 3
"#Danish"       2
"#MeToo"        1
"#TimesUp"      1
"#Vatican City" 1
"#adventure"    1
"#antichrist"   1
"#boring #Lukeiamyourfather"    1
"#boring"       1
"#documentary"  1
"#entertaining" 1
"#exorcism"     1
"#fantasy"      2
"#hanks #muchstories"   1
```

Le mapper émet `(tag, 1)` pour chaque ligne valide. Le reducer somme les compteurs par tag. On dénombre 72 959 tags distincts.

---

## 5. Pour chaque film, combien de tags le même utilisateur a-t-il introduits ?

### Script Python : `tags_par_film_utilisateur.py`

```
from mrjob.job import MRJob
from mrjob.step import MRStep

class TagsParFilmUtilisateur(MRJob):
    def steps(self):
        return [MRStep(mapper=self.mapper_get_tags, reducer=self.reducer_count_tags)]

    def mapper_get_tags(self, _, line):
        try:
            (userId, movieId, tag, timestamp) = line.split(',', 3)
            if userId != 'userId':
                yield (movieId + "," + userId), 1
        except Exception:
            pass

    def reducer_count_tags(self, key, counts):
        yield key, sum(counts)

if __name__ == '__main__':
    TagsParFilmUtilisateur.run()
```

### Commande Hadoop :

```
#Créer fichier python
nano tags_par_film_utilisateur.py
#Exécuter python
python tags_par_film_utilisateur.py -r hadoop --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar hdfs:///user/maria_dev/datasets/tags.csv -o hdfs:///user/maria_dev/output_par_film_utilisateur
```

### Echantillon du résultat :

```
hdfs dfs -cat /user/maria_dev/output_par_film_utilisateur/part-00000 | head -n 15

"1,100538"      4
"1,10231"       2
"1,102568"      4
"1,102901"      1
"1,103368"      1
"1,103371"      1
"1,103883"      3
"1,104394"      9
"1,1048"        1
"1,105717"      1
"1,105809"      5
"1,107432"      2
"1,109146"      2
"1,109258"      1
"1,110339"      3
```

La clé composite `"movieId,userId"` est émise comme une chaîne unique par le mapper. Le reducer agrège les compteurs pour chaque paire (film, utilisateur). On obtient 305 356 paires distinctes.

---


