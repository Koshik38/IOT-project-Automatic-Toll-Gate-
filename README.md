#include <ESP8266WiFi.h>
#include <Servo.h>

Servo myservo;

// Pins
const int trigPin = D5;  // GPIO14
const int echoPin = D6;  // GPIO12
const int servoPin = D4; // GPIO2

// Wi-Fi AP credentials
const char* ssid = "TollGate_WiFi";
const char* password = "12345678";

WiFiServer server(80);

long duration;
int distanceCm;
bool autoMode = true; // automatic mode by default
unsigned long lastOpenMillis = 0;
const unsigned long openCooldown = 5000; // 5 seconds between auto triggers
IPAddress apIP; // store AP IP address

void setup() {
  Serial.begin(115200);
  myservo.attach(servoPin);
  myservo.write(0); // start closed

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Start Wi-Fi Access Point
  WiFi.softAP(ssid, password);
  apIP = WiFi.softAPIP();
  server.begin();

  Serial.println("AP started");
  Serial.print("Connect to: "); Serial.println(ssid);
  Serial.print("Password: "); Serial.println(password);
  Serial.print("AP IP address: "); Serial.println(apIP);
}

int readDistance() {
  // Trigger ultrasonic sensor
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long pulse = pulseIn(echoPin, HIGH, 30000); // 30ms timeout
  if (pulse == 0) return -1; // no reading
  int cm = pulse * 0.034 / 2;
  return cm;
}

String webpageHTML(bool mode, int lastDist) {
  String html = "<html><head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">";
  html += "<style>body{font-family:Arial;text-align:center;padding:20px}button{padding:12px 20px;margin:6px;font-size:18px}</style></head><body>";
  html += "<h2>IoT Toll Gate</h2>";
  html += "<p>IP Address: <b>" + apIP.toString() + "</b></p>";
  html += "<p>Mode: <b>" + String(mode ? "Automatic" : "Manual") + "</b></p>";
  html += "<p>Last distance: " + String(lastDist) + " cm</p>";
  html += "<a href='/open'><button>Open</button></a>";
  html += "<a href='/close'><button>Close</button></a><br>";
  html += "<a href='/auto'><button>Auto Mode</button></a>";
  html += "</body></html>";
  return html;
}

void handleClient(WiFiClient &client, int lastDist) {
  String req = client.readStringUntil('\r');
  client.flush();

  if (req.indexOf("/open") != -1) {
    myservo.write(90);
    autoMode = false;
    Serial.println("Gate opened manually");
  } else if (req.indexOf("/close") != -1) {
    myservo.write(0);
    autoMode = false;
    Serial.println("Gate closed manually");
  } else if (req.indexOf("/auto") != -1) {
    autoMode = true;
    Serial.println("Switched to automatic mode");
  }

  String html = webpageHTML(autoMode, lastDist);
  client.print("HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nConnection: close\r\n\r\n");
  client.print(html);
  delay(1);
  client.stop();
}

void loop() {
  static int lastDist = -1;

  // Check if a client has connected
  WiFiClient client = server.available();
  if (client) {
    handleClient(client, lastDist);
  }

  if (autoMode) {
    int dist = readDistance();
    if (dist > 0) {
      lastDist = dist;
      Serial.print("Distance measured: ");
      Serial.print(dist);
      Serial.println(" cm");
    }

    // Open gate automatically if object closer than 15cm and cooldown passed
    if (dist > 0 && dist < 15 && (millis() - lastOpenMillis) > openCooldown) {
      lastOpenMillis = millis();
      myservo.write(90); // open
      Serial.println("Gate opened automatically");
      delay(3000);
      myservo.write(0);  // close
      Serial.println("Gate closed automatically");
    }
  }

  delay(100);
}
