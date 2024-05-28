 #include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// Replace with your network credentials
const char* ssid = "project";///wifi name 
const char* password = "project147";//password

// DHT settings
#define DHTPIN D4 //dht 
#define DHTTYPE DHT22

// OLED settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// LED pin
#define LEDPIN D3  /relay or led connection

DHT dht(DHTPIN, DHTTYPE);
ESP8266WebServer server(80);

bool ledState = false;

void setup() {
  Serial.begin(115200);
  delay(10);

  // Initialize DHT sensor
  dht.begin();

  // Initialize OLED display
 if(!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println(F("SSD1306 allocation failed"));
    for(;;); // Don't proceed, loop forever
  }
  display.display();
  delay(2000);
  display.clearDisplay();
  
  // Initialize LED pin
  pinMode(LEDPIN, OUTPUT);
  digitalWrite(LEDPIN, LOW);

  // Connect to Wi-Fi
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());

  // Display IP address on OLED
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("IP Address:");
  display.setCursor(0, 10);
  display.print(WiFi.localIP());
  display.display();

  // Define web server routes
  server.on("/", handleRoot);
  server.on("/temperature", handleTemperature);
  server.on("/humidity", handleHumidity);
  server.on("/gas", handleGas);
  server.on("/led/on", handleLedOn);
  server.on("/led/off", handleLedOff);
  server.onNotFound(handleNotFound);

  // Start the server
  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();

  // Read sensor values
  float temperature = dht.readTemperature();
  float humidity = dht.readHumidity();
  int gasValue = analogRead(A0);

  // Display sensor values on OLED
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("IP: ");
  display.print(WiFi.localIP());
  display.setCursor(0, 10);
  display.print("Temp: ");
  display.print(temperature);
  display.println(" *C");
  display.setCursor(0, 20);
  display.print("Humidity: ");
  display.print(humidity);
  display.println(" %");
  display.setCursor(0, 30);
  display.print("Gas: ");
  display.print(gasValue);
  display.display();

  delay(2000);
}

void handleRoot() {
  String html = "<!DOCTYPE html><html><head>";
  html += "<title>WeMos D1 Mini Web Server</title>";
  html += "<style>";
  html += "body { font-family: Arial, sans-serif; background-color: #f0f0f0; color: #333; }";
  html += "h1 { color: #0066cc; text-align: center; }";
  html += ".container { max-width: 600px; margin: 0 auto; padding: 20px; text-align: center; }";
  html += ".button { display: inline-block; margin: 10px; padding: 10px 20px; text-decoration: none; color: white; background-color: #0066cc; border-radius: 5px; }";
  html += ".button:hover { background-color: #005bb5; }";
  html += "#data { margin-top: 20px; }";
  html += "</style></head><body>";
  html += "<h1>WeMos D1 Mini Web Server</h1>";
  html += "<div class=\"container\">";
  html += "<div id=\"data\">";
  html += "<p>Temperature: <span id=\"temperature\">--</span> &deg;C</p>";
  html += "<p>Humidity: <span id=\"humidity\">--</span> %</p>";
  html += "<p>Gas Level: <span id=\"gas\">--</span></p>";
  html += "</div>";
  html += "<a href=\"#\" class=\"button\" onclick=\"toggleLED('on')\">Turn LED On</a>";
  html += "<a href=\"#\" class=\"button\" onclick=\"toggleLED('off')\">Turn LED Off</a>";
  html += "</div>";
  html += "<script>";
  html += "function fetchData() {";
  html += "  fetch('/temperature').then(response => response.text()).then(data => document.getElementById('temperature').innerText = data);";
  html += "  fetch('/humidity').then(response => response.text()).then(data => document.getElementById('humidity').innerText = data);";
  html += "  fetch('/gas').then(response => response.text()).then(data => document.getElementById('gas').innerText = data);";
  html += "}";
  html += "function toggleLED(state) {";
  html += "  fetch('/led/' + state).then(response => response.text()).then(alert);";
  html += "}";
  html += "setInterval(fetchData, 5000);"; // Update data every 5 seconds
  html += "fetchData();"; // Initial fetch
  html += "</script>";
  html += "</body></html>";
  server.send(200, "text/html", html);
}

void handleTemperature() {
  float temperature = dht.readTemperature();
  if (isnan(temperature)) {
    server.send(500, "text/plain", "Failed to read from DHT sensor!");
  } else {
    server.send(200, "text/plain", String(temperature));
  }
}

void handleHumidity() {
  float humidity = dht.readHumidity();
  if (isnan(humidity)) {
    server.send(500, "text/plain", "Failed to read from DHT sensor!");
  } else {
    server.send(200, "text/plain", String(humidity));
  }
}

void handleGas() {
  int gasValue = analogRead(A0);//gas sensor connection
  server.send(200, "text/plain", String(gasValue));
}

void handleLedOn() {
  digitalWrite(LEDPIN, HIGH);
  ledState = true;
  server.send(200, "text/plain", "LED is now ON");
}

void handleLedOff() {
  digitalWrite(LEDPIN, LOW);
  ledState = false;
  server.send(200, "text/plain", "LED is now OFF");
}

void handleNotFound() {
  server.send(404, "text/plain", "404: Not Found");
}
