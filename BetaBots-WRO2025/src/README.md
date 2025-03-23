Control software
====
//definición de los  pines

// Sensores ultrasónicos
#define TRIG_FRONT 9
#define ECHO_FRONT 10
#define TRIG_BACK 11
#define ECHO_BACK 12

// Sensores IR
#define IR_FRONT_LEFT A0
#define IR_FRONT_RIGHT A1
#define IR_SIDE_LEFT A2
#define IR_SIDE_RIGHT A3
#define IR_BACK_LEFT A4
#define IR_BACK_RIGHT A5

// Sensor de color TCS3200
#define S2 2
#define S3 3
#define SENSOR_OUT 4

// Puente H - para el Control de Motor 
#define MOTOR_EN 5   // PWM para velocidad
#define MOTOR_IN1 6  // Dirección motor
#define MOTOR_IN2 7  // Dirección motor

// Servo motor de dirección
#define SERVO_DIRECCION 8

Servo direccion;

// ------------------- ESTADOS -------------------
enum Estado { MOVIENDO, EVITANDO, RODEANDO_SENAL, FINAL_TRAMO, ESTACIONANDO, DETENIDO };
Estado estado = MOVIENDO;

// ------------------- FUNCIONES DE SENSORES -------------------

// Función para medir distancia las con sensores ultrasónicos
int medirDistancia(int trigPin, int echoPin) {
    digitalWrite(trigPin, LOW);
    delayMicroseconds(2);
    digitalWrite(trigPin, HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPin, LOW);

    int duracion = pulseIn(echoPin, HIGH, 30000);
    int distancia = (duracion * 0.0343) / 2;

    return (distancia > 0) ? distancia : 100; // Si no mide bien, asumir 100cm
}

// Función para detectar los obstáculos con los sensores IR
bool detectarIR(int pin) {
    // Se asume que el IR devuelve LOW si hay un obstáculo
    return (digitalRead(pin) == LOW);
}

// Función para detección de color con el sensor TCS3200
String detectarColor() {
    digitalWrite(S2, LOW);
    digitalWrite(S3, LOW);
    delay(10);
    int rojo = pulseIn(SENSOR_OUT, LOW);

    digitalWrite(S2, HIGH);
    digitalWrite(S3, HIGH);
    delay(10);
    int verde = pulseIn(SENSOR_OUT, LOW);

    digitalWrite(S2, LOW);
    digitalWrite(S3, HIGH);
    delay(10);
    int azul = pulseIn(SENSOR_OUT, LOW);

         //ajustar con pruebas 
    if (rojo < verde && rojo < azul) return "ROJO";
    if (verde < rojo && verde < azul) return "VERDE";

    //detectamos "MAGENTA" si rojo y azul son pequeños, y verde es grande (ejemplo)
          //ajustar según pruebas 
    if (rojo < 500 && azul < 500 && verde > 1000) return "MAGENTA";

    return "OTRO";
}

// ------------------- FUNCIONES DE MOTORES -------------------
void avanzar(int velocidad = 150) {
    digitalWrite(MOTOR_IN1, HIGH);
    digitalWrite(MOTOR_IN2, LOW);
    analogWrite(MOTOR_EN, velocidad);
}

void retroceder(int velocidad = 150) {
    digitalWrite(MOTOR_IN1, LOW);
    digitalWrite(MOTOR_IN2, HIGH);
    analogWrite(MOTOR_EN, velocidad);
}

void detener() {
    digitalWrite(MOTOR_IN1, LOW);
    digitalWrite(MOTOR_IN2, LOW);
    analogWrite(MOTOR_EN, 0);
}

// ------------------- FUNCIONES DE DIRECCIÓN (SERVO) -------------------
void girarIzquierda() { direccion.write(45); }
void girarDerecha() { direccion.write(135); }
void recto() { direccion.write(90); }

// --------------FUNCIÓN SETUP -------------------
void setup() {
    // Pines ultrasónicos
    pinMode(TRIG_FRONT, OUTPUT);
    pinMode(ECHO_FRONT, INPUT);
    pinMode(TRIG_BACK, OUTPUT);
    pinMode(ECHO_BACK, INPUT);

    // Pines IR
    pinMode(IR_FRONT_LEFT, INPUT);
    pinMode(IR_FRONT_RIGHT, INPUT);
    pinMode(IR_SIDE_LEFT, INPUT);
    pinMode(IR_SIDE_RIGHT, INPUT);
    pinMode(IR_BACK_LEFT, INPUT);
    pinMode(IR_BACK_RIGHT, INPUT);

    // Pines puente H
    pinMode(MOTOR_EN, OUTPUT);
    pinMode(MOTOR_IN1, OUTPUT);
    pinMode(MOTOR_IN2, OUTPUT);

    // Pines TCS3200
    pinMode(S2, OUTPUT);
    pinMode(S3, OUTPUT);
    pinMode(SENSOR_OUT, INPUT);

    // Servo
    direccion.attach(SERVO_DIRECCION);
    direccion.write(90);

    Serial.begin(9600);
}

// -------------FUNCIÓN LOOP -------------------
void loop() {
    // Lecturas de sensores
    int distanciaFront = medirDistancia(TRIG_FRONT, ECHO_FRONT);
    int distanciaBack = medirDistancia(TRIG_BACK, ECHO_BACK);

    bool obstaculoFrente = detectarIR(IR_FRONT_LEFT) || detectarIR(IR_FRONT_RIGHT);
    bool obstaculoLateral = detectarIR(IR_SIDE_LEFT) || detectarIR(IR_SIDE_RIGHT);
    bool obstaculoAtras  = detectarIR(IR_BACK_LEFT)  || detectarIR(IR_BACK_RIGHT);

    String color = detectarColor();

    Serial.print("Frente: "); Serial.print(distanciaFront);
    Serial.print(" cm | Atras: "); Serial.print(distanciaBack);
    Serial.print(" cm | Color: "); Serial.println(color);

    switch (estado) {
        case MOVIENDO:
            // 1. Evitar obstaculos
            if (distanciaFront < 20 || obstaculoFrente) {
                detener();
                delay(500);
                girarDerecha();
                delay(800);
                recto();
                estado = EVITANDO;
            }
            // 2. Rodear señal ROJA
            else if (color == "ROJO") {
                detener();
                delay(500);
                girarDerecha();
                avanzar(150);
                delay(800);
                girarIzquierda();
                avanzar(150);
                delay(800);
                girarIzquierda();
                avanzar(150);
                delay(800);
                girarDerecha();
                recto();
                estado = MOVIENDO;
            }
            // 3. Rodear señal VERDE
            else if (color == "VERDE") {
                detener();
                delay(500);
                girarIzquierda();
                avanzar(150);
                delay(800);
                girarDerecha();
                avanzar(150);
                delay(800);
                girarDerecha();
                avanzar(150);
                delay(800);
                girarIzquierda();
                recto();
                estado = MOVIENDO;
            }
            // 4. Detectar zona de estacionamiento (MAGENTA)
            else if (color == "MAGENTA") {
                detener();
                Serial.println("Entrando a estacionar...");
                estado = ESTACIONANDO;
            }
            // 5. Final del tramo
            else if (distanciaFront > 500) {
                detener();
                estado = FINAL_TRAMO;
            }
            // 6. Avanzar normal
            else {
                recto();
                avanzar(map(distanciaFront, 15, 100, 100, 255));
            }
            break;

        case EVITANDO:
            // Espera a que el obstaculo frontal desaparezca
            if (distanciaFront > 25 && !obstaculoFrente) {
                estado = MOVIENDO;
            }
            break;

        case FINAL_TRAMO:
            Serial.println("Final del tramo, girando...");
            if (!obstaculoLateral) {
                girarIzquierda();
            } else {
                girarDerecha();
            }
            delay(1000);
            recto();
            estado = MOVIENDO;
            break;

        // ------------------- ESTACIONANDO -------------------
        case ESTACIONANDO:
            // Ejemplo de maniobra:
            // 1) Gira un poco para acomodar
            girarDerecha();
            retroceder(100);
            delay(1000);

            // 2) Endereza y sigue retrocediendo
            recto();
            retroceder(80);
            delay(1500);

            // 3) Verifica si ya está bien posicionado
            if (distanciaBack < 15 || obstaculoAtras) {
                detener();
                Serial.println("Estacionado con éxito.");
                estado = DETENIDO; // O se queda estacionado
            }
            break;

        case DETENIDO:
            detener();
            Serial.println("Coche detenido.");
            break;
    }

    delay(100);
}

This directory must contain code for control software which is used by the vehicle to participate in the competition and which was developed by the participants.

All artifacts required to resolve dependencies and build the project must be included in this directory as well.