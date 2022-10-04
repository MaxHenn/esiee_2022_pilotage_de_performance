# ESIEE - 2022 - Pilotage de performance

Ce projet a été testé sur une machine virtuel **Ubuntu 22.04** sous **VirtualBox**.

Le but de celui-ci est de **déployer**, **monitorer** et **tester** plusieurs applications sur un serveur.

L'application de démonstration utilisé est disponible ici : [github - mybatis-spring-boot-jpetstore](https://github.com/kazuki43zoo/mybatis-spring-boot-jpetstore).

## Sommaire

- [1 - Pré-requis](#pre-requis)
    - [1.1 - Mise à jour et outils utils]()
    - [1.2 - Net Tools]()
    - [1.3 - JDK 11]()
    - [1.4 - Configuration réseau]()
- [2 - TP - 1 - Installation d'une application Java JEE]()
    - [2.1 - Clone & Run]()
    - [2.2 - Accés via *localhost* et *`*ip*]()
- [3 - TP - 2 - Configuration de 2 JPetStore avec LoadBalancer]()
    - [3.1 - Configuration de 2 JPetStore]()
    - [3.2 - Préparation d'un LoadBalancer avec Apache]()
        - [3.2.1 - Installation de Apache]()
        - [3.2.2 - Configuration de Apache]()
    - [3.3 - Lancement de plusieurs applications]()
    - [3.4 - Finis - Je check 😉]()
    - [3.5 - Lecture des logs de apache]()
    - [3.6 - Lecture des logs des applications]()
    - [3.7 - Accessibilité](#accessibilite)
- [4 - TP - 2 - ]()

## 1 - Pré-requis <a name="pre-requis"></a>

### 1.1 - Mise à jour et outils utils

```
sudo apt update && apt upgrade
sudo apt install tree htop
```

### 1.2 - Net Tools

```
sudo apt install net-tools
```

Permet de faire plein de chose, comme `ifconfig` 😉

![img](_img/001.png)

### 1.3 - JDK 11 

```
sudo apt install openjdk-11-jre-headless
```

Vérification de JDK

```
> java --version
openjdk 11.0.16 2022-07-19
OpenJDK Runtime Environment (build 11.0.16+8-post-Ubuntu-0ubuntu122.04)
OpenJDK 64-Bit Server VM (build 11.0.16+8-post-Ubuntu-0ubuntu122.04, mixed mode, sharing)
```

### 1.4 - Configuration réseau (Si Ubuntu en VM)

![img](_img/003.png)

Sélectionné le nom de la carte réseau principal de la machine utilisant VirtualBox (ou VMWare ... ou ce que tu veux 😉 )

## 2 - TP - 1 - Installation d'une application Java JEE

### 2.1 - Clone & Run

Cloner le projet jpetstore stocker sur git :

```
git clone https://github.com/kazuki43zoo/mybatis-spring-boot-jpetstore.git
```

Déplacer dans le dossier :

```
cd mybatis-spring-boot-jpetstore
```

Démarrage du projet avec Maven :

```
./mvnw clean spring-boot:run
```

### 2.2 - Accés via `localhost` et `ip`

Accès par : 

- [http://locahost:8080/](http://locahost:8080/)
- [http://172.16.202.226:8080/](http://172.16.202.226:8080/)

Changer le port pour `8081` :

```
nano src/main/resources/application.properties
```

![img](_img/002.png)

Accès par : 

- [http://locahost:8081/](http://locahost:8081/)
- [http://172.16.202.226:8081/](http://172.16.202.226:8081/)

## 3 - TP - 2 - Configuration de 2 JPetStore avec LoadBalancer

### 3.1 - Configuration de 2 JPetStore

1. Avoir 2 instances de JPetStore
2. Changé les ports de chaque applications pour :
    - JPetStore_1 : `8081`
    - JPetStore_2 : `8082`

Duplication de **jpetstore** vers **jpetstore_1** et **jpetstore_2**.

```
mkdir apps
cp -r mybatis-spring-boot-jpetstore/ apps/jpetstore_1
cp -r mybatis-spring-boot-jpetstore/ apps/jpetstore_2
```

Vérification de la création :

```
ls -ali apps/

total 16
1476918 drwxrwxr-x  4 ldumay ldumay 4096 oct.   3 16:15 .
1327599 drwxr-x--- 22 ldumay ldumay 4096 oct.   3 16:13 ..
1477139 drwxrwxr-x  8 ldumay ldumay 4096 oct.   3 16:15 jpetstore_1
1477158 drwxrwxr-x  8 ldumay ldumay 4096 oct.   3 16:15 jpetstore_2
```

Modification des fichiers `application.properties` por **jpetstore_1** et **jpetstore_2**.

```
nano apps/jpetstore_1/src/main/resources/application.properties
nano apps/jpetstore_2/src/main/resources/application.properties
```

Modifier la configuration du datasource Spring :

```
spring.datasource.url=jdbc:hsqldb:file:~/db/jpetstore;hsqldb.lock_file=false
```

Compiler chaque projet en jar :

```
cd apps/jpetstore_1/
./mvnw clean package -DskipTests=true
```

> Ramplacer `jpetstore_1` par le dossier de l'application cible.

### 3.2 - Préparation d'un LoadBalancer avec Apache

#### 3.2.1 - Installation de Apache

Installer Apache :

```
sudo apt install apache2
sudo systemctl enable apache2
sudo systemctl start apache2.service
sudo systemctl status apache2.service
```

Activer le service de Proxy :

```
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod headers
sudo a2enmod lbmethod_byrequests
sudo systemctl restart apache2.service
```

#### 3.2.2 - Configuration de Apache

Récupérer l'ip de la machine :

```
> ifconfig

enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.202.151  netmask 255.255.255.0  broadcast 172.16.202.255
        inet6 fe80::2dfa:f1ba:cfab:a207  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:c4:7d:a5  txqueuelen 1000  (Ethernet)
        RX packets 544  bytes 319003 (319.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 246  bytes 32572 (32.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Créer et éditer un fichier de configuration de **vhost** pour les application jpetstore que l'on appellera `jpetstore.conf`.

```
sudo nano /etc/apache2/sites-available/jpetstore.conf
```

Ci-dessous, le contenu du ficher `jpetstore.conf`.

```xml
<VirtualHost *:80>
    Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED
    ProxyRequests Off
    ProxyPreserveHost On

    <Proxy "balancer://mycluster">
        BalancerMember "http://172.16.202.151:8081" route=1
                #attention: il faut changer les IPs et vérifier les ports
        BalancerMember "http://172.16.202.151:8082" route=2
        ProxySet stickysession=ROUTEID
    </Proxy>

    ProxyPass "/" "balancer://mycluster/"
    ProxyPassReverse "/" "balancer://mycluster/"

    ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Vérifier la bonne écriture et le contenu du fichier avec :

```
cat /etc/apache2/sites-available/jpetstore.conf
```

Désactiver le site par défaut d'apache.

```
sudo nano /etc/apache2/apache2.conf
```

> Avant :
> 
> ![img](_img/005.png)

> Après
> 
> ![img](_img/006.png)

Désactiver la configuration par défaut de apache :

```
sudo a2dissite
Your choices are: 000-default jpetstore
Which site(s) do you want to disable (wildcards ok)?
000-default
Site 000-default disabled.
To activate the new configuration, you need to run:
  systemctl reload apache2
```
> ▶ `000-default`

Activer la configuration `jpetstore.conf`. 

```
sudo a2ensite

Your choices are: 000-default default-ssl jpetstore
Which site(s) do you want to enable (wildcards ok)?
jpetstore
Enabling site jpetstore.
To activate the new configuration, you need to run:
  systemctl reload apache2
```

> ▶ `jpetstore`

Redémarrer apache.

```
systemctl reload apache2
```


### 3.3 - Lancement de plusieurs applications

Préparer les fichiers de logs des applications.

```
mkdir apps/logs/
touch apps/logs/jpetstore_1.logs
touch apps/logs/jpetstore_2.logs
tree apps/logs/
```

Résultat : 

```
apps/logs/
├── jpetstore_1.logs
└── jpetstore_2.logs

0 directories, 2 files
```

Lancer chaque applications indépendemment.

Effecuté la commande ci-dessous pour lancer une 1ère application indépendante.

```
nohup java -jar apps/jpetstore_1/target/mybatis-spring-boot-jpetstore-2.0.0-SNAPSHOT.jar > apps/logs/jpetstore_1.logs &
```

Résultat :

```
[1] 2444
ldumay@ldumay-vm:~$ nohup: entrée ignorée et sortie d'erreur standard redirigée vers la sortie standard
```

L'application est lancé et les logs de celle-ci sont enregistré dans le fichier `jpetstore_1.logs`. Faite ensuite `CTRL` + `C` pour reprendre la main sur la console. Bien sûr, le nouveau processus `[1] 2444` n'est pas arreté.

Résultat :

```
^C
ldumay@ldumay-vm:~$
```

Refaite la même chose pour la 2e applications.

> Je sympas, voilà commande pour jpetstore_2 :
> 
> ```
> nohup java -jar apps/jpetstore_2/target/mybatis-spring-boot-jpetstore-2.0.0-SNAPSHOT.jar > apps/logs/jpetstore_2.logs &
> ```

### 3.4 - Finis - Je check 😉

Normalement, si tout est **OK**, il devrais avoir 2 instance java actifs. Pour vérifier, faite la commande `top`. Celle-ci ouvre le monteur d'acivité en console. Pour le fermer, faite `CTRL`+ `C`.

![img](_img/004.png)

> Sur la capture, les 2 applications java sont d'ids **2444** et **3538**.

### 3.5 - Lecture des logs de apache

Pour lire les logs de apache.

```
cat /var/log/apache2/error.log
cat /var/log/apache2/access.log
```

### 3.6 - Lecture des logs des applications

Pour lire les logs de chaque applications en temps réel, faite : 

```
tail -f apps/logs/jpetstore_1.logs

OU

tail -f apps/logs/jpetstore_2.logs
```

Pour le fermer, faite `CTRL`+ `C`.

### 3.7 - Accessibilité <a name="accessibilite"></a>

Le service est donc acessible à l'adresse du serveur, ici [http://172.16.202.151](http://172.16.202.151), qui va lui même se charger de redirriger vers l'appplication **jpetstore_1 / port:8081** ou **jpetstore_2 / port:8081**.

## 4 - TP - 2 - 