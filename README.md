# FlyFi 
 
## Description

le projet FlyFi est un projet qui a pour but de permettre de se connecter en Bluetooth a l'écran de l'avion au port jack pour utiliser notre propre casque bluetooth.

## Matériel nécessaire

1. **ESP32** : Ce microcontrôleur est assez puissant pour gérer la communication Bluetooth et le traitement du signal audio. [ESP32](https://fr.aliexpress.com/w/wholesale-esp32.html)
2. **Module convertisseur DAC (Digital to Analog Converter)** : Pour convertir le signal numérique provenant de l'ESP32 en un signal analogique que l'on peut envoyer à un casque.
3. **Module ADC (Analog to Digital Converter)** : Pour convertir le signal analogique du port jack en un signal numérique que l'ESP32 peut traiter. [ADC](https://fr.aliexpress.com/w/wholesale-adc.html)
4. **Câble jack 3,5 mm** : Pour se connecter à la sortie audio de l'écran.
5. **Batterie ou source d’alimentation** : Pour alimenter l'ESP32 et les modules ADC/DAC.
6. **Ecran LCD** : Pour afficher les informations de connexion et de déconnexion.


## Schéma de montage

1. **ESP32 à Module ADC** :
   - **Broches de données** : Connectez les broches SDA (Data) et SCL (Clock) de l'ESP32 aux broches correspondantes de l'ADC si vous utilisez une communication I2C. Si votre ADC utilise SPI, connectez les broches MOSI, MISO et SCK de l'ESP32 aux broches correspondantes de l'ADC.
   - **Alimentation** : Connectez VCC de l'ADC à une broche 3.3V ou 5V de l'ESP32 (selon la spécification de votre ADC), et GND à GND.

2. **ESP32 à Module DAC** :
   - **Broches de données** : De manière similaire à l'ADC, si le DAC utilise I2C, connectez SDA et SCL de l'ESP32 aux broches correspondantes du DAC. Pour SPI, connectez les broches MOSI, MISO et SCK.
   - **Alimentation** : Connectez VCC du DAC à une broche 3.3V ou 5V de l'ESP32, et GND à GND.

3. **Module ADC à Port Jack** :
   - Connectez la sortie (L et R pour stéréo, sinon juste L pour mono) du port jack à l'entrée correspondante sur l'ADC. Assurez-vous également de connecter les masses.

4. **Module DAC à Bluetooth (sortie audio)** :
   - Connectez la sortie du DAC à un émetteur Bluetooth ou à un circuit d'amplification si nécessaire avant de le transmettre au casque Bluetooth.

5. **ESP32 à Écran LCD** :
   - Si vous utilisez un écran LCD avec une interface I2C, connectez les broches SDA et SCL de l'ESP32 aux broches correspondantes de l'écran LCD.
   - Pour les connexions GPIO (pour les écrans non-I2C), connectez les broches GPIO de l'ESP32 aux broches de données de l'écran LCD selon votre bibliothèque ou pilote d'écran spécifique.
   - **Alimentation** : Connectez VCC de l'écran à une broche 3.3V ou 5V de l'ESP32, et GND à GND.

## Code
 
```c
#include "BluetoothSerial.h"
#include "driver/i2s.h"

// Vérifiez si Bluetooth est disponible sur le module
#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED)
#error Bluetooth is not enabled! Please run `make menuconfig` to and enable it
#endif

BluetoothSerial SerialBT;

#define I2S_NUM         (0) // Numéro de I2S utilisé
#define SAMPLE_RATE     (44100)
#define SAMPLE_BITS     (16)
#define CHANNEL_NUM     (2)

void setup() {
  Serial.begin(115200);

  // Initialisation du Bluetooth
  SerialBT.begin("FlyFi Audio System"); // Nom du Bluetooth
  Serial.println("Bluetooth device is ready to pair");

  // Configuration de I2S pour l'ADC
  i2s_config_t i2s_config = {
      .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX | I2S_MODE_TX),
      .sample_rate = SAMPLE_RATE,
      .bits_per_sample = (i2s_bits_per_sample_t)SAMPLE_BITS,
      .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT, // stéréo
      .communication_format = I2S_COMM_FORMAT_I2S | I2S_COMM_FORMAT_I2S_MSB,
      .intr_alloc_flags = ESP_INTR_FLAG_LEVEL1, // Interruption de priorité haute
      .dma_buf_count = 8,
      .dma_buf_len = 64,
      .use_apll = false,
      .tx_desc_auto_clear = true,
      .fixed_mclk = 0
  };

  i2s_pin_config_t pin_config = {
      .bck_io_num = 26,  // BCLK
      .ws_io_num = 25,   // LRCLK
      .data_out_num = 22,// DIN
      .data_in_num = 23  // DOUT
  };

  // Installation du pilote I2S
  i2s_driver_install(I2S_NUM, &i2s_config, 0, NULL);
  i2s_set_pin(I2S_NUM, &pin_config);
}

void loop() {
  // Buffer pour stocker les données audio
  uint8_t data[512];
  size_t bytes_read;

  // Lire les données audio de l'ADC via I2S
  i2s_read(I2S_NUM, &data, sizeof(data), &bytes_read, portMAX_DELAY);
  
  // Envoyer les données audio au Bluetooth
  if (bytes_read > 0) {
    SerialBT.write(data, bytes_read);
  }
}
```