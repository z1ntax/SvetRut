#include <SPI.h>
#include <MFRC522.h>

#define SS_PIN 5
#define RST_PIN 22
#define LIGHT_PIN 4

MFRC522 rfid(SS_PIN, RST_PIN);

// Разрешённая карточка: FC 78 2F 8E
byte allowedUID[4] = {0xFC, 0x78, 0x2F, 0x8E};

bool lightState = false;
bool exitTimerActive = false;
bool cardLocked = false; // защита от повторного считывания, пока карта не убрана

unsigned long exitTimerStart = 0;
const unsigned long delayBeforeOff = 5000; // 5 секунд до выключения

bool isAllowedCard() {
  if (rfid.uid.size != 4) {
    return false;
  }

  for (byte i = 0; i < 4; i++) {
    if (rfid.uid.uidByte[i] != allowedUID[i]) {
      return false;
    }
  }

  return true;
}

void printUID() {
  Serial.print("Card UID: ");

  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) {
      Serial.print("0");
    }

    Serial.print(rfid.uid.uidByte[i], HEX);

    if (i < rfid.uid.size - 1) {
      Serial.print(" ");
    }
  }

  Serial.println();
}

bool isAnyCardNear() {
  byte bufferATQA[2];
  byte bufferSize = sizeof(bufferATQA);

  MFRC522::StatusCode result = rfid.PICC_WakeupA(bufferATQA, &bufferSize);

  if (result == MFRC522::STATUS_OK || result == MFRC522::STATUS_COLLISION) {
    return true;
  }

  return false;
}

void turnLightOn() {
  digitalWrite(LIGHT_PIN, HIGH);
  lightState = true;
  exitTimerActive = false;

  Serial.println("ENTRY registered");
  Serial.println("LIGHT ON");
}

void startExitTimer() {
  exitTimerActive = true;
  exitTimerStart = millis();

  Serial.println("EXIT registered");
  Serial.println("LIGHT OFF timer started");
}

void turnLightOff() {
  digitalWrite(LIGHT_PIN, LOW);
  lightState = false;
  exitTimerActive = false;

  Serial.println("LIGHT OFF");
}

void processAllowedCard() {
  if (!lightState) {
    // Свет выключен — значит это вход
    turnLightOn();
  } else {
    // Свет включен — значит это выход
    if (!exitTimerActive) {
      startExitTimer();
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(3000);

  pinMode(LIGHT_PIN, OUTPUT);
  digitalWrite(LIGHT_PIN, LOW);

  SPI.begin(18, 19, 23, 5);
  rfid.PCD_Init();

  Serial.println();
  Serial.println("SVETRUT START");
  Serial.println("Mode: first scan = entry, second scan = exit");
  Serial.println("Waiting for card...");
}

void loop() {
  // Если карта уже была считана, ждём пока её уберут
  if (cardLocked) {
    if (!isAnyCardNear()) {
      cardLocked = false;
      Serial.println("Card removed");
      delay(300);
    }
  }

  // Считываем карту только если предыдущую уже убрали
  if (!cardLocked && rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    printUID();

    if (isAllowedCard()) {
      processAllowedCard();
    } else {
      Serial.println("ACCESS DENIED");
    }

    cardLocked = true;

    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();

    delay(300);
  }

  // Таймер выключения после второго прикладывания
  if (exitTimerActive && millis() - exitTimerStart >= delayBeforeOff) {
    turnLightOff();
  }

  delay(100);
}
