```cpp

#include <WiFi.h>
#include <PubSubClient.h>

//  Configurações de rede e Ubidots
#define WIFISSID "SHARE-RESIDENTE"
#define PASSWORD "Share@residente23"
#define TOKEN "BBUS-C9jMBSEH7QDQ3HYVLxYbuIAmbmB4YN"
#define DEVICE_LABEL "semaforo_esp32"
#define VARIABLE_LDR "ldr_luminosidade"
#define VARIABLE_ATIVO "sistema_ativo"
#define MQTT_SERVER "industrial.api.ubidots.com"
#define MQTT_PORT 1883

//  Sensor LDR
#define LDR_PIN 35

//  Sensor Ultrassônico
#define TRIG_PIN 19
#define ECHO_PIN 18

//  Pinos dos LEDs dos semáforos
#define VERMELHO1 33
#define AMARELO1 27
#define VERDE1 26

#define VERMELHO2 21
#define AMARELO2 22
#define VERDE2 23

WiFiClient espClient;
PubSubClient client(espClient);

//  Controle de tempo
unsigned long tempoAtual;
unsigned long ultimoTempo = 0;
unsigned long ultimoSensor = 0;
unsigned long tempoUltimaDeteccao = 0;
int fase = 0; // 0 = verde, 1 = amarelo, 2 = troca
bool modoNoturno = false;
bool sistemaAtivo = false;
bool semaforo1Ativo = true;
bool alertaSensor = false;

//  Classe que representa um semáforo
class Semaforo {
  public:
    int vermelho, amarelo, verde;
    Semaforo(int v, int a, int g) {
      vermelho = v;
      amarelo = a;
      verde = g;
      pinMode(vermelho, OUTPUT);
      pinMode(amarelo, OUTPUT);
      pinMode(verde, OUTPUT);
    }

    void vermelhoOn() {
      digitalWrite(vermelho, HIGH);
      digitalWrite(amarelo, LOW);
      digitalWrite(verde, LOW);
    }

    void verdeOn() {
      digitalWrite(vermelho, LOW);
      digitalWrite(amarelo, LOW);
      digitalWrite(verde, HIGH);
    }

    void amareloOn() {
      digitalWrite(vermelho, LOW);
      digitalWrite(amarelo, HIGH);
      digitalWrite(verde, LOW);
    }

    void apagar() {
      digitalWrite(vermelho, LOW);
      digitalWrite(amarelo, LOW);
      digitalWrite(verde, LOW);
    }

    void piscaAmarelo() {
      digitalWrite(amarelo, millis() % 1000 < 300 ? HIGH : LOW);
      digitalWrite(vermelho, LOW);
      digitalWrite(verde, LOW);
    }

    void piscarTodos() {
      bool estado = millis() % 1000 < 400;
      digitalWrite(vermelho, estado);
      digitalWrite(amarelo, estado);
      digitalWrite(verde, estado);
    }
};

Semaforo semaforo1(VERMELHO1, AMARELO1, VERDE1);
Semaforo semaforo2(VERMELHO2, AMARELO2, VERDE2);

//  Recebe comandos MQTT do Ubidots
void callback(char* topic, byte* payload, unsigned int length) {
  String incoming = "";
  for (int i = 0; i < length; i++) {
    incoming += (char)payload[i];
  }

  if (String(topic).indexOf(VARIABLE_ATIVO) != -1) {
    sistemaAtivo = incoming.toInt() == 1;
    Serial.print("Sistema ativo via botão: ");
    Serial.println(sistemaAtivo);
    if (!sistemaAtivo) {
      semaforo1.apagar();
      semaforo2.apagar();
    }
  }
}

//  Reconecta ao servidor MQTT
void reconnect() {
  while (!client.connected()) {
    Serial.println("Tentando conectar ao MQTT...");
    if (client.connect("ESP32Client", TOKEN, "")) {
      Serial.println("Conectado ao Ubidots!");
      String topicAtivo = "/v1.6/devices/" + String(DEVICE_LABEL) + "/" + String(VARIABLE_ATIVO) + "/lv";

      client.subscribe(topicAtivo.c_str());
    } else {
      Serial.print("Falha na conexão, rc=");
      Serial.println(client.state());
      delay(2000);
    }
  }
}

//  Lê o valor do sensor LDR
int lerLDR() {
  return analogRead(LDR_PIN);
}

//  Lê a distância do sensor ultrassônico
float lerDistanciaCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duracao = pulseIn(ECHO_PIN, HIGH, 30000);
  return duracao * 0.034 / 2;
}

//  Setup inicial
void setup() {
  Serial.begin(115200);
  WiFi.begin(WIFISSID, PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("WiFi conectado!");

  client.setServer(MQTT_SERVER, MQTT_PORT);
  client.setCallback(callback);

  pinMode(LDR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
}

//  Loop principal
void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  tempoAtual = millis();
  int ldrValor = lerLDR();

  //  Leitura do sensor ultrassônico a cada 300ms
  float distancia = 999;
  if (tempoAtual - ultimoSensor > 300) {
    distancia = lerDistanciaCM();
    ultimoSensor = tempoAtual;

    // Se detectar objeto próximo, ativa alerta e salva tempo
    if (distancia > 2 && distancia < 8) {
      alertaSensor = true;
      tempoUltimaDeteccao = tempoAtual;
    }
  }

  //  Envia valor do LDR para Ubidots
  String payload = "{\"" VARIABLE_LDR "\":" + String(ldrValor) + "}";
  String topic = "/v1.6/devices/" DEVICE_LABEL;
  client.publish(topic.c_str(), payload.c_str());

  //  Sistema desligado
  if (!sistemaAtivo) {
    semaforo1.apagar();
    semaforo2.apagar();
    return;
  }

  //  Alerta do sensor ultrassônico com persistência de 1 segundo
  if (alertaSensor) {
    if (tempoAtual - tempoUltimaDeteccao < 1000) {
      semaforo1.piscarTodos();
      semaforo2.piscarTodos();
      return;
    } else {
      alertaSensor = false;
      semaforo1.apagar();
      semaforo2.apagar();
    }
  }

  //  Modo noturno automático com histerese
  if (modoNoturno && ldrValor >= 900) {
    modoNoturno = false;
  } else if (!modoNoturno && ldrValor < 800) {
    modoNoturno = true;
  }

  if (modoNoturno) {
    semaforo1.piscaAmarelo();
    semaforo2.piscaAmarelo();
    return;
  }

  //  Ciclo dos semáforos
  if (tempoAtual - ultimoTempo > 8000) {
    fase = 0;
    ultimoTempo = tempoAtual;
    semaforo1Ativo = !semaforo1Ativo;
  }

  if (fase == 0) {
    if (semaforo1Ativo) {
      semaforo1.verdeOn();
      semaforo2.vermelhoOn();
    } else {
      semaforo2.verdeOn();
      semaforo1.vermelhoOn();
    }
    if (tempoAtual - ultimoTempo > 6000) fase = 1;
  } else if (fase == 1) {
    if (semaforo1Ativo) {
      semaforo1.amareloOn();
      semaforo2.vermelhoOn();
    } else {
      semaforo2.amareloOn();
      semaforo1.vermelhoOn();
    }
    if (tempoAtual - ultimoTempo > 8000) fase = 2;
  }
}

```
