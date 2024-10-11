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

Après Test on recois bien les caractere du printf. 


