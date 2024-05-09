# FlyFi

![GitHub last commit](https://img.shields.io/github/last-commit/M1txY/FlyFi)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/M1txY/FlyFi)
![GitHub contributors](https://img.shields.io/github/contributors/M1txY/FlyFi)
![License](https://img.shields.io/badge/license-MIT-green)

## Description

FlyFi est un projet qui permet aux passagers d'avion d'utiliser leur propre casque Bluetooth en se connectant sans fil à l'écran de l'avion via un port jack. Ce dispositif assure une meilleure qualité de son et une hygiène accrue en évitant l'utilisation des casques fournis.

## Matériel nécessaire

- **ESP32**: Microcontrôleur robuste pour gérer la communication Bluetooth et le traitement audio.
- **Module convertisseur DAC**: Convertit les signaux numériques de l'ESP32 en analogique.
- **Module ADC**: Convertit les signaux analogiques du jack en numérique pour l'ESP32.
- **Câble jack 3,5 mm**: Pour la connexion au système audio de l'avion.
- **Batterie ou source d'alimentation**: Alimente le dispositif.
- **Écran LCD QAPASS**: Affiche les informations de connexion.
- **Boutons**: Pour l'appairage et la gestion de la connexion Bluetooth.

## Schéma de montage

![Schéma](link-to-schematic-image)

### Instructions de montage

1. **ESP32 à Module ADC**:
   - **Données**: Connectez SDA, SCL pour I2C ou MOSI, MISO, SCK pour SPI.
   - **Alimentation**: VCC à 3.3V ou 5V, GND à GND.

2. **ESP32 à Module DAC**:
   - Répétez les connexions de données et d'alimentation comme pour l'ADC.

3. **Module ADC à Port Jack**:
   - Connectez les sorties audio (L, R) au port jack.

4. **Module DAC à Bluetooth**:
   - La sortie audio du DAC va à l'émetteur Bluetooth.

5. **ESP32 à Écran LCD et Boutons**:
   - Pour l'I2C, connectez SDA et SCL; pour les GPIO, utilisez les connexions appropriées.


# Code
```c
#include "BluetoothSerial.h"
#include "driver/i2s.h"
#include "Wire.h"
#include "LiquidCrystal_I2C.h"  // Assurez-vous d'installer cette bibliothèque pour votre écran LCD

// Vérifiez si Bluetooth est disponible sur le module
#if !defined(CONFIG_BT_ENABLED) || !defined(CONFIG_BLUEDROID_ENABLED)
#error Bluetooth is not enabled! Please run `make menuconfig` to and enable it
#endif

BluetoothSerial SerialBT;
LiquidCrystal_I2C lcd(0x27, 16, 2); // Adresse I2C et dimensions de l'écran (modifiez si nécessaire)

#define I2S_NUM         (0) // Numéro de I2S utilisé
#define SAMPLE_RATE     (44100)
#define SAMPLE_BITS     (16)
#define CHANNEL_NUM     (2)

// Pins pour les boutons
#define BUTTON_PAIR_PIN 32
#define BUTTON_CONNECT_PIN 33

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PAIR_PIN, INPUT_PULLUP);
  pinMode(BUTTON_CONNECT_PIN, INPUT_PULLUP);

  // Initialisation de l'écran LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Initializing...");

  // Initialisation du Bluetooth
  SerialBT.begin("FlyFi Audio System"); // Nom du Bluetooth
  Serial.println("Bluetooth device is ready to pair");
  lcd.setCursor(0, 1);
  lcd.print("BT Ready");

  // Configuration de I2S pour l'ADC
  i2s_config_t i2s_config = {
      .mode = (i2s_mode_t)(I2S_MODE_MASTER | I2S_MODE_RX | I2S_MODE_TX),
      .sample_rate = SAMPLE_RATE,
      .bits_per_sample = SAMPLE_BITS,
      .channel_format = I2S_CHANNEL_FMT_RIGHT_LEFT,
      .communication_format = I2S_COMM_FORMAT_I2S_MSB,
      .intr_alloc_flags = 0, // Default interrupt priority
      .dma_buf_count = 8,
      .dma_buf_len = 64,
      .use_apll = false,
      .tx_desc_auto_clear = true,
      .fixed_mclk = 0
  };

  i2s_pin_config_t pin_config = {
      .bck_io_num = 26,
      .ws_io_num = 25,
      .data_out_num = 22,
      .data_in_num = 23
  };

  i2s_driver_install(I2S_NUM, &i2s_config, 0, NULL);
  i2s_set_pin(I2S_NUM, &pin_config);
}

void loop() {
  static bool isPaired = false;
  static bool isConnected = false;

  // Gestion des boutons
  if (digitalRead(BUTTON_PAIR_PIN) == LOW) {
    isPaired = !isPaired;  // Simuler un basculement de l'état de pairage
    lcd.setCursor(0, 0);
    lcd.print(isPaired ? "Paired      " : "Unpaired    ");
  }

  if (digitalRead(BUTTON_CONNECT_PIN) == LOW) {
    isConnected = !isConnected;  // Simuler un basculement de l'état de connexion
    lcd.setCursor(0, 1);
    lcd.print(isConnected ? "Connected   " : "Disconnected");
  }

  // Simulation de la lecture et de l'envoi des données audio
  uint8_t data[512];
  size_t bytes_read;

  i2s_read(I2S_NUM, &data, sizeof(data), &bytes_read, portMAX_DELAY);

  if (bytes_read > 0 && isConnected) {
    SerialBT.write(data, bytes_read);
  }
}
```