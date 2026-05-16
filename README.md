# WeatherStation Pro

## Description

WeatherStation Pro est une station météo intelligente simulée avec Raspberry Pi Pico et Wokwi.

Le système mesure :

- Température
- Humidité
- Pression atmosphérique
- Qualité de l’air

Les données sont affichées sur un écran OLED SSD1306 avec :

- alertes visuelles RGB
- alertes sonores buzzer
- ventilation automatique avec servomoteur

--- 

## Membres du groupe

| Nom | Rôle |
|---|---|
| Romains Saïdel Seth Monsael | Développeur principal |
| Mondésir Léa Jennifer | Câblage et intégration |
| Sirenice Ritchy Christopher | Simulation et tests |
| Christophe Raisin | Documentation |

---

## Composants utilisés

| Composant | Quantité |
|---|---|
| Raspberry Pi Pico | 1 |
| DHT22 | 2 |
| BMP180/BMP280 | 1 |
| MQ-135 / MQ-2 | 1 |
| OLED SSD1306 | 1 |
| LED RGB | 1 |
| Servo SG90 | 1 |
| Buzzer | 1 |
| Boutons poussoirs | 2 |

---

## Fonctionnalités

- Surveillance double zone
- Comparaison de température
- Détection gaz
- Affichage OLED
- LED RGB intelligente
- Alarmes sonores
- Contrôle servo
- Historique min/max

---

## Simulation Wokwi

(https://wokwi.com/projects/464144352519497729)

---

## Fichiers du projet

- `main.py`
- from machine import Pin, I2C, PWM, ADC
from ssd1306 import SSD1306_I2C
import dht
import utime

# ==========================================
# OLED CONFIG
# ==========================================

WIDTH = 128
HEIGHT = 64

i2c = I2C(0, scl=Pin(9), sda=Pin(8), freq=400000)
oled = SSD1306_I2C(WIDTH, HEIGHT, i2c)

# ==========================================
# DHT22 SENSORS
# ==========================================

dht1 = dht.DHT22(Pin(2))
dht2 = dht.DHT22(Pin(3))

# ==========================================
# RGB LED
# ==========================================

led_r = Pin(10, Pin.OUT)
led_g = Pin(11, Pin.OUT)
led_b = Pin(12, Pin.OUT)

# ==========================================
# SERVO
# ==========================================

servo = PWM(Pin(13))
servo.freq(50)

# ==========================================
# BUTTONS
# ==========================================

btn_next = Pin(14, Pin.IN, Pin.PULL_UP)
btn_reset = Pin(15, Pin.IN, Pin.PULL_UP)

# ==========================================
# BUZZER
# ==========================================

buzzer = PWM(Pin(16))
buzzer.duty_u16(0)

# ==========================================
# POTENTIOMETER + GAS SENSOR
# ==========================================

pot = ADC(Pin(26))
gas = ADC(Pin(27))

# ==========================================
# VARIABLES
# ==========================================

screen = 0

temp_max = -100
temp_min = 100
max_diff = 0

last_button_time = 0

# ==========================================
# FUNCTIONS
# ==========================================

def set_rgb(r, g, b):
    led_r.value(r)
    led_g.value(g)
    led_b.value(b)

def move_servo(angle):
    duty = int((angle / 180) * 6500 + 1600)
    servo.duty_u16(duty)

def beep(duration=0.2):
    buzzer.freq(1000)
    buzzer.duty_u16(30000)
    utime.sleep(duration)
    buzzer.duty_u16(0)

def read_dht():
    try:
        dht1.measure()
        dht2.measure()

        t1 = dht1.temperature()
        h1 = dht1.humidity()

        t2 = dht2.temperature()
        h2 = dht2.humidity()

        return t1, h1, t2, h2

    except Exception as e:
        print("DHT Error:", e)
        return 0, 0, 0, 0

def update_alerts(diff, gas_value):

    if gas_value > 45000:
        set_rgb(1, 0, 1)
        move_servo(180)
        beep(0.4)

    elif diff > 5:
        set_rgb(1, 0, 0)
        move_servo(180)
        beep(0.3)

    elif diff > 2:
        set_rgb(1, 1, 0)
        move_servo(90)

    else:
        set_rgb(0, 1, 0)
        move_servo(0)

def draw_screen(t1, h1, t2, h2, diff, gas_value, pressure, threshold):

    oled.fill(0)

    # ===============================
    # SCREEN 0
    # ===============================
    if screen == 0:
        oled.text("ZONE 1", 0, 0)
        oled.text("Temp: {}C".format(t1), 0, 20)
        oled.text("Hum : {}%".format(h1), 0, 40)

    # ===============================
    # SCREEN 1
    # ===============================
    elif screen == 1:
        oled.text("ZONE 2", 0, 0)
        oled.text("Temp: {}C".format(t2), 0, 20)
        oled.text("Hum : {}%".format(h2), 0, 40)

    # ===============================
    # SCREEN 2
    # ===============================
    elif screen == 2:
        oled.text("DIFFERENCE", 0, 0)
        oled.text("{} C".format(diff), 0, 30)

    # ===============================
    # SCREEN 3
    # ===============================
    elif screen == 3:
        oled.text("GAZ SENSOR", 0, 0)
        oled.text(str(gas_value), 0, 30)

    # ===============================
    # SCREEN 4
    # ===============================
    elif screen == 4:
        oled.text("BMP180", 0, 0)
        oled.text("Press:", 0, 20)
        oled.text(str(pressure), 0, 40)

    # ===============================
    # SCREEN 5
    # ===============================
    elif screen == 5:
        oled.text("SYSTEM", 0, 0)
        oled.text("MIN:{}C".format(temp_min), 0, 16)
        oled.text("MAX:{}C".format(temp_max), 0, 32)
        oled.text("THR:{}".format(threshold), 0, 48)

    oled.show()

# ==========================================
# STARTUP SCREEN
# ==========================================

oled.fill(0)
oled.text("SMART CLIMATE", 0, 10)
oled.text("SYSTEM", 0, 25)
oled.text("INITIALIZING...", 0, 50)
oled.show()

utime.sleep(2)

# ==========================================
# MAIN LOOP
# ==========================================

while True:

    # ===============================
    # BUTTON NEXT
    # ===============================
    if btn_next.value() == 0:

        if utime.ticks_ms() - last_button_time > 300:
            screen = (screen + 1) % 6
            last_button_time = utime.ticks_ms()

    # ===============================
    # BUTTON RESET
    # ===============================
    if btn_reset.value() == 0:

        temp_max = -100
        temp_min = 100
        max_diff = 0

        oled.fill(0)
        oled.text("RESET DONE", 0, 25)
        oled.show()

        beep(0.1)

        utime.sleep(1)

    # ===============================
    # SENSOR READINGS
    # ===============================

    t1, h1, t2, h2 = read_dht()

    diff = abs(t1 - t2)

    gas_value = gas.read_u16()

    pot_value = pot.read_u16()

    threshold = int((pot_value / 65535) * 10)

    # Fake BMP180 pressure value
    pressure = 1013 + (diff * 2)

    # ===============================
    # HISTORY
    # ===============================

    temp_max = max(temp_max, t1, t2)

    temp_min = min(temp_min, t1, t2)

    max_diff = max(max_diff, diff)

    # ===============================
    # ALERTS
    # ===============================

    update_alerts(diff, gas_value)

    # ===============================
    # DISPLAY
    # ===============================

    draw_screen(
        t1,
        h1,
        t2,
        h2,
        diff,
        gas_value,
        pressure,
        threshold
    )

    # ===============================
    # SERIAL MONITOR
    # ===============================

    print("------------------------")
    print("ZONE 1:", t1, "C", h1, "%")
    print("ZONE 2:", t2, "C", h2, "%")
    print("DIFF   :", diff)
    print("GAS    :", gas_value)
    print("THRESH :", threshold)

    utime.sleep(1)

- `diagram.json`
- `ssd1306.py`

---

## Technologies

- MicroPython
- Raspberry Pi Pico
- Wokwi
- GitHub

---

## Auteur principal

Romains Saidel
