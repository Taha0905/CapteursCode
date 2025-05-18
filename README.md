# CapteursCode
import Adafruit_DHT
import struct
import serial
import time
import spidev
import math
import paho.mqtt.client as mqtt

# -------------------------------
# CONFIGURATIONS
# -------------------------------

# Capteur DHT (humidité et température)
DHT_SENSOR = Adafruit_DHT.DHT11
DHT_PIN = 4  # GPIO4 pour le DHT11
mqtt_broker_adress = "172.31.254.254"

# Capteur SDS011 (particules fines)
ser = serial.Serial('/dev/ttyUSB0', baudrate=9600, timeout=2)  # Port USB

# Capteur Gravity Sound Level Meter (via MCP3004 - canal 0)
spi = spidev.SpiDev()
spi.open(0, 0)  # Bus 0, périphérique CE0
spi.max_speed_hz = 1350000  # Vitesse SPI


# -------------------------------
# FONCTIONS
# -------------------------------

def capteurDHT():
    """Lit la température et l'humidité du capteur DHT11."""
    humidity, temperature = Adafruit_DHT.read_retry(DHT_SENSOR, DHT_PIN)
    if humidity is not None and temperature is not None:
        return round(humidity, 1), round(temperature, 2)
    return None, None

def capteurSDS():
    """Lit les données PM2.5 et PM10 du capteur SDS011."""
    data = ser.read(10)
    if len(data) == 10 and data[0] == 0xAA and data[1] == 0xC0 and data[9] == 0xAB:
        pm25 = struct.unpack('<H', data[2:4])[0] / 10.0
        pm10 = struct.unpack('<H', data[4:6])[0] / 10.0
        return pm25, pm10
    return None, None


def lire_mcp3004(channel):
    """Lit une valeur analogique du MCP3004 (canal 0 à 3)."""
    if not 0 <= channel <= 3:
        raise ValueError("Le canal doit être entre 0 et 3")
    adc = spi.xfer2([1, (8 + channel) << 4, 0])
    valeur = ((adc[1] & 3) << 8) + adc[2]
    return valeur


def lire_mcp3004_moyenne(channel, nb_lectures=10):
    """Lit plusieurs fois le canal analogique et retourne la moyenne."""
    total = 0
    for _ in range(nb_lectures):
        total += lire_mcp3004(channel)
        time.sleep(0.01)  # pause courte entre les lectures (10ms)
    moyenne = total / nb_lectures
    return moyenne


def convertir_en_db(tension):
    """Convertit la tension lue en dB SPL."""
    try:
        v_ref = 0.00631  # tension de référence correspondant à ~0 dB SPL
        if tension > 0:
            db = 20 * math.log10(tension / v_ref)
            return round(db, 1)
        else:
            return 0
    except:
        return 0


# -------------------------------
# BOUCLE PRINCIPALE
# -------------------------------

def main():
    try:
        mqtt_topic_humidity = "climat/humidity"
        mqtt_topic_temperature = "climat/temperature"
        mqtt_topic_sound_level = "climat/decibels"
        mqtt_topic_air_quality_10 = "climat/air_quality_10"
        mqtt_topic_air_quality_25 = "climat/air_quality_25"
        client = mqtt.Client("capteurs_parking")
        client.connect(mqtt_broker_adress)
        while True:
            humidity, temperature = capteurDHT()
            pm25, pm10 = capteurSDS()
            valeur_son = lire_mcp3004_moyenne(0, 10)
            tension_son = (valeur_son * 3.3) / 1023
            db_son = convertir_en_db(tension_son)

            print("======================================")
            if humidity is not None and temperature is not None:
                print(f"Température : {temperature}°C | Humidité : {humidity}%")
            else:
                print("⚠️ Erreur de lecture DHT11.")

            if pm25 is not None and pm10 is not None:
                print(f" PM2.5 : {pm25} µg/m3 | PM10 : {pm10} µg/m3")
            else:
                print("️ Erreur de lecture SDS011.")

            print(f" Niveau sonore : {db_son} dB SPL (Tension : {tension_son:.2f} V)")
            print("======================================\n")
            
            client.publish(mqtt_topic_humidity, humidity)
            client.publish(mqtt_topic_temperature, temperature)
            client.publish(mqtt_topic_sound_level, db_son)
            client.publish(mqtt_topic_air_quality_10, pm10)
            client.publish(mqtt_topic_air_quality_25, pm25)

            time.sleep(0.5)

    except KeyboardInterrupt:
        print("Programme arrêté.")
        spi.close()
        ser.close()

# -------------------------------
# LANCEMENT
# -------------------------------

if __name__ == '__main__':
    main()
