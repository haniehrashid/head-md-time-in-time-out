/*
  Règles

  Projet dans le cadre du Master à l'head. 

  The circuit:
  - 3x LED attached from pin 13, 12 and 27 to 3.3V and ground through 220 ohm resistor
  - 2x pushbutton attached from pin 2 to +3.3V
  - 2x 10 kilohm resistor attached from pin 2 to ground
  - ...

  created January 2023
  by Hanieh Rashid

*/
#include <ESP32Servo360.h>

// Variables
int etatBoutonRegles = LOW;
int boutonReglesPressCount = 0;
int etatBoutonTemp = LOW;
int boutonTempPressCount = 0;
bool etatBuzzer = false;
int etatLed1 = LOW;
int etatLed2 = LOW;
int etatLed3 = LOW;
int etatRegles = 0;
int etatOvule = 0;
int jourDepuisOvule = 0;
int posMoteurRegles = 0;
int posMoteurOvule = 0;
int posMoteurJour = 0;
int tempVal;
float temperature = 0;
float previousTemperature = 0;

// Variables pour la mesure du temps
unsigned long previousMillisLed = 0;          // Temps depuis le dernier passage de la fonction LED
unsigned long previousJourMillis = 0;         // Temps depuis le dernier passage de la fonction jour
unsigned long boutonReglesPressStart = 0;     // Variable pour stocker le temps du début de la pression
unsigned long boutonReglesPressDuration = 0;  // Variable pour stocker la duration du temps de la pression
unsigned long boutonTempPressStart = 0;       // Variable pour stocker le temps du début de la pression
unsigned long boutonTempPressDuration = 0;    // Variable pour stocker la duration du temps de la pression
unsigned long buzzerStart = 0;                // Variable pour stocker la duration du temps d'activation du buzzer

// constantes qui ne changent pas
const long intervalLed = 500;                                                 // interval clignottement led
const long intervalBoutonRegles = 2000;                                       // 2 secondes pour le bouton regles
const long intervalBoutonTemp = 2000;                                         // 2 secondes pour le bouton température
const long intervalJour = 10000;                                              // 10 secondes pour 1 jour
const long intervalBuzzer = 500;                                              // Bip actif pendant 500ms
const float diffTemp = 2.0;                                                   // Différence de température
int positionsMoteurRegles[8] = {0, 45, 90, 135, 180, 225, 270, 315};  // Position du moteur règles
int positionsMoteurOvule[8] = {0, 45, 90, 135, 180, 225, 270, 315};   // Position du moteur Ovulation
int positionsMoteurJour[36] = {};                                             // Les positions seront définies par une boucle

// PIN
const int pinLed1 = 13;
const int pinLed2 = 12;
const int pinLed3 = 27;
const int pinBoutonRegles = 22;
const int pinBoutonTemp = 23;
const int pinTemp = A0;
const int pinServoRegles = 14;
const int pinFeedServoRegles = A1;
const int pinServoOvule = 32;
const int pinFeedServoOvule = A2;
const int pinServoJour = 15;
const int pinFeedServoJour = A3;
const int pinBuzzer = 21;

// Déclare les servos moteurs
ESP32Servo360 servoRegles;
ESP32Servo360 servoOvule;
ESP32Servo360 servoJour;

// Déclare un timer
hw_timer_t *timerBuzzer = NULL;

// Déclare la fonction que va appeller le timer
void IRAM_ATTR onBuzzerTimer();

void setup() {
  // Activation communication
  Serial.begin(9600);
  // Déclare les leds comme sortie
  pinMode(pinLed1, OUTPUT);
  pinMode(pinLed2, OUTPUT);
  pinMode(pinLed3, OUTPUT);
  pinMode(pinBuzzer, OUTPUT);

  // Déclare les boutons comme entrée
  pinMode(pinBoutonRegles, INPUT);
  pinMode(pinBoutonTemp, INPUT);

  // Initialisation du timer pour le buzzer
  timerBuzzer = timerBegin(0, 80, true);
  timerAttachInterrupt(timerBuzzer, &onBuzzerTimer, true);
  timerAlarmWrite(timerBuzzer, 10000, true);
  timerAlarmEnable(timerBuzzer);

  // Assigne les pins au servomoteurs
  servoRegles.attach(pinServoRegles, pinFeedServoRegles);
  servoOvule.attach(pinServoOvule, pinFeedServoOvule);
  servoJour.attach(pinServoJour, pinFeedServoJour);

  servoRegles.setSpeed(100);
  servoOvule.setSpeed(100);
  servoJour.setSpeed(100);

  // Création du tableau de position pour les jours
  for (int i = 0; i < 36; i++) {
    positionsMoteurJour[i] = i * 50;
  }

  // Met les moteur à leur position initiale
  servoRegles.rotateTo(positionsMoteurRegles [7]);
  servoOvule.rotateTo(positionsMoteurOvule [0]);
  servoJour.rotateTo(positionsMoteurJour [0]);

  servoRegles.wait();
  servoOvule.wait();
  servoJour.wait();
}

void loop() {
  // Récupère le temps actuel
  unsigned long currentMillis = millis();

  // Lit l'état des boutons et des ports analogiques
  etatBoutonRegles = digitalRead(pinBoutonRegles);
  etatBoutonTemp = digitalRead(pinBoutonTemp);
  tempVal = analogRead(pinTemp);

  // Code de pression du boutton règles
  if (etatBoutonRegles == HIGH) {

    // Si le bouton est pressé pour la première fois, met à jour le temps
    if (boutonReglesPressStart == 0) {
      boutonReglesPressStart = currentMillis;
    }

    // Calcule le temps de la pression du boutton règles
    boutonReglesPressDuration = currentMillis - boutonReglesPressStart;

    // Si la pression dure plus que le temps voulu
    if (boutonReglesPressDuration > intervalBoutonRegles) {
      boutonReglesPressCount++;
      boutonReglesPressStart = 0;

      // Active le buzzer
      etatBuzzer = true;  // Activer le buzzer
      buzzerStart = currentMillis;

      if (boutonReglesPressCount == 1 && etatOvule == 0) {  // Si c'est la première pression et que c'est pas en mode ovule
        etatLed3 = HIGH;
        etatRegles = 1;  // Active le mode règles
        Serial.println("Etat regles active");
        // Remet le moteur jour au début
        posMoteurJour = 0;
        servoJour.rotateTo(positionsMoteurJour [posMoteurJour]);
        servoJour.wait();
        Serial.print("Le moteur jour bouge vers position num: ");
        Serial.print(posMoteurJour);
        Serial.print(". Valeur de position du moteur : ");
        Serial.println(positionsMoteurJour[posMoteurJour]);
        posMoteurJour++;

        // Active le moteur regles
        //servoRegles.rotateTo(positionsMoteurRegles [posMoteurRegles]); //positionsMoteurRegles [posMoteurRegles]);
        servoRegles.rotateTo(45);
        /*
        Ici ca bug. Il bouge pas
        */
        Serial.println("Commande moteur regles donne");
        servoRegles.wait();
        Serial.println("Attente terminee");
        Serial.print("Le moteur regles bouge vers position num: ");
        Serial.print(posMoteurRegles);
        Serial.print(". Valeur de position du moteur : ");
        Serial.println(positionsMoteurRegles[posMoteurRegles]);
        posMoteurRegles++;

      } else if (boutonReglesPressCount == 2 && etatOvule == 0) {
        etatRegles = 0;
        etatLed3 = LOW;
        Serial.println("Etat regles desactive");

        boutonReglesPressCount = 0;
        servoRegles.rotateTo(positionsMoteurRegles [7]); // Va sur la position pas de règles
        servoRegles.wait();
        Serial.print("Le moteur regles bouge vers position num: 7");
        Serial.print(". Valeur de position du moteur : ");
        Serial.println(positionsMoteurRegles[posMoteurRegles]);
        posMoteurRegles = 0;  // Remise à Zero de la postion moteur

        etatOvule = 1;  // Activation mode ovulation
      } else {
        boutonReglesPressCount = 0;
      }
    }
  } else {
    boutonReglesPressStart = 0;  // Si le bouton n'est plus pressé, remet le temps de pression du bouton à 0
  }

  // Code pression du bouton température
  if (etatBoutonTemp == HIGH) {

    // Si le bouton est pressé pour la première fois, met à jour le temps
    if (boutonTempPressStart == 0) {
      boutonTempPressStart = currentMillis;
    }

    // Calcule le temps de la pression du boutton règles
    boutonTempPressDuration = currentMillis - boutonTempPressStart;

    // Si la pression dure plus que le temps voulu
    if (boutonTempPressDuration > intervalBoutonTemp) {
      boutonTempPressCount++;
      boutonTempPressStart = 0;

      // Active le buzzer
      etatBuzzer = true;  // Activer le buzzer
      buzzerStart = currentMillis;

      if (boutonTempPressCount == 1) {
        etatLed2 = HIGH;
        // Mesure la température
        // Imprime les valeurs analogique de la température
        Serial.print("Temp sensor Value: ");
        Serial.print(tempVal);

        float voltage = ((float)tempVal / 4096.0) * 3300.0;  // Converti la valeur de l'ADC en volts

        // Imprime les valeurs du voltage de la température
        Serial.print(", Volts: ");
        Serial.print(voltage);
        Serial.print(" mV");

        // Converti les valeurs de volts en degrés Celcius
        // Capteur augmente de 10 mV par degré avec un offset de 500 mV
        Serial.print(", degrees C: ");
        temperature = (voltage - 500) / 10;
        Serial.println(temperature);

        // Si en mode ovulation et une différence de température de plus de 2°C, passe directement a la position 3
        if (etatOvule == 1 && (temperature - previousTemperature) > diffTemp) {
          posMoteurOvule = 3;
          //active le moteur ovule
          servoOvule.rotateTo(positionsMoteurOvule [posMoteurOvule]);
          servoOvule.wait();
          Serial.print("Le moteur ovule bouge vers position num: ");
          Serial.print(posMoteurOvule);
          Serial.print(". Valeur de position du moteur : ");
          Serial.println(positionsMoteurOvule[posMoteurOvule]);
          jourDepuisOvule = 0;
        }
      }
    }
  } else {
    boutonTempPressStart = 0;  // Si le bouton n'est plus pressé, remet le temps de pression du bouton à 0
  }

  //Mettre a jour les positions des moteur a la fin de la journée
  if (currentMillis - previousJourMillis >= intervalJour) {
    previousJourMillis = currentMillis;  // Remet a 0 le temps de la journée
    Serial.println("Fin journee");
    // Si en mode règles
    if (etatRegles == 1) {
      // Active le moteur regles
      servoRegles.rotateTo(positionsMoteurRegles [posMoteurRegles]);
      servoRegles.wait();
      Serial.print("Le moteur regles bouge vers position num: ");
      Serial.print(posMoteurRegles);
      Serial.print(". Valeur de position du moteur : ");
      Serial.println(positionsMoteurRegles[posMoteurRegles]);
      posMoteurRegles++;

      // Si arrivé à jour 7 sans clic, desactive mode règles
      if (posMoteurRegles == 7) {
        etatRegles = 0;
        etatLed3 = LOW;
        Serial.println("Etat regles desactive");

        boutonReglesPressCount = 0;
        servoRegles.rotateTo(positionsMoteurRegles [7]); // Va sur la position pas de règles
        servoRegles.wait();
        Serial.print("Le moteur regles bouge vers position num: ");
        Serial.print(posMoteurRegles);
        Serial.print(". Valeur de position du moteur : ");
        Serial.println(positionsMoteurRegles[posMoteurRegles]);
        posMoteurRegles = 0;  // Remise à Zero de la postion moteur

        etatOvule = 1;
      }
    }

    // Si en mode ovulation
    if (etatOvule == 1) {
      jourDepuisOvule++;

      // Si le moteur n'a pas bougé depuis 3 jours et qu'il est à moins de 3, augmente
      if (posMoteurOvule < 3 && jourDepuisOvule > 3) {
        posMoteurOvule++;
        jourDepuisOvule = 0;
      } else if (posMoteurOvule == 3 && jourDepuisOvule >= 1) {  // Si la position est trois, augmente après 2 jours
        posMoteurOvule++;
        jourDepuisOvule = 0;
      } else if (posMoteurOvule > 3) {  // Si la position est plus de trois, augmente tous les jours
        posMoteurOvule++;
        jourDepuisOvule = 0;
      }
      //active le moteur ovule
      servoOvule.rotateTo(positionsMoteurOvule [posMoteurOvule]);
      servoOvule.wait();
      Serial.print("Le moteur ovule bouge vers position num: ");
      Serial.print(posMoteurOvule);
      Serial.print(". Valeur de position du moteur : ");
      Serial.println(positionsMoteurOvule[posMoteurOvule]);

      if (posMoteurOvule >= 7) {  // Si moteur arrive a la fin du tableau, fin de l'ovulation
        posMoteurOvule = 0;
        etatOvule = 0;
      }
    }

    // Change de jour le servo
    servoJour.rotateTo(positionsMoteurJour [posMoteurJour]);
    servoJour.wait();
    Serial.print("Le moteur jour bouge vers position num: ");
    Serial.print(posMoteurJour);
    Serial.print(". Valeur de position du moteur : ");
    Serial.println(positionsMoteurJour[posMoteurJour]);

    posMoteurJour++;

    if (posMoteurJour == 36) {
      posMoteurJour = 0;
    }

    //etatPrevBoutonRegles = etatBoutonRegles; // Remise à zero de l'état du boutton préssé
    boutonTempPressCount = 0;           //Remet à zéro le compatge du nombre de pression du bouton température car une mesure par jour.
    etatLed2 = LOW;                     // Eteint la led température
    previousTemperature = temperature;  // Met a jour la valeur de température du jour précédent
  }

  //Fait clignotter la led toutes les 500ms
  if (currentMillis - previousMillisLed >= intervalLed) {
    previousMillisLed = currentMillis;  // Sauvegarde le derniers temps que la led a clignotté
    if (etatLed1 == LOW) {
      etatLed1 = HIGH;
    } else {
      etatLed1 = LOW;
    }
  }

  //Met a jour l'état des leds
  digitalWrite(pinLed1, etatLed1);
  digitalWrite(pinLed2, etatLed2);
  digitalWrite(pinLed3, etatLed3);

  delay(1);
}

void IRAM_ATTR onBuzzerTimer() {
  if (etatBuzzer) {
    tone(pinBuzzer, 900);  // Activer le buzzer
    // Si 500 ms se sont écoulées depuis le début de l'activation du buzzer
    if (millis() - buzzerStart >= intervalBuzzer) {
      noTone(pinBuzzer);   // Désactiver le buzzer
      etatBuzzer = false;  // Mettre à jour l'état du buzzer
    }
  }
}
