# 🌐 Projeto IoT Cloud FIWARE - Vinheria Inteligente

*link wokwi:* https://wokwi.com/projects/445693935236921345

*link azure:* https://portal.azure.com/#@fiap.com.br/resource/subscriptions/3d864ba7-ef8d-4815-a813-c39277044caf/resourceGroups/azure-rg/providers/Microsoft.Compute/virtualMachines/vinheria-vm/networkSettings

### INTEGRANTES

Gustavo Oliveira Barroso: 565705
João Marcelo Diniz Vespa: 564038
Nicolas Santana Gara: 561461
Gustavo Garcia Silva: 562078

 ---
 
## 📘 Descrição do Projeto
Este projeto tem como objetivo **validar uma arquitetura IoT** e realizar uma **prova de conceito funcional (PoC)** demonstrando a comunicação entre dispositivos IoT simulados (ESP32) e a **plataforma IoT Cloud FIWARE**.  
A simulação é voltada ao **Projeto Vinheria**, um sistema de monitoramento inteligente de vinhedos, capaz de coletar e enviar dados ambientais em tempo real.
 
---
 
## 🎯 Objetivo Geral
Validar a arquitetura IoT e demonstrar a comunicação completa entre **dispositivos simulados no Wokwi** e a **plataforma IoT Cloud FIWARE**, coletando e transmitindo dados de sensores como temperatura, umidade, luminosidade e nível de alagamento (ultrassom).
 
---

### IMPORTANTE

Não foi possível realizar a conexão do wokwi com a plataforma IoT utilizada (VM criada no Azure), justificando a ausência de prints do Oreon recebendo as informações da implementação

---

## 🧩 Arquitetura Geral do Sistema
 
### Componentes Utilizados
- **Simulador Wokwi**: para emular o microcontrolador **ESP32** com sensores virtuais.  
- **Sensores simulados:**
  - DHT22 → Temperatura e Umidade  
  - LDR → Luminosidade  
  - HC-SR04 → Alagamento  
- **Plataforma IoT FIWARE**: responsável pelo gerenciamento, publicação e subscrição dos dados IoT.  
- **Protocolo MQTT**: utilizado para comunicação entre o ESP32 e o FIWARE.  
- **Broker FIWARE / Orion Context Broker**: processa os dados recebidos e os disponibiliza para visualização.  
- **Postman**: usado para validar as requisições de integração e consultar os dados publicados.
 
---
 
## 🧠 Fluxo de Comunicação
1. O ESP32 simulado no **Wokwi** coleta dados de sensores.
2. Os dados são enviados via **MQTT** para o **IoT Agent** configurado na **IoT Cloud FIWARE**.
3. O **IoT Agent** traduz os dados para o formato NGSI e os envia ao **Orion Context Broker**.
4. O **Orion Broker** permite a visualização e atualização dos dados em tempo real.
5. O **Postman** é utilizado para validar os endpoints e verificar as informações armazenadas.
 
---

## ⚙️ Etapas da Implementação
 
### 🧩 Entrega 1 – Criação da Plataforma IoT FIWARE (20 pts)
- Criação de uma **VM Linux com IP público** para instalação da plataforma FIWARE.
- Configuração do **Orion Context Broker**, **IoT Agent**, e **MongoDB**.  
- Alternativamente, uso do IP público **9.234.138.146**, com namespace `lamp00X` (ex: lamp003 até lamp020).
 
---

### 💻 Entrega 2 – Simulador Wokwi (40 pts)
- Criação do simulador usando **ESP32**
- Integração com sensores:
  - DHT22 → Temperatura / Umidade  
  - LDR → Luminosidade  
  - HC-SR04 → Alagamento  
- Configuração de envio de dados MQTT para a plataforma FIWARE (IoT Agent).  
- Visualização dos dados publicados e atualizados em tempo real na FIWARE.  
- **Prints anexados** mostrando o simulador enviando dados com sucesso.
 
---

### 🔄 Entrega 3 – Comunicação IoT (20 pts)
- Teste de **publicação e subscrição** de dados em tempo real.  
- Demonstração da comunicação funcional entre o **ESP32 (Wokwi)** e a **plataforma FIWARE**.  
- **Prints anexados** com o Orion Context Broker recebendo e exibindo as entidades atualizadas.

---

### CÓDIGO FONTE DO WOKWI

#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include "DHT.h"

#define DHTPIN 15
#define DHTTYPE DHT22
#define LDR_PIN 34
#define TRIG_PIN 5
#define ECHO_PIN 18

DHT dht(DHTPIN, DHTTYPE);

// === CONFIG WIFI ===
const char* WIFI_SSID = "Wokwi-GUEST";
const char* WIFI_PASS = "";  // vazio para o Wokwi-GUEST

// === CONFIG FIWARE ===
const char* serverName = "http://68.211.72.18/iot/d";  // ou o IP da sua VM FIWARE
const char* FIWARE_SERVICE = "Vinheria";

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LDR_PIN, INPUT);

  Serial.println("Conectando ao WiFi...");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi conectado!");
}

float readUltrasonic() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH);
  float distance = duration * 0.034 / 2;
  return distance; // cm
}

void loop() {
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  int luminosity = analogRead(LDR_PIN);
  float distance = readUltrasonic();

  if (isnan(temperature) || isnan(humidity)) {
    Serial.println("Erro ao ler DHT22!");
    delay(2000);
    return;
  }

  StaticJsonDocument<256> doc;
  doc["id"] = String("urn:ngsi-ld:Device:") + DEVICE_ID;
  doc["type"] = "VinheriaSensor";

  JsonObject temp = doc.createNestedObject("temperature");
  temp["value"] = temperature;
  temp["type"] = "Number";

  JsonObject hum = doc.createNestedObject("humidity");
  hum["value"] = humidity;
  hum["type"] = "Number";

  JsonObject lum = doc.createNestedObject("luminosity");
  lum["value"] = luminosity;
  lum["type"] = "Number";

  JsonObject flood = doc.createNestedObject("floodDistance");
  flood["value"] = distance;
  flood["type"] = "Number";

  char body[512];
  serializeJson(doc, body);

  HTTPClient http;
  String url = String("http://") + ORION_IP + ":" + ORION_PORT + "/v2/entities";
  http.begin(url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("Fiware-Service", FIWARE_SERVICE);
  http.addHeader("Fiware-ServicePath", "/");

  int code = http.POST((uint8_t*)body, strlen(body));
  Serial.println("===============================");
  Serial.println("Enviando dados para FIWARE:");
  Serial.println(body);
  Serial.print("Código de resposta: ");
  Serial.println(code);
  Serial.println("===============================");
  http.end();

  delay(5000); // envia a cada 5s
}
 
---

### IMAGENS

<img width="589" height="527" alt="image" src="https://github.com/user-attachments/assets/a13b621f-df44-4bc9-9e27-db0d5677ab0b" />

---

<img width="1162" height="591" alt="image" src="https://github.com/user-attachments/assets/8e2eeead-97f7-4d76-b92b-59550d377273" />

---



