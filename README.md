# TP_ESE_BusEtResaux

Compte rendu de Nolan Jacquot et Gabriel Guiffault

2. TP 1 - Bus I2C

Objectif: Interfacer un STM32 avec des capteurs I2C

TP1

La première étape est de mettre en place la communication entre le microcontrôleur et les capteurs (température, pression, accéléromètre...) via  le bus I2C.
Le capteur comporte 2 composants I2C, qui partagent le même bus. Le STM32 jouera le rôle de Master sur le bus.Le code du STM32 sera écrit en langage C, en utilisant la bibliothèque HAL.

2.1. Capteur BMP280

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

2.2. Setup du STM32

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
2.3. Communication I²C

Primitives I²C sous STM32_HAL

Communication avec le BMP280

Identification du BMP280

L'identification du BMP280 consiste en la lecture du registre ID

En I²C, la lecture se déroule de la manière suivante:

envoyer l'adresse du registre ID puis recevoir 1 octet correspondant au contenu du registre
```C
#define BMP280_I2C_ADDRESS (0x77 << 1)
```
L'adresse est indiqué directement sur la seriographie attention au decalage de 1 bit!!
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
 Le registre "id" est 0xD0, et la valeur d'identification est 0x58.Il est preferable d'utiliser check ID plutot que read ID
```C


 uint8_t id = bmp280_read_id();
    if (id == 0x58)
    {
        printf("BMP280 detected! ID: 0x%02X\r\n", id);
    }
    else
    {
        printf("Failed to detect BMP280. ID: 0x%02X\r\n", id);
    }
```
Nous avons rajouté ce code pour verifier si l'ID étais bien presente.

Configuration du BMP280

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

Récupérez en une fois le contenu des registres qui contiennent l'étalonnage du BMP280.

Dans la boucle infinie du STM32, récupérez les valeurs de la température et de la pression. Envoyez sur le port série la valeurs 32 bit non compensées de la pression de la température. 

Calcul des températures et des pression compensées

Retrouvez dans la datasheet du STM32 le code permettant de compenser la température et la pression à l'aide des valeurs de l'étalonnage au format entier 32 bits (on utilisera pas les flottants pour des problèmes de performance).

Transmettez sur le port série les valeurs compensés de température et de pression sous un format lisible.
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
Apparamment c'est la seule méthode de le faire
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

