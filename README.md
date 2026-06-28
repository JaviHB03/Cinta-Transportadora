#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

//PINES
// Motor L298N
const byte ENA = 9;
const byte IN1 = 5;
const byte IN2 = 6;

// Sensor HC-SR04
const byte trigPin = A1;
const byte echoPin = A0;

// Botones
const byte botonInicio = 11;
const byte botonReset = 12;

// LEDs
const byte ledVerde = 7;
const byte ledRojo = 4;

// Buzzer
const byte buzzer = 8;

// CONFIGURACION

const int VELOCIDAD = 255;
const int DISTANCIA_DETECCION = 10;
const int DISTANCIA_LIBERAR = 15;
const int LIMITE_LOTE = 10;

const unsigned long TIEMPO_FALLA = 5000;
const unsigned long TIEMPO_BOTON = 250;
const unsigned long TIEMPO_LCD = 300;

//ESTADOS 

enum EstadoSistema
{
  ESPERA,
  MARCHA,
  FALLA,
  LOTE_COMPLETO
};

EstadoSistema estado = ESPERA;


int contador = 0;

bool objetoDetectado = false;

// CONTROL BUZZER
bool beepActivo = false;
unsigned long tiempoBeep = 0;

unsigned long ultimoObjeto = 0;
unsigned long ultimoBoton = 0;
unsigned long ultimoLCD = 0;

long distancia = 0;

//SETUP 

void setup()
{
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

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("CINTA LISTA");

  lcd.setCursor(0, 1);
  lcd.print("CNT:0");

  detenerMotor();

  ultimoObjeto = millis();
}
void loop()
{
  //BOTON INICIO 
  if (digitalRead(botonInicio) == LOW && millis() - ultimoBoton > TIEMPO_BOTON)
  {
    ultimoBoton = millis();

    if (estado == ESPERA)
    {
      estado = MARCHA;
      arrancarMotor();
      ultimoObjeto = millis();
    }
    else if (estado == MARCHA)
    {
      estado = ESPERA;
      detenerMotor();
    }
  }

  //BOTON RESET
  if (digitalRead(botonReset) == LOW && millis() - ultimoBoton > TIEMPO_BOTON)
  {
    ultimoBoton = millis();

    contador = 0;
    objetoDetectado = false;
    estado = ESPERA;
    ultimoObjeto = millis();

    detenerMotor();

    digitalWrite(ledRojo, LOW);
    digitalWrite(ledVerde, LOW);

    noTone(buzzer);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SISTEMA RESET");
    lcd.setCursor(0, 1);
    lcd.print("CNT:0");

    delay(500);
    lcd.clear();
  }

  //SENSOR 
  distancia = distanciaFiltrada();

  if (estado == MARCHA)
  {
    if (distancia > 0 &&
        distancia < DISTANCIA_DETECCION &&
        !objetoDetectado)
    {
      contador++;
      objetoDetectado = true;
      ultimoObjeto = millis();

      digitalWrite(ledVerde, HIGH);

      // (OBJETO DETECTADO)
      tone(buzzer, 2000);
      beepActivo = true;
      tiempoBeep = millis();
    }

    if (distancia > DISTANCIA_LIBERAR)
    {
      objetoDetectado = false;
      digitalWrite(ledVerde, LOW);
    }
  }

  //FALLA 
  if (estado == MARCHA)
  {
    if (millis() - ultimoObjeto >= TIEMPO_FALLA)
    {
      estado = FALLA;
      detenerMotor();
    }
  }

  //=========== LOTE COMPLETO ===========
  if (contador >= LIMITE_LOTE)
  {
    estado = LOTE_COMPLETO;
    detenerMotor();
  }

  //LCD
  if (millis() - ultimoLCD >= TIEMPO_LCD)
  {
    ultimoLCD = millis();
    actualizarLCD();
  }

  // ALARMA 
  actualizarAlarma();

  //APAGAR 
  if (beepActivo && millis() - tiempoBeep >= 80)
  {
    noTone(buzzer);
    beepActivo = false;
  }
}

// FUNCIONES

long medirDistancia()
{
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);

  digitalWrite(trigPin, LOW);

  long duracion = pulseIn(echoPin, HIGH, 30000);

  if (duracion == 0) return 0;

  return duracion * 0.0343 / 2;
}

long distanciaFiltrada()
{
  long suma = 0;
  byte lecturas = 0;

  for (byte i = 0; i < 3; i++)
  {
    long d = medirDistancia();

    if (d > 0)
    {
      suma += d;
      lecturas++;
    }

    delay(5);
  }

  if (lecturas == 0) return 0;

  return suma / lecturas;
}

void arrancarMotor()
{
  analogWrite(ENA, VELOCIDAD);
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
}

void detenerMotor()
{
  analogWrite(ENA, 0);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
}

void actualizarLCD()
{
  lcd.setCursor(0, 0);

  switch (estado)
  {
    case ESPERA: lcd.print("ESPERA         "); break;
    case MARCHA: lcd.print("MOTOR ON       "); break;
    case FALLA: lcd.print("FALLA          "); break;
    case LOTE_COMPLETO: lcd.print("LOTE COMPLETO  "); break;
  }

  lcd.setCursor(0, 1);
  lcd.print("C:");
  lcd.print(contador);
  lcd.print(" D:");
  lcd.print(distancia);
  lcd.print("   ");
}

void actualizarAlarma()
{
  static unsigned long ultimoCambio = 0;
  static bool estadoLed = false;

  if (estado == FALLA)
  {
    if (millis() - ultimoCambio >= 250)
    {
      ultimoCambio = millis();
      estadoLed = !estadoLed;

      digitalWrite(ledRojo, estadoLed);

      if (estadoLed) tone(buzzer, 1500);
      else noTone(buzzer);
    }
    return;
  }

  if (estado == LOTE_COMPLETO)
  {
    if (millis() - ultimoCambio >= 200)
    {
      ultimoCambio = millis();
      estadoLed = !estadoLed;

      digitalWrite(ledRojo, estadoLed);

      if (estadoLed) tone(buzzer, 2200);
      else noTone(buzzer);
    }
    return;
  }

  digitalWrite(ledRojo, LOW);
}
