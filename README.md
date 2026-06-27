# Cinta-Transportadora
Proyecto de cinta transportadora con Arduino
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

// MOTOR 
const int ENA = 9;
const int IN1 = 5;
const int IN2 = 6;

//SENSOR ULTRASONICO
const int trigPin = A1;
const int echoPin = A0;

//BOTONES
const int botonInicio = 11;
const int botonReset = 12;

// LEDS
const int ledVerde = 7;
const int ledRojo = 4;

//BUZZER 
const int buzzer = 8;

// VARIABLES
bool motorActivo = false;
bool objetoDetectado = false;
bool falla = false;
bool loteCompleto = false;

int contador = 0;
const int LIMITE_LOTE = 10;

unsigned long ultimoObjeto = 0;
unsigned long ultimoBoton = 0;

void setup() {

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  pinMode(botonInicio, INPUT_PULLUP);
  pinMode(botonReset, INPUT_PULLUP);

  pinMode(ledVerde, OUTPUT);
  pinMode(ledRojo, OUTPUT);

  pinMode(buzzer, OUTPUT);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0,0);
  lcd.print("CINTA LISTA");

  lcd.setCursor(0,1);
  lcd.print("CNT=0");

  detenerMotor();

  ultimoObjeto = millis();
}

void loop() {

  // ===== BOTON INICIO =====
  if (digitalRead(botonInicio) == LOW &&
      millis() - ultimoBoton > 300 &&
      !falla &&
      !loteCompleto) {

    ultimoBoton = millis();

    motorActivo = !motorActivo;

    if (motorActivo)
      arrancarMotor();
    else
      detenerMotor();
  }

  //  BOTON RESET 
  if (digitalRead(botonReset) == LOW &&
      millis() - ultimoBoton > 300) {

    ultimoBoton = millis();

    contador = 0;

    falla = false;
    loteCompleto = false;
    objetoDetectado = false;

    motorActivo = false;

    ultimoObjeto = millis();

    digitalWrite(ledRojo, LOW);
    digitalWrite(ledVerde, LOW);

    noTone(buzzer);

    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("SISTEMA RESET");
    lcd.setCursor(0,1);
    lcd.print("CNT=0");

    delay(1000);

    detenerMotor();
  }

  long distancia = medirDistancia();

  //DETECCION DE OBJETO
  if (motorActivo &&
      !falla &&
      !loteCompleto &&
      distancia > 0 &&
      distancia < 10 &&
      !objetoDetectado) {

    contador++;
    objetoDetectado = true;

    ultimoObjeto = millis();

    // Parpadeo LED verde
    digitalWrite(ledVerde, HIGH);
    delay(150);
    digitalWrite(ledVerde, LOW);

    // Beep corto
    tone(buzzer, 1200, 100);

    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("OBJETO #");
    lcd.print(contador);

    lcd.setCursor(0,1);
    lcd.print("CNT=");
    lcd.print(contador);
  }

  // Libera el conteo para el siguiente objeto
  if (distancia > 15) {
    objetoDetectado = false;
  }

  // FALLA 
  if (motorActivo &&
      !falla &&
      !loteCompleto &&
      (millis() - ultimoObjeto >= 5000)) {

    falla = true;

    detenerMotor();

    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("FALLA");

    lcd.setCursor(0,1);
    lcd.print("SIN OBJETO");
  }

  //LOTE COMPLETO
  if (contador >= LIMITE_LOTE &&
      !loteCompleto) {

    loteCompleto = true;

    detenerMotor();

    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("LOTE COMPLETO");

    lcd.setCursor(0,1);
    lcd.print("TOTAL=10");
  }

  //ALARMA DE FALLA 
  if (falla) {

    digitalWrite(ledRojo, HIGH);
    tone(buzzer, 1500);

    delay(250);

    digitalWrite(ledRojo, LOW);
    noTone(buzzer);

    delay(250);
  }

  // ALARMA LOTE COMPLETO 
  if (loteCompleto) {

    digitalWrite(ledRojo, HIGH);
    tone(buzzer, 2000);

    delay(250);

    digitalWrite(ledRojo, LOW);
    noTone(buzzer);

    delay(250);
  }
}


// SENSOR ULTRASONICO

long medirDistancia() {

  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);

  digitalWrite(trigPin, LOW);

  long duracion = pulseIn(echoPin, HIGH, 30000);

  long distancia = duracion * 0.034 / 2;

  return distancia;
}


// MOTOR ON

void arrancarMotor() {

  analogWrite(ENA, 255);

  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  digitalWrite(ledRojo, LOW);

  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("MOTOR ON");

  lcd.setCursor(0,1);
  lcd.print("CNT=");
  lcd.print(contador);
}


// MOTOR OFF

void detenerMotor() {

  analogWrite(ENA, 0);

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);

  digitalWrite(ledVerde, LOW);

  noTone(buzzer);

  lcd.setCursor(0,1);
  lcd.print("CNT=");
  lcd.print(contador);
}
