# TP_ESE_BusEtResaux

Compte rendu de Nolan Jacquot et Gabriel Guiffault
## TP 1 - Bus I2C

Objectif: Interfacer un STM32 avec des capteurs I²C



La première étape est de mettre en place la communication entre le microcontrôleur et les capteurs (température, pression, accéléromètre...) via  le bus I²C.
Le capteur comporte 2 composants I²C, qui partagent le même bus. Le STM32 jouera le rôle de Master sur le bus.Le code du STM32 sera écrit en langage C, en utilisant la bibliothèque HAL.

### 2.1. Capteur BMP280 

Mise en œuvre du BMP280
À partir de la datasheet du BMP280, nous identifions les éléments suivants:

1. Les adresses I²C possibles pour ce composant:
   - Le BMP280 a deux adresses I²C possibles selon l'état du pin SDO : 0x76 si SDO est connecté à GND, et 0x77 si SDO est connecté à VDDIO. Cela se trouve à la page 29

2. Le registre et la valeur permettant d'identifier ce composant :
   - Le registre "id" est 0xD0, et la valeur d'identification est 0x58. Cela se trouve à la page 24

3. Le registre et la valeur permettant de placer le composant en mode normal :
   - Le registre de contrôle est 0xF4. Les bits [1:0] dans ce registre contrôlent le mode d'alimentation, où '11' active le mode normal. Cela se trouve à la page 16

4. Les registres contenant l’étalonnage du composant:
   - Les coefficients d'étalonnage sont stockés dans les registres allant de 0x88 à 0xA1. Cela se trouve à la page 21.

5. Les registres contenant la température (ainsi que le format) :
   - Les registres de température sont 0xFA (MSB), 0xFB (LSB), et 0xFC (XLSB). La température est stockée sur 20 bits. Cela se trouve à la page 27.

6. Les registres contenant la pression (ainsi que le format) :
   - Les registres de pression sont 0xF7 (MSB), 0xF8 (LSB), et 0xF9 (XLSB). La pression est également stockée sur 20 bits. Cela se trouve à la page 26.

7. Les fonctions permettant le calcul de la température et de la pression compensées en format entier 32 bits:
   - Les formules de compensation pour la température et la pression sont données à la page 
22. Elles nécessitent l'utilisation des coefficients d'étalonnage et sont basées sur un algorithme en virgule fixe sur 32 bits.

### 2.2. Setup du STM32

Configuration du STM32

Pour ce TP, nous avons besoin des connections suivantes:
Une liaison I²C. On utilisera les broches compatibles avec l'empreinte arduino (broches PB8 et PB9) 
Une UART sur USB (UART2 sur les broches PA2 et PA3) 

Test de la chaîne de compilation et communication UART sur USB avec un programme echo
EXEMPLE CODE:
```C
while (1) {
   uint8_t data;
       /* Lire un caractère depuis l'UART */
   if (HAL_UART_Receive(&huart2, &data, 1, HAL_MAX_DELAY) == HAL_OK) { 
         /* Envoyer le caractère recu (echo) */ 
      HAL_UART_Transmit(&huart2, &data, 1, HAL_MAX_DELAY);
         /* Envoyer le caractère avec le printf */ 
      printf("Received: %s\r\n", data);
      /*Attention %c risque de ne pas marcher car si un seul caractere le buffer n'est jamais rempli*/
       }
    }
 }
```
### 2.3. Communication I²C

Primitives I²C sous STM32_HAL

Communication avec le BMP280

Identification du BMP280

L'identification du BMP280 consiste en la lecture du registre ID

En I²C, la lecture se déroule de la manière suivante:
>[!Warning]
>envoyer l'adresse du registre ID puis recevoir 1 octet correspondant au contenu du registre
>```C
>#define BMP280_I2C_ADDRESS (0x77 << 1)
>```
>L'adresse est indiqué directement sur la seriographie attention au decalage de 1 bit!!

```C
uint8_t bmp280_read_id(void)
{
   uint8_t reg = 0xD0;
   uint8_t id = 0;
   HAL_I2C_Master_Transmit(&hi2c1, BMP280_I2C_ADDRESS, &reg, 1, HAL_MAX_DELAY);
   HAL_I2C_Master_Receive(&hi2c1, BMP280_I2C_ADDRESS, &id, 1, HAL_MAX_DELAY);
   return id;
}

```
>[!Tip]
> Le registre "id" est 0xD0, et la valeur d'identification est 0x58.Il est preferable d'utiliser check ID plutot que read ID
>```C
> uint8_t id = bmp280_read_id();
>   if (id == 0x58)
>    {
>        printf("BMP280 detected! ID: 0x%02X\r\n", id);
>    }
>    else
>    {
>        printf("Failed to detect BMP280. ID: 0x%02X\r\n", id);
>    }
>```
>Nous avons rajouté ce code pour verifier si l'ID étais bien presente.

### Configuration du BMP280

Avant de pouvoir faire une mesure, il faut configurer le BMP280.

Pour commencer, nous allons utiliser la configuration suivante: mode normal, Pressure oversampling x16, Temperature oversampling x2

En I²C, l'écriture dans un registre se déroule de la manière suivante:

envoyer l'adresse du registre à écrire, suivi de la valeur du registre

si on reçoit immédiatement, la valeur reçu sera la nouvelle valeur du registre

Voici le code pour écrire la configuration dans le registre.
```C
void bmp280_configure(void)
{
   uint8_t config[2];
   config[0] = 0xF4;/*adresse du registre de configuration*/
   config[1] = 0x27;/*0010 0111*/
   HAL_I2C_Master_Transmit(&hi2c1, BMP280_I2C_ADDRESS, config, 2, HAL_MAX_DELAY);
}
```
Explication de la valeur 0xF4

Le registre de contrôle du BMP280 (adresse 0xF4) est utilisé pour configurer le mode de fonctionnement du capteur et les niveaux d'oversampling pour la pression et la température. Voir page 16 de la doc du composant

Explication de la valeur 0x27

Le code 0x27 (en binaire 0010 0111) configure les bits comme suit :

Bits 7-5 : osrs_t (Oversampling de la température),ici osrs_t = 001 signifie oversampling x2 pour la température.

Bits 4-2 : osrs_p (Oversampling de la pression), ici osrs_p = 101 signifie oversampling x16 pour la pression.

Bits 1-0 : mode (Mode de fonctionnement) Ces deux bits définissent le mode de fonctionnement du BMP280.
Dans le code config[1] = 0x27, les bits 1-0 sont définis comme 11, ce qui signifie mode normal.

Dans le while(1) : 
```C
   if (id == 0x58)
    {
        printf("BMP280 detected! ID: 0x%02X\r\n", id);
        bmp280_configure();
        printf("BMP280 configured!\r\n");
    }
```
Récupération de l'étalonnage, de la température et de la pression

Nous récupérons en une fois le contenu des registres qui contiennent l'étalonnage du BMP280.

Dans la boucle infinie du STM32, récupérez les valeurs de la température et de la pression. Envoyez sur le port série la valeurs 32 bit non compensées de la pression de la température. 

```C
void bmp280_read_calibration(BMP280_CalibData *calib)
{
   uint8_t calib_data[24];
   uint8_t reg = 0x88;
   HAL_I2C_Master_Transmit(&hi2c1, BMP280_I2C_ADDRESS, &reg, 1, HAL_MAX_DELAY);
   HAL_I2C_Master_Receive(&hi2c1, BMP280_I2C_ADDRESS, calib_data, 24, HAL_MAX_DELAY);
   calib->dig_T1 = (uint16_t)((calib_data[1] << 8) | calib_data[0]);
   calib->dig_T2 = (int16_t)((calib_data[3] << 8) | calib_data[2]);
   calib->dig_T3 = (int16_t)((calib_data[5] << 8) | calib_data[4]);
   calib->dig_P1 = (uint16_t)((calib_data[7] << 8) | calib_data[6]);
   calib->dig_P2 = (int16_t)((calib_data[9] << 8) | calib_data[8]);
   calib->dig_P3 = (int16_t)((calib_data[11] << 8) | calib_data[10]);
   calib->dig_P4 = (int16_t)((calib_data[13] << 8) | calib_data[12]);
   calib->dig_P5 = (int16_t)((calib_data[15] << 8) | calib_data[14]);
   calib->dig_P6 = (int16_t)((calib_data[17] << 8) | calib_data[16]);
   calib->dig_P7 = (int16_t)((calib_data[19] << 8) | calib_data[18]);
   calib->dig_P8 = (int16_t)((calib_data[21] << 8) | calib_data[20]);
   calib->dig_P9 = (int16_t)((calib_data[23] << 8) | calib_data[22]);
}
```
Apparemment c'est la seule méthode de le faire
```C
void bmp280_read_raw_data(int32_t *temperature, int32_t *pressure)
{
   uint8_t data[6];
   uint8_t reg = 0xF7;
   HAL_I2C_Master_Transmit(&hi2c1, BMP280_I2C_ADDRESS, &reg, 1, HAL_MAX_DELAY);
   HAL_I2C_Master_Receive(&hi2c1, BMP280_I2C_ADDRESS, data, 6, HAL_MAX_DELAY);
   // La pression brute
   *pressure = (int32_t)(((data[0] << 16) | (data[1] << 8) | data[2]) >> 4);
   // La température brute
   *temperature = (int32_t)(((data[3] << 16) | (data[4] << 8) | data[5]) >> 4);
}
```
### Calcul des températures et des pression compensées

Il faut retrouvez dans la datasheet du STM32 le code permettant de compenser la température et la pression à l'aide des valeurs de l'étalonnage au format entier 32 bits (on utilisera pas les flottants pour des problèmes de performance).

Nous transmettons sur le port série les valeurs compensés de température et de pression sous un format lisible.

Nous nous basons sur le code d'exemple si dessous fourni par la documentation du composant (page 22).
```C
// Returns temperature in DegC, resolution is 0.01 DegC. Output value of “5123” equals 51.23 DegC.
// t_fine carries fine temperature as global value
BMP280_S32_t t_fine;
BMP280_S32_t bmp280_compensate_T_int32(BMP280_S32_t adc_T)
{
BMP280_S32_t var1, var2, T;
var1 = ((((adc_T>>3) – ((BMP280_S32_t)dig_T1<<1))) * ((BMP280_S32_t)dig_T2)) >> 11;
var2 = (((((adc_T>>4) – ((BMP280_S32_t)dig_T1)) * ((adc_T>>4) – ((BMP280_S32_t)dig_T1))) >> 12) *
((BMP280_S32_t)dig_T3)) >> 14;
t_fine = var1 + var2;
T = (t_fine * 5 + 128) >> 8;
return T;
}
```
```C
// Returns pressure in Pa as unsigned 32 bit integer in Q24.8 format (24 integer bits and 8 fractional bits).
// Output value of “24674867” represents 24674867/256 = 96386.2 Pa = 963.862 hPa
BMP280_U32_t bmp280_compensate_P_int64(BMP280_S32_t adc_P)
{
BMP280_S64_t var1, var2, p;
var1 = ((BMP280_S64_t)t_fine) – 128000;
var2 = var1 * var1 * (BMP280_S64_t)dig_P6;
var2 = var2 + ((var1*(BMP280_S64_t)dig_P5)<<17);
var2 = var2 + (((BMP280_S64_t)dig_P4)<<35);
var1 = ((var1 * var1 * (BMP280_S64_t)dig_P3)>>8) + ((var1 * (BMP280_S64_t)dig_P2)<<12);
var1 = (((((BMP280_S64_t)1)<<47)+var1))*((BMP280_S64_t)dig_P1)>>33;
if (var1 == 0)
{
return 0; // avoid exception caused by division by zero
}
p = 1048576-adc_P;
p = (((p<<31)-var2)*3125)/var1;
var1 = (((BMP280_S64_t)dig_P9) * (p>>13) * (p>>13)) >> 25;
var2 = (((BMP280_S64_t)dig_P8) * p) >> 19;
p = ((p + var1 + var2) >> 8) + (((BMP280_S64_t)dig_P7)<<4);
return (BMP280_U32_t)p;
```

Autre code d'exemple fourni par la documentation du composant (page 45).

```C
// Returns temperature in DegC, double precision. Output value of “51.23” equals 51.23 DegC.
// t_fine carries fine temperature as global value
BMP280_S32_t t_fine;
double bmp280_compensate_T_double(BMP280_S32_t adc_T)
{
double var1, var2, T;
var1 = (((double)adc_T)/16384.0 – ((double)dig_T1)/1024.0) * ((double)dig_T2);
var2 = ((((double)adc_T)/131072.0 – ((double)dig_T1)/8192.0) *
(((double)adc_T)/131072.0 – ((double) dig_T1)/8192.0)) * ((double)dig_T3);
t_fine = (BMP280_S32_t)(var1 + var2);
T = (var1 + var2) / 5120.0;
return T;
}
```
```C
// Returns pressure in Pa as double. Output value of “96386.2” equals 96386.2 Pa = 963.862 hPa
double bmp280_compensate_P_double(BMP280_S32_t adc_P)
{
double var1, var2, p;
var1 = ((double)t_fine/2.0) – 64000.0;
var2 = var1 * var1 * ((double)dig_P6) / 32768.0;
var2 = var2 + var1 * ((double)dig_P5) * 2.0;
var2 = (var2/4.0)+(((double)dig_P4) * 65536.0);
var1 = (((double)dig_P3) * var1 * var1 / 524288.0 + ((double)dig_P2) * var1) / 524288.0;
var1 = (1.0 + var1 / 32768.0)*((double)dig_P1);
if (var1 == 0.0)
{
return 0; // avoid exception caused by division by zero
}
p = 1048576.0 – (double)adc_P;
p = (p – (var2 / 4096.0)) * 6250.0 / var1;
var1 = ((double)dig_P9) * p * p / 2147483648.0;
var2 = p * ((double)dig_P8) / 32768.0;
p = p + (var1 + var2 + ((double)dig_P7)) / 16.0;
return p;
}
```
Ci dessous le code final que nous avons fait fonctionner.
```C
int32_t bmp280_compensate_T(int32_t adc_T, BMP280_CalibData *calib, int32_t *t_fine)
{
   int32_t var1, var2, T;
   var1 = ((((adc_T >> 3) - ((int32_t)calib->dig_T1 << 1))) * ((int32_t)calib->dig_T2)) >> 11;
   var2 = (((((adc_T >> 4) - ((int32_t)calib->dig_T1)) * ((adc_T >> 4) - ((int32_t)calib->dig_T1))) >> 12) * ((int32_t)calib->dig_T3)) >> 14;
   *t_fine = var1 + var2;  // Mise à jour de t_fine via le pointeur
   T = (*t_fine * 5 + 128) >> 8;
   return T;
}
```
Ce code retourne la temperature compensé en degré
```C
uint32_t bmp280_compensate_P(int32_t adc_P, BMP280_CalibData *calib, int32_t t_fine)
{
   int64_t var1, var2, p;
   var1 = ((int64_t)t_fine) - 128000;
   var2 = var1 * var1 * (int64_t)calib->dig_P6;
   var2 = var2 + ((var1 * (int64_t)calib->dig_P5) << 17);
   var2 = var2 + (((int64_t)calib->dig_P4) << 35);
   var1 = ((var1 * var1 * (int64_t)calib->dig_P3) >> 8) + ((var1 * (int64_t)calib->dig_P2) << 12);
   var1 = (((((int64_t)1) << 47) + var1)) * (int64_t)calib->dig_P1 >> 33;
   if (var1 == 0) {
       return 0;  // Éviter une division par zéro
   }
   p = 1048576 - adc_P;
   p = (((p << 31) - var2) * 3125) / var1;
   var1 = (((int64_t)calib->dig_P9) * (p >> 13) * (p >> 13)) >> 25;
   var2 = (((int64_t)calib->dig_P8) * p) >> 19;
   p = ((p + var1 + var2) >> 8) + (((int64_t)calib->dig_P7) << 4);
   return (uint32_t)p;  // Pression en Pa au format Q24.8
}
```
Ce code retourne la pression compensé en Pascal

Dans le main on envoie regulierement des actualisation des mesures
```C
   BMP280_CalibData calib_data;
   int32_t temperature_raw, pressure_raw;
   int32_t temperature_comp, t_fine;
   uint32_t pressure_comp;
   bmp280_read_calibration(&calib_data);

 while (1)
 {
   /* USER CODE END WHILE */
   /* USER CODE BEGIN 3 */
     bmp280_read_raw_data(&temperature_raw, &pressure_raw);
     temperature_comp = bmp280_compensate_T(temperature_raw, &calib_data, &t_fine);
     pressure_comp = bmp280_compensate_P(pressure_raw, &calib_data, t_fine);
     printf("Temperature compensee: %ld.%02ld C\r\n", temperature_comp / 100, temperature_comp % 100);
     printf("Pression compensee: %lu Pa\r\n", pressure_comp / 256);
     HAL_Delay(1000);  // Délai de 1 seconde
 }
 /* USER CODE END 3 */
}
```

## TP2 - Interfaçage STM32 - Raspberry

### 3.1. Mise en route du Raspberry PI Zéro

>[!Note]
>Configuration réseau du routeur utilisé en TP :
>SSID : ESE_Bus_Network  Password : ilovelinux
>
>ou
>
>SSID : D060-2Ghz
>
>Password : ilovelinux
>
>Identifiant: nolan
>
>Mot de passe SSH: ESE2021
>
>Pour activer le port série sur connecteur GPIO, sur la partition boot, nous modifions le fichier config.txt en utilisant sudo nano pour editer le fichier avec les ligne ci dessous:
>```
>enable_uart=1
>dtoverlay=disable-bt
>```
>Pour  dans le fichier cmdline.txt, pour que le noyau libère le port UART, retirez l'option suivante: 
>```
>console=serial0,115200
>```

>[!Warning]
>ne pas oublier de reboot pour valider les changement

### 3.2. Port Série Loopback

Branchez le port série du Raspberry en boucle: RX sur TX.

Utilisez le logiciel minicom sur le raspberry pour tester le port série.

minicom -D /dev/ttyAMA0 

Une fois dans minicom configurer le port série en pressant CTRL+A suivi de O. Pensez à déactiver le contrôle de flux matériel (on utilise pas les lignes RTS/CTS).
Écrire quelques lettres au clavier. Si elles s'affichent, le loopback fonctionne (essayez en le débranchant).
CTRL+A Q pour quitter minicom.

Communication avec la STM32

Nous modifions la fonction printf pour quelle affiche sur les 2 ports série en même temps. 

Le protocole de communication entre le Raspberry et la STM32 est le suivant( Requête du RPi, Réponse du STM, Commentaire):

GET_T 	T=+12.50_C 	   Température compensée sur 10 caractères

GET_P 	P=102300Pa 	   Pression compensée sur 10 caractères

SET_K=1234 	SET_K=OK 	Fixe le coefficient K (en 1/100e)

GET_K 	K=12.34000 	   Coefficient K sur 10 caractères

GET_A 	A=125.7000     Angle sur 10 caractères

## TP3 - Interface REST

### 4.1. Installation du serveur Python

Installation
```py
sudo apt update
sudo apt install pipenv
pipenv --python 3
pipenv install - p requirements.txt
pip shell
```
Nous rajoutons un nouvel utilisateur différent de pi, avec les droits de sudo et  d'accès au port série (dialout):
```
sudo adduser gabriel
//mdp:torero2021
sudo usermod -aG sudo gabriel
sudo usermod -aG dialout gabriel
```
 Nous logguons en tant que gabriel
 
Il faut  installez pip pour python3 sur le Raspberry:
```
sudo apt update
sudo apt install python3-pip
```
Nous créeons un répertoire pour le développement de votre serveur avec un fichier nommé requirement.txt:
```
pyserial
flask
```
Il faut installer ces bibliothèques par la commande: 
```
pip3 install -r requirement.txt
```
À nouveau, déloggue, puis reloggue en tant que gabriel pour mettre à jour le PATH et permettre de lancer flask.

### Premier fichier Web

Nous créeons un fichier hello.py avec le code suivant:
```py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!\n'
```
Nous lançons le nouveau serveur web avec:
```py
pi@raspberrypi:~/server $ FLASK_APP=hello.py flask run
```
<p align="center"> <img src="preuve serveur.PNG" width="100%" height="auto" /> </p>
test du serveur avec la commande curl (dans un 2e terminal):
```py
pi@raspberrypi:~/server $ curl http://127.0.0.1:5000 -s -D -
```
Les options -s -D - permettent de visualiser les headers de la réponse HTTP en particulier le champ Server

Le serveur fonctionne sur la loopback. Cela est résolue avec:
```py
pi@raspberrypi:~/server $ FLASK_APP=hello.py FLASK_ENV=development flask run --host 0.0.0.0
```
La constante FLASK_ENV=development permet de lancer un mode debug. À partir de maintenant, vous pouvez tester votre serveur web avec un navigateur.

### 4.2. Première page REST

Première route

Ajoutez les lignes suivantes au fichier hello.py:
```py
welcome = "Welcome to 3ESE API!"

@app.route('/api/welcome/')
def api_welcome():
    return welcome
    
@app.route('/api/welcome/<int:index>')
def api_welcome_index(index):
    return welcome[index]
```
<p align="center"> <img src="Preuve nouvelle version.PNG" width="100%" height="auto" /> </p>
Quel est le rôle du décorateur @app.route?

Quel est le role du fragment <int:index>?

Pour pouvoir prétendre être RESTful, votre serveur va devoir:

répondre sous forme JSON.

différencier les méthodes HTTP

C’est ce que nous allons voir maintenant.


Première page REST
Réponse JSON

Un module JSON est disponible dans la librairie standard de python: https://docs.python.org/3/library/json.html Le plus simple pour générer du JSON est d’utiliser la fonction json.dumps() sur un objet Python. Vous pouvez par exemple remplacer la dernière ligne de la fonction api_welcome_index par:
```py
return json.dumps({"index": index, "val": welcome[index]})
```
(oubliez pas le import json en début de fichier!)

Testez le résultat. Est-ce suffisant pour dire que la réponse est bien du JSON? Observez en particulier les entêtes de la réponse: sous Firefox ou Chrome ouvrez les outils de développement (F12), selectionnez l’onglet “réseau” et rechargez la page. Vous pouvez normalement trouver l’entête de réponse Content-Type: ce n’est pas du JSON!
1re solution

Il faut modifier la réponse renvoyée par flask, en ajoutant au contenu du return des entêtes personnalisés sous forme d’un dictionnaire:
```py
return json.dumps({"index": index, "val": welcome[index]}), {"Content-Type": "application/json"}
```
À partir de maintenant la réponse est bien du JSON, et Firefox vous présente le résultat de manière différente (Chrome aussi, mais c’est moins visible).
2e solution

L’utilisation de json avec flask étant très fréquente, une fonction jsonify() existe dans la bibliothèque. Elle est accessible après un from flask import jsonify. Cette fonction gère à la fois la conversion en json et l’ajout de l’entête.

Modifiez votre code pour utiliser jsonify et testez le.
Erreur 404

Il arrive souvent que les URL demandées soient fausses, il faut donc que votre serveur renvoie une erreur 404.

Téléchargez le fichiers page_not_found.html (en ressource) et placez le dans un nouveau répertoire templates (nom de chemin imposé par flask). Le plus simple pour créer ce fichier est de créer un fichier vide, puis de copier-coller son contenu (<shift>+<insert> sous windows). Une autre solution est d'utiliser un utilitaire de copie sur ssh: scp (pscp sous windows, à télécharger sur le site: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).

Ajoutez les lignes suivantes à votre hello.py:
```py
@app.errorhandler(404)
def page_not_found(error):
    return render_template('page_not_found.html'), 404
```
Ainsi vous contrôlez la page d’erreur 404.

Modifiez la fonctions api_welcome_index de manière à retourner cette page 404 si jamais l’index n’est pas correct. Flask fournit une fonction pour cela : abort(404).

Une autre méthode aurai pu être utilisée: redirect avec url_for. plus d’info: https://flask.palletsprojects.com/en/1.1.x/quickstart/#redirects-and-errors
