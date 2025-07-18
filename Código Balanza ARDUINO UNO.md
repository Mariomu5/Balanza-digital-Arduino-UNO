#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "HX711.h"

// --- Pines ---
const int LOADCELL_DOUT_PIN = 2;
const int LOADCELL_SCK_PIN  = 3;
const int BTN_TARE = 4;
const int BTN_GANANCIA = 5;
const int BTN_RESET = 7;
const int BTN_FRASE = 6;  // NUEVO BOTÓN

// --- Objetos ---
HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// --- Variables globales ---
float cal_value_base = 1903.5;
float cal_value = cal_value_base;

bool mostrarEnMiligramos = true;
bool enTare = false;
bool gananciaAlta = true;
bool enCalibracion = false;

unsigned long tiempoUltimaLectura = 0;
unsigned long tiempoEstado = 0;

// Estados botones
bool btnTarePrev = HIGH;
bool btnGananciaPrev = HIGH;
bool btnResetPrev = HIGH;
bool btnFrasePrev = HIGH;

int estadoCalibracion = 0;
unsigned long ultimaPulsacionTare = 0;
int conteoClicsTare = 0;
bool sistemaEncendido = true;

// Filtro exponencial
float pesoFiltrado = 0.0;
const float ALPHA = 0.5;
float ultimoPesoMostrado = -9999.0;

void setup() {
  Serial.begin(57600);
  scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);

  scale.set_gain(128);
  cal_value = cal_value_base;
  scale.set_scale(cal_value);

  lcd.init();
  lcd.backlight();
  lcd.clear();

  pinMode(BTN_TARE, INPUT_PULLUP);
  pinMode(BTN_GANANCIA, INPUT_PULLUP);
  pinMode(BTN_RESET, INPUT_PULLUP);
  pinMode(BTN_FRASE, INPUT_PULLUP);

  lcd.setCursor(1, 0);
  lcd.print("Tare inicial...");
  delay(100);
  scale.tare(20);
  pesoFiltrado = 0.0;
  ultimoPesoMostrado = -9999.0;
  lcd.setCursor(2, 1);
  lcd.print("Listo!");
  delay(100);
  lcd.clear();

  mostrarDecoracionLCD();
  mostrarPesoLCD(0.0);
  tiempoUltimaLectura = millis();
}

void loop() {
  unsigned long ahora = millis();

  manejarBotones(ahora);
  verificarFraseEspecial();
  if (!sistemaEncendido) return;

  manejarCalibracion(ahora);
  mostrarPesoPeriodicamente(ahora);
}

void manejarBotones(unsigned long ahora) {
  bool btnTare = digitalRead(BTN_TARE);

  if (btnTare == LOW && btnTarePrev == HIGH) {
    unsigned long tiempoDesdeUltimo = ahora - ultimaPulsacionTare;
    ultimaPulsacionTare = ahora;
    conteoClicsTare++;

    if (conteoClicsTare == 2 && tiempoDesdeUltimo < 600) {
      iniciarCalibracion(ahora);
      conteoClicsTare = 0;
    }
  }

  if (conteoClicsTare == 1 && (ahora - ultimaPulsacionTare > 600)) {
    iniciarTare(ahora);
    conteoClicsTare = 0;
  }

  btnTarePrev = btnTare;

  if (enTare && !enCalibracion && ahora - tiempoEstado >= 500) {
    scale.tare(30);
    pesoFiltrado = 0.0;
    ultimoPesoMostrado = -9999.0;
    lcd.clear();
    mostrarDecoracionLCD();
    mostrarPesoLCD(0.0);
    Serial.println(">>> Tare completado");
    enTare = false;
  }

  bool btnGanancia = digitalRead(BTN_GANANCIA);
  if (btnGananciaPrev == HIGH && btnGanancia == LOW) {
    gananciaAlta = !gananciaAlta;
    mostrarEnMiligramos = gananciaAlta;

    scale.set_gain(gananciaAlta ? 128 : 64);
    cal_value = gananciaAlta ? cal_value_base : cal_value_base / 1.982;
    scale.set_scale(cal_value);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Modo: ");
    lcd.print(gananciaAlta ? "mg /128" : "g /64");
    lcd.setCursor(0, 1);
    lcd.print("Cal: ");
    lcd.print(cal_value, 1);

    Serial.print(">>> Modo cambiado a ");
    Serial.print(gananciaAlta ? "mg (128)" : "g (64)");
    Serial.print(" | Calibracion: ");
    Serial.println(cal_value, 2);

    delay(2000);
    lcd.clear();
  }
  btnGananciaPrev = btnGanancia;

  bool btnReset = digitalRead(BTN_RESET);
  if (btnReset == LOW && btnResetPrev == HIGH) {
    sistemaEncendido = !sistemaEncendido;

    if (sistemaEncendido) {
      Serial.println(">>> Sistema ENCENDIDO");
      lcd.backlight();
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Reiniciando...");
      delay(100);
      reiniciarSistema();
    } else {
      Serial.println(">>> Sistema APAGADO");
      lcd.clear();
      lcd.noBacklight();
    }
  }
  btnResetPrev = btnReset;
}

void reiniciarSistema() {
  cal_value = cal_value_base;
  mostrarEnMiligramos = true;
  gananciaAlta = true;
  enTare = false;
  enCalibracion = false;
  tiempoUltimaLectura = millis();
  estadoCalibracion = 0;
  conteoClicsTare = 0;
  pesoFiltrado = 0.0;
  ultimoPesoMostrado = -9999.0;

  scale.set_gain(128);
  scale.set_scale(cal_value);
  scale.tare(20);

  lcd.clear();
  lcd.setCursor(1, 0);
  lcd.print("Sistema listo!");
  delay(100);
  lcd.clear();
  mostrarDecoracionLCD();
  mostrarPesoLCD(0.0);
}

void iniciarTare(unsigned long ahora) {
  enTare = true;
  tiempoEstado = ahora;
  pesoFiltrado = 0.0;
  ultimoPesoMostrado = -9999.0;
  lcd.clear();
  lcd.setCursor(4, 0);
  lcd.print("Tarando...");
}

void iniciarCalibracion(unsigned long ahora) {
  if (!gananciaAlta) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Calib. solo en");
    lcd.setCursor(0, 1);
    lcd.print("modo miligramos");
    delay(1500);
    lcd.clear();
    return;
  }

  enCalibracion = true;
  estadoCalibracion = 1;
  tiempoEstado = ahora;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Modo calibracion");
  delay(500);
}

void manejarCalibracion(unsigned long ahora) {
  if (!enCalibracion) return;

  switch (estadoCalibracion) {
    case 1:
      lcd.clear();
      lcd.setCursor(2, 0);
      lcd.print("Retira peso");
      tiempoEstado = ahora;
      estadoCalibracion = 2;
      break;

    case 2:
      if (ahora - tiempoEstado >= 3000) {
        scale.set_scale();
        scale.tare();
        pesoFiltrado = 0.0;
        ultimoPesoMostrado = -9999.0;
        lcd.clear();
        lcd.setCursor(3, 0);
        lcd.print("Coloca 100g");
        tiempoEstado = ahora;
        estadoCalibracion = 3;
      }
      break;

    case 3:
      if (ahora - tiempoEstado >= 4000) {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Leyendo peso...");
        delay(300);

        float lectura = scale.get_units(20);
        cal_value_base = lectura / 100.0;
        cal_value = cal_value_base;
        scale.set_scale(cal_value_base);

        Serial.print(">>> Nuevo cal_value_base: ");
        Serial.println(cal_value_base, 3);

        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Nuevo Cal:");
        lcd.setCursor(0, 1);
        lcd.print(cal_value_base, 2);

        tiempoEstado = ahora;
        estadoCalibracion = 4;
      }
      break;

    case 4:
      if (ahora - tiempoEstado >= 3000) {
        lcd.clear();
        estadoCalibracion = 0;
        enCalibracion = false;
      }
      break;
  }
}

void mostrarPesoPeriodicamente(unsigned long ahora) {
  if (enTare || enCalibracion) return;

  if (ahora - tiempoUltimaLectura >= 500) {  // 3 segundos entre lecturas
    tiempoUltimaLectura = ahora;

    float lecturaActual = scale.get_units(5);

    if (pesoFiltrado == 0.0) {
      pesoFiltrado = lecturaActual;
    } else {
      pesoFiltrado = ALPHA * lecturaActual + (1 - ALPHA) * pesoFiltrado;
    }

    mostrarDecoracionLCD();
    mostrarPesoLCD(pesoFiltrado);
  }
}

void mostrarDecoracionLCD() {
  lcd.setCursor(0, 0);
  lcd.print("Esto pesa...");
}

void mostrarPesoLCD(float peso) {
  float pesoRedondeado = mostrarEnMiligramos
    ? round(peso * 200.0) / 200.0   // 5 mg
    : round(peso * 10.0) / 10.0;    // 0.1 g

  bool estabilizado = abs(pesoRedondeado - ultimoPesoMostrado) < 0.05;

  // Mostrar asterisco si estable
  lcd.setCursor(15, 1);
  lcd.print(estabilizado && abs(pesoRedondeado) > 0.001 ? "*" : " ");

  if (estabilizado) return;

  ultimoPesoMostrado = pesoRedondeado;

  lcd.setCursor(4, 1);
  lcd.print("          ");
  lcd.setCursor(4, 1);

  if (mostrarEnMiligramos) {
    lcd.print(max(0.0, pesoRedondeado * 1000), 0);
    lcd.print(" mg");
  } else {
    lcd.print(pesoRedondeado >= 0 ? pesoRedondeado : 0.0, 1);
    lcd.print(" g");
  }
}

void verificarFraseEspecial() {
  if (!sistemaEncendido || enCalibracion || enTare) return;

  bool btnFrase = digitalRead(BTN_FRASE);
  if (btnFrasePrev == HIGH && btnFrase == LOW) {
    lcd.clear();
    lcd.setCursor(2, 0);
    lcd.print("Le Gusto");
    lcd.setCursor(4, 1);
    lcd.print("su merced?");
    Serial.println(">>> Frase especial mostrada");
    delay(2000);
    lcd.clear();
    mostrarDecoracionLCD();
    mostrarPesoLCD(ultimoPesoMostrado);
  }
  btnFrasePrev = btnFrase;
}
