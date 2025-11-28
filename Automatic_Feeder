#include <ESP8266WiFi.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <Servo.h>
#include <ESP8266WebServer.h>
#include <stdio.h>

typedef struct {
    int horas_ativacao; 
    int minutos_ativacao; 
} Horario;

const char *rede = "NOME_REDE";
const char *senha = "SENHA_REDE";

WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", -10800);

Servo Servo1;
int pino_servo = D2;
const int trig_pin = D5;
const int echo_pin = D6;
long duracao;
float distanciaCm;
int distancia_tampa_base = 30; 
float cm_racao;
float percent_racao;

ESP8266WebServer server(80);

Horario horario;
void abre_servo() { Servo1.write(0); }
void fecha_servo() { Servo1.write(180); }

void setupWiFi() {
  Serial.print("Conectando em ");
  Serial.print(rede);
  WiFi.begin(rede, senha);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println(WiFi.localIP());
}

void calculaRacao() {
  digitalWrite(trig_pin, LOW);
  delayMicroseconds(2);
  digitalWrite(trig_pin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trig_pin, LOW);
  
  duracao = pulseIn(echo_pin, HIGH);
  distanciaCm = duracao * 0.034 / 2;
  
  cm_racao = distancia_tampa_base - distanciaCm;
  if (cm_racao <= 1.5) cm_racao = 0;

  percent_racao = (cm_racao / distancia_tampa_base) * 100;
}

void setHorario(int horas, int minutos) {
  horario.horas_ativacao = horas;
  horario.minutos_ativacao = minutos;
}

void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Controle de Dispenser de Ração</title>
  <style>
    body { font-family: Arial, sans-serif; display: flex; align-items: center; justify-content: center; height: 100vh; background-color: #f4f4f9; }
    .container { text-align: center; }
    .progress-bar-container { width: 80%; background-color: #ddd; border-radius: 10px; margin: 20px auto; height: 30px; }
    .progress-bar { height: 100%; width: 0; background-color: #4caf50; border-radius: 10px; text-align: center; color: white; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Dispenser de Ração</h1>
    <div class="progress-bar-container">
      <div class="progress-bar" id="progress-bar">0%</div>
    </div>
    <div>
      <label for="horas">Horas:</label>
      <input type="number" id="horas" name="horas" min="0" max="23" value=")rawliteral" + String(horario.horas_ativacao) + R"rawliteral(">
      <label for="minutos">Minutos:</label>
      <input type="number" id="minutos" name="minutos" min="0" max="59" value=")rawliteral" + String(horario.minutos_ativacao) + R"rawliteral(">
      <button onclick="setHorario()">Definir Horário</button>
    </div>
  </div>

  <script>
    function setHorario() {
      const horas = document.getElementById('horas').value;
      const minutos = document.getElementById('minutos').value;
      fetch(`/set?hours=${horas}&minutes=${minutos}`)
        .then(response => response.text())
        .then(data => alert(data));
    }

    // receba o ajax
    function atualizarProgresso() {
      fetch("/progress")
        .then(response => response.json())
        .then(data => {
          const progressBar = document.getElementById("progress-bar");
          progressBar.style.width = data.percent + "%";
          progressBar.textContent = data.percent + "%";
        });
    }

    // Atualiza a barra de progresso a cada 5 segundos
    setInterval(atualizarProgresso, 5000);
  </script>
</body>
</html>
  )rawliteral";

  server.send(200, "text/html", html);
}


void handleProgress() {
  calculaRacao();
  String json = "{\"percent\": " + String(percent_racao) + "}";
  server.send(200, "application/json", json);
}


void handleSet() {
  if (server.hasArg("hours") && server.hasArg("minutes")) {
    int horas = server.arg("hours").toInt();
    int minutos = server.arg("minutes").toInt();
    setHorario(horas, minutos);

    server.send(200, "text/html", "<p>Horário atualizado para " + String(horas) + ":" + String(minutos) + "</p>");
  } else {
    server.send(400, "text/html", "<p>Parâmetros inválidos. Use ?hours=HH&minutes=MM</p>");
  }
}

void setup() {
  Serial.begin(9600);
  setupWiFi();

  pinMode(trig_pin, OUTPUT);
  pinMode(echo_pin, INPUT);

  Servo1.attach(pino_servo);
  fecha_servo();

  timeClient.begin();
  server.on("/", handleRoot);
  server.on("/set", handleSet);
  server.on("/progress", handleProgress); 
  server.begin();
  Serial.println("Servidor iniciado!");
}

void loop() {
  server.handleClient();
  timeClient.update();
  calculaRacao();

  int horas = timeClient.getHours();
  int minutos = timeClient.getMinutes();

  if (horas == horario.horas_ativacao && minutos == horario.minutos_ativacao) {
    abre_servo();
    delay(800);
    fecha_servo();
    delay(60000); 
  }
}
