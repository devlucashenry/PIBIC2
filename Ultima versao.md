#define BLYNK_TEMPLATE_ID "TMPL2XFIn-qUZ"
#define BLYNK_TEMPLATE_NAME "Lucas Henry"
#define BLYNK_AUTH_TOKEN "huUPVgRbrmUZ9OETceP2fl0BV7yemWYu"  // Seu token Blynk

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>

// Dados da rede Wi-Fi
char ssid[] = "Saga";         // Nome da sua rede Wi-Fi
char pass[] = "12345678";     // Senha da sua rede Wi-Fi

// Pinos
const int LED_Pin = 26;
const int PINO_SENSOR = 25;

// Variáveis do sensor de fluxo
volatile unsigned long contador = 0;
const float FATOR_CALIBRACAO = 4.5;
float fluxo = 0;
float volume_total = 0;
unsigned long tempo_antes = 0;

// VPin do Blynk
#define VPIN_FLUXO V0
#define VPIN_VOLUME V1
#define VPIN_LED V2

// Função de interrupção para contar pulsos do sensor
void IRAM_ATTR contador_pulso() {
  contador++;
}

// Controle manual do LED pelo Blynk (se quiser usar também pelo app)
BLYNK_WRITE(VPIN_LED) {
  int valor = param.asInt();
  digitalWrite(LED_Pin, valor);
}

// Função para calcular fluxo e volume
void calcularFluxo() {
  detachInterrupt(PINO_SENSOR);

  unsigned long tempo_agora = millis();
  float tempo_passado = tempo_agora - tempo_antes;

  fluxo = ((1000.0 / tempo_passado) * contador) / FATOR_CALIBRACAO;
  float volume = fluxo / 60.0; // volume em litros (L)
  volume_total += volume;

  // Verificar se fluxo é aproximadamente 12 L/min
  if (abs(fluxo - 12.0) <= 0.2) { // Margem de erro de ±0.2 L/min
    digitalWrite(LED_Pin, HIGH);  // Acende o LED
    Blynk.logEvent("fluxo_ideal", "Fluxo atingiu 12 L/min!"); // Envia notificação
  } else {
    digitalWrite(LED_Pin, LOW);   // Apaga o LED
  }

  // Enviar valores para o Blynk
  Blynk.virtualWrite(VPIN_FLUXO, fluxo);
  Blynk.virtualWrite(VPIN_VOLUME, volume_total);

  // Exibir no Serial Monitor
  Serial.print("Fluxo: ");
  Serial.print(fluxo, 2);
  Serial.println(" L/min");

  Serial.print("Volume total: ");
  Serial.print(volume_total, 3);
  Serial.println(" L\n");

  contador = 0;
  tempo_antes = tempo_agora;

  attachInterrupt(PINO_SENSOR, contador_pulso, FALLING);
}

void setup() {
  Serial.begin(115200);

  pinMode(LED_Pin, OUTPUT);
  pinMode(PINO_SENSOR, INPUT_PULLUP);

  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  tempo_antes = millis();
  attachInterrupt(PINO_SENSOR, contador_pulso, FALLING);
}

void loop() {
  Blynk.run();

  // Atualizar o cálculo de fluxo a cada segundo
  static unsigned long ultima_att = 0;
  if (millis() - ultima_att >= 1000) { // A cada 1000 ms (1 segundo)
    ultima_att = millis();
    calcularFluxo();
  }
}

