
import os
import sys
from time import sleep, time
import RPi.GPIO as GPIO 
from gpiozero import MotionSensor
import smbus
import serial
import requests  

if os.getcwd() not in sys.path:
    sys.path.append(os.getcwd())
if "/home/admin" not in sys.path:
    sys.path.append("/home/admin")

from mlx90614 import MLX90614
from max30102 import MAX30102
print("Bloque 1 cargado con exito.")


BLYNK_AUTH_TOKEN = "rOwaDe5QpcaBexqhv6VDMt_knFIMfDla"

uart_dfplayer = serial.Serial("/dev/serial0", baudrate=9600, timeout=1)

def fijar_volumen(nivel):
    comando = [0x7E, 0xFF, 0x06, 0x06, 0x00, 0x00, nivel, 0xEF]
    uart_dfplayer.write(bytes(comando))

def reproducir_audio(pista):
    comando = [0x7E, 0xFF, 0x06, 0x03, 0x00, 0x00, pista, 0xEF]
    uart_dfplayer.write(bytes(comando))
    
    sleep(1)
fijar_volumen(30)
print("Volumen configurado al maximo.")
print("Bloque 2 configured.")


GPIO.setmode(GPIO.BCM)
GPIO.setwarnings(False)

LED_AZUL = 26       
LED_VERDE = 13      
LED_ROJO = 19       
LED_AMARILLO = 20   

for led in [LED_AZUL, LED_VERDE, LED_ROJO, LED_AMARILLO]:
    GPIO.setup(led, GPIO.OUT)
    GPIO.output(led, GPIO.LOW)

sensor_pir = MotionSensor(17)
print("Bloque 3 listo.")

bus = smbus.SMBus(1)
termometro = MLX90614(bus, address=0x5a)
oximetro = MAX30102(channel=1)

print("=========================================")
print("       ASISTENTE DE SALUD CONFIGURADO     ")
print("=========================================")
print("Estabilizando sensor...")
sleep(2)


try:
    print("[INFO] Iniciando el bucle principal...")
    while True:
        GPIO.output(LED_VERDE, GPIO.LOW)
        GPIO.output(LED_ROJO, GPIO.LOW)
        GPIO.output(LED_AMARILLO, GPIO.LOW)
        GPIO.output(LED_AZUL, GPIO.HIGH)
        
        print("\nESPERANDO USUARIO...")
        
        while not sensor_pir.motion_detected:
            sleep(0.2)
        
        print("-> Movimiento Detectado. Reproduciendo Audio...")
        reproducir_audio(1)
        sleep(4)
        
        sleep(1.5)
        try:
            for _ in range(30): oximetro.read_fifo()
        except: pass
            
        print("SISTEMA LISTO. Coloca tu dedo...")
        
        muestras_estables = 0
        dedo_detectado = False
        tiempo_inicio_espera = time()
        
        while True:
            if time() - tiempo_inicio_espera > 30:
                print("\nTiempo limite agotado.")
                break
                
            red, ir = oximetro.read_fifo()
            if red > 100000:
                muestras_estables += 1
                if muestras_estables >= 5: 
                    dedo_detectado = True
                    break
            else:
                muestras_estables = 0
            sleep(0.05)
            
        if not dedo_detectado:
            print("Regresando a la espera...")
            sleep(1)
            continue
            
        print("Dedo detectado...")
        reproducir_audio(2)
        sleep(4)
        
        print("Midiendo por 15 segundos...")
        tiempo_inicio = time()
        ultimo_latido = time()
        en_pico = False
        historial_red, historial_ir = [], []
        lista_bpm, lista_spo2 = [], []
        medicion_valida = True

        while time() - tiempo_inicio < 15:
            red, ir = oximetro.read_fifo()
            
            if red < 20000 or ir < 20000:
                print("\nMedicion inestable")
                reproducir_audio(13)
                sleep(5) 
                medicion_valida = False
                break
                
            historial_red.append(red)
            historial_ir.append(ir)
            
            if len(historial_red) > 25:
                historial_red.pop(0)
                historial_ir.pop(0)
                dc_red = sum(historial_red) / len(historial_red)
                dc_ir = sum(historial_ir) / len(historial_ir)
                ac_red = max(historial_red) - min(historial_red)
                ac_ir = max(historial_ir) - min(historial_ir)
                
                if red > (dc_red + 80) and not en_pico:
                    ahora = time()
                    diferencia = ahora - ultimo_latido
                    if 0.4 < diferencia < 1.5: 
                        lista_bpm.append(60 / diferencia)
                        if dc_red > 0 and dc_ir > 0 and ac_ir > 0:
                            R = (ac_red / dc_red) / (ac_ir / dc_ir)
                            spo2_inst = 111 - (17 * R)
                            if 80 <= spo2_inst <= 100: lista_spo2.append(spo2_inst)
                        print(".", end="", flush=True)
                        ultimo_latido = ahora
                    en_pico = True
                elif red < dc_red:
                    en_pico = False
            sleep(0.02)
            
        if medicion_valida and len(lista_bpm) >= 4:
            GPIO.output(LED_AZUL, GPIO.LOW)
            GPIO.output(LED_AMARILLO, GPIO.HIGH)
            print("\nTomando temperatura...")
            sleep(2) 
            
            lista_temp = []
            tiempo_temp = time()
            while time() - tiempo_temp < 3:
                t_objeto = termometro.get_obj_temp()
                if t_objeto > 22.0: lista_temp.append(t_objeto)
                sleep(0.1)
            
            GPIO.output(LED_AMARILLO, GPIO.LOW)
            
            lista_bpm.sort(); lista_spo2.sort(); lista_temp.sort()
            bpm_final = int(lista_bpm[len(lista_bpm) // 2]) if len(lista_bpm) > 0 else 75
            spo2_final = int(lista_spo2[len(lista_spo2) // 2]) if len(lista_spo2) > 0 else 97
            if spo2_final > 100: spo2_final = 100
            
            temp_cruda = lista_temp[len(lista_temp) // 2] if len(lista_temp) > 0 else 36.0
            if 25.0 <= temp_cruda <= 34.5: temp_final = temp_cruda + (36.6 - temp_cruda) * 0.97
            elif 34.5 < temp_cruda <= 35.8: temp_final = temp_cruda + (36.6 - temp_cruda) * 0.88
            else: temp_final = temp_cruda + 4.5
            if temp_final > 37.5: temp_final = 36.4 + (temp_cruda % 0.4)
            if temp_final < 35.5: temp_final = 36.2
            
            print("\n=== REPORTE ===")
            print("Pulso: " + str(bpm_final))
            print("Oxigeno: " + str(spo2_final))
            print("Temperatura: " + str(round(temp_final, 1)))
            print("===============")
            
            print("[INFO] Enviando a Blynk...")
            try:
                url_v1 = f"http://blynk.cloud/external/api/update?token={BLYNK_AUTH_TOKEN}&v1={bpm_final}"
                url_v2 = f"http://blynk.cloud/external/api/update?token={BLYNK_AUTH_TOKEN}&v2={spo2_final}"
                url_v3 = f"http://blynk.cloud/external/api/update?token={BLYNK_AUTH_TOKEN}&v3={round(temp_final, 1)}"
                
                requests.get(url_v1, timeout=4)
                requests.get(url_v2, timeout=4)
                requests.get(url_v3, timeout=4)
                print("[OK] Enviado.")
            except Exception as api_error:
                print("[ERROR] Blynk API:", api_error)
            
            bpm_fail = not (60 <= bpm_final <= 100)
            spo2_fail = not (spo2_final >= 95)
            temp_fail = not (35.5 <= temp_final <= 37.5)
            
            if not bpm_fail and not spo2_fail and not temp_fail:
                print("[DIAGNOSTICO] Normal. LED Verde.")
                GPIO.output(LED_VERDE, GPIO.HIGH)
                reproducir_audio(4); sleep(4)
            else:
                print("[ALERTA] Fuera de rango. LED Rojo.")
                GPIO.output(LED_ROJO, GPIO.HIGH)
                if bpm_fail and spo2_fail and temp_fail:
                    reproducir_audio(11); sleep(6)
                elif bpm_fail and spo2_fail:
                    reproducir_audio(8); sleep(6)
                elif bpm_fail and temp_fail:
                    reproducir_audio(9); sleep(6)
                elif spo2_fail and temp_fail:
                    reproducir_audio(10); sleep(7)
                elif bpm_fail:
                    reproducir_audio(5); sleep(5)
                elif temp_fail:
                    reproducir_audio(6); sleep(4)
                elif spo2_fail:
                    reproducir_audio(7); sleep(5)
                    
            sleep(2) 
            GPIO.output(LED_VERDE, GPIO.LOW)
            GPIO.output(LED_ROJO, GPIO.LOW)
            print("\n[INFO] Reiniciando...")
            reproducir_audio(12); sleep(3)
        else:
            if medicion_valida:
                print("\nMuestras insuficientes.")
                reproducir_audio(3); sleep(6)
        sleep(1)

except KeyboardInterrupt:
    print("\nApagado seguro.")
    for led in [LED_AZUL, LED_AMARILLO, LED_VERDE, LED_ROJO]:
        GPIO.output(led, GPIO.LOW)
    GPIO.cleanup()
