# INFORME 5
## Variabilidad de la Frecuencia Cardiaca usando la Transformada Wavelet 

En este informe de laboratorio se plantea como objetivo analizar la variabilidad de la frecuencia cardíaca (HRV) utilizando la transformada wavelet para identificar cambios en las frecuencias características y analizar la dinámica de esta.

El laboratorio se realizó tomando a un paciente en el cual se le realizaria un electrocardiograma (ECG), la señal cardiaca seria procesada por un sensor ECG y esta señal será adquirida a través de una blu phil (microcontrolador), sirviendo como un sistema de adquisición de datos.
En esta toma de datos el paciente se verá sometido a tres tipos de actividades distintas, la primera es estar una actividad de estrés o activiad fisica esto con el fin de activar el sistema simpático, aumentando la frecuencia. A continuación, se dispondría a el paciente en un estado de “normalidad” escuchando el ambiente y hablando con personas. Y por último en reposo, con la intención que el sistema parasimpático del cuerpo humano, bajara la frecuencia cardiaca. Esta medicion se dispondra de 5 minutos para cada tipo de actividad.
Todo esto con el fin de poder saber cómo estímulos externos activan los sistemas (simpático y parasimpático) y gracias a esta activación como afecta directamente a la variabilidad de la frecuencia cardiaca (HRV).
La HRV (Heart Rate Variability) o  en español la variabilidad de la frecuencia cardiaca, es la medida de las fluctuaciones en el tiempo entre latidos sucesivos, específicamente entre los picos R del complejo QRS en un electrocardiograma (intervalo R–R). Cuanto más variable es ese intervalo, más flexible y adaptativo es el sistema nervioso autónomo.

A continuación se vera este paso a paso en un diagrama de flujo explicando brevente que se hace en esta practica:

![Diagrama de flujo](https://github.com/user-attachments/assets/6641a73c-cfdb-49d3-99fb-ca7c38ec1a5f)
(se anexa el link para una mejor visualizacion: https://miro.com/welcomeonboard/cFJ1Wi8yKy8wOGtzbXZyZnlxeVI3Y2dWK0lIWUYyZUxXS1lrY0NTNmwwRjBFM1NQZ043am5uNEFJcVhWSXgxd1BLdDZOVE5ubXJqUVIwQytqWHo1QVd6RGxGb2tRUnJRdUQyR0RMVHNJL1JkQzliTnlBam1YbDlSY29xTHgrYmRNakdSWkpBejJWRjJhRnhhb1UwcS9BPT0hdjE=?share_link_id=211899885668)

Las frecuencias que se deben tener en cuenta deben ser las siguientes (con transformada de Wavelet):
- ULF (Ultra Low Frequency) < 0.003 Hz: Cambios de muy largo plazo.
- VLF (Very Low Frequency) 0.003 – 0.04 Hz Influencia hormonal, regulación de la temperatura corporal.
- LF (Low Frequency) 0.04 – 0.15 Hz Actividad simpática y parasimpática.
- HF (High Frequency) 0.15 – 0.4 Hz Actividad parasimpática, refleja la respiración.

 En los resultados esperamos que la HRV se vea satisfactoriamente los tres momentos de estudio el paciente en reposo, el paciente en estado normal y en paciente en una situación de estrés.

## Codigo de adquisicion de datos (matlab)
Antes de procesar la señal, se tuvo que adquirir dicha señal ECG, por lo tanto, se utilizo el siguiente codigo para la adquisicion de los datos del ECG:

##### limpieza del entorno y consola
en esta parte inicial limpiaremos la consola `clc`, borrar variables `clear all` y cierra ventanas`close all`.
```
clc; clear all; close all;
```
##### Cerrar puertos seriales abiertos anteriormente (si hay)
```
 if ~isempty(serialportlist)
        for p = serialportlist
            try
                clear(serialport(p)); % Intentar cerrar el puerto
            catch
                % Si da error (por ejemplo, ya cerrado), lo ignora
            end
        end
    end
```
Esto lo que hace es revisa si hay puertos abiertos y trata de cerrarlos para evitar conflictos al abrir uno nuevo. Ya que estos datos se estan capturando en tiempo real.
##### Configuracion de puerto serial
```
puerto    = 'COM5';
    baudios   = 115200;        
    sp        = serialport(puerto, baudios);
    sp.Timeout = 1;           
    flush(sp);                   % Limpiar datos previos del buffer
```
Nombre del puerto COM (ajustar según PC)`puerto`  en este caso es el puerto 5,` 'COM5';`.
La velocidad de transmision `  baudios  `.
Creacion de objeto serial a travez de `serialport(puerto, baudios);`.
Tiempo maximo de espera en segundos `sp.Timeout`
Limpia datos previos del buffer `flush(sp);`

##### Creacion de archivo para guardar los datos
```
 fname = 'PRUEBBA.txt';
    fid   = fopen(fname, 'w');  
    if fid == -1
        error('No se pudo abrir %s para escritura.', fname); 
    end
```
Es el nombre del archivo que se generara `fname `.
Esta funcion nos permite abrir el archivo en modo escritura ` fid   = fopen(fname, 'w');`.
Si falla, lanza un error gracias al if.

##### Parametros de adquisicion y grafica
 ```
 T_total    = 300;
    tStart     = tic;
    plotInterval = 0.1;
    lastPlot   = tic;
```
Duración de captura: 300 segundos (5 minutos).
Se usan cronómetros `(tic)` para medir tiempo de adquisición y para refrescar la gráfica cada 0.1 s.

##### Buffers para almacenar muestras en memoria y graficarlas
```
  bufSize    = 500;
    ventanaMM  = 5;
    datos      = zeros(1, bufSize, 'uint8');
    datosF     = zeros(1, bufSize);
```
 `bufSize` Es el tamaño del buffer de muestra.
 El tamaño de la ventana de media movil se da por `ventanaMM `.
La linea de codigo 2 es el buffer de datos crudos y la linea 4 es el buffer de datos filtrados
##### Preparacion de ventana de la grafica(figura)
```
    hFig = figure('Name','ECG 5 min','NumberTitle','off');
    hRaw  = plot(datos,'b','LineWidth',1); hold on;
    hFilt = plot(datosF,'r','LineWidth',1);
    ylim([0,255]); grid on;
    xlabel('Muestras'); ylabel('Valor (uint8)');
    hTitle = title('0 / 300 s','FontSize',12);
```
La funcion `figure()` nos permite crear la ventana, en esta ventana se mostrara la señal filtrada (en rojo) `   hFilt` y no filtrada (en azul) ` hRaw`.
##### Bucle principal de adquisicion 
```
while toc(tStart) < T_total
        nAvail = sp.NumBytesAvailable;
        if nAvail > 0
            chunk = read(sp, nAvail, 'uint8');
            fprintf(fid, '%u\n', chunk);


```
este bucle correra por 300 segundos es decir 5 minutos. Ademas de, Leer los datos disponibles (tipo uint8) en la funcion `read()`  y los guarda línea por línea en el archivo `fprintf()`.
```
 for y = chunk
                datos  = [datos(2:end), y];
            end
            datosF = movmean(datos, ventanaMM);
        end
```
Estas lineas de codigo nos permiten actualizar buffer de datos sin ningun filtrado, ademas de desplazar y agregar nueva muestra, y aplicar media movil.
##### Actualizar gráfica solo si pasó suficiente tiempo
```
if toc(lastPlot) > plotInterval
            set(hRaw,  'YData', datos);
            set(hFilt, 'YData', datosF);
            elapsed = toc(tStart);
            set(hTitle,'String', sprintf('%.1f / 300 s', elapsed));
            drawnow limitrate;
            lastPlot = tic;
        end
    end
```
En la linea 1 veremos la actualizacion de la señal sin filtrar y en la linea 2 veremos la actualizacion de la señal filtrada. Refresca la gráfica si ya pasó `plotInterval` (0.1 s).

##### Finalizar: cerrar archivo y liberar puerto
```
fclose(fid);
    clear sp;
    fprintf('→ Captura completa de 5 minutos guardada en %s\n', fname);
end
```
Cierra el archivo, libera el puerto serial y muestra un mensaje indicando que la captura ha finalizado.
## Procesamiento de la señal ECG
A continuacion se realizara el procesamiento y análisis de una señal ECG, incluyendo filtrado, detección de picos R y análisis de variabilidad del ritmo cardíaco (HRV) usando transformada wavelet.
desglosaremos el codigo a continuacion:
##### Importación de librerías
```
import numpy as np
import matplotlib.pyplot as plt
from scipy.signal import butter, filtfilt, find_peaks, cwt, morlet2
```
`numpy`: para operaciones numéricas.
`matplotlib.pyplot`: para graficar.
`scipy.signal`: contiene funciones de filtrado, detección de picos y análisis con wavelet.
##### Carga de la señal ECG
```
ecg = np.loadtxt("SARA01.txt")
ecg = ecg.astype(float)
```
`np.loadtxt()`: Carga datos desde un archivo de texto
`ecg.astype`: Convierte los datos a tipo float (reales con decimales)
##### Parámetros del filtro
```
fs = 250                  # Frecuencia de muestreo (Hz)
lowcut = 0.5
highcut = 40
order = 4
```
#####  Diseño del filtro IIR Butterworth pasa banda
```
def butter_bandpass(lowcut, highcut, fs, order):
    nyq = fs / 2
    low = lowcut / nyq
    high = highcut / nyq
    b, a = butter(order, [low, high], btype='band')
    return b, a
```
Esta función devuelve los coeficientes del filtro Butterworth.
`nyq`: frecuencia de Nyquist.
`butter`: diseña el filtro.
```
b, a = butter_bandpass(lowcut, highcut, fs, order)
```
Guarda los coeficientes numerador `b` y denominador `a` del filtro.
##### Mostrar ecuación en diferencias del filtro
```
print("Ecuación en diferencias del filtro:")
print("Salida[n] =", " + ".join([f"{b[i]:.4f}*x[n-{i}]" for i in range(len(b))]), 
      "-", " - ".join([f"{a[i]:.4f}*y[n-{i}]" for i in range(1, len(a))]))
```
Muestra cómo se ve la ecuación del filtro en forma discreta. siendo esta:


![image](https://github.com/user-attachments/assets/8224cc3a-85d8-43df-80a6-e2a9293611ff)


##### Aplicación del filtro
```
ecg_filt = filtfilt(b, a, ecg)
```
Aplica el filtro IIR pasa bandas.

##### Detección de picos R
```
peaks, _ = find_peaks(ecg_filt, distance=fs*0.6, height=np.mean(ecg_filt))
```
Detecta picos en la señal filtrada.
``distance=fs*0.6``: mínimo 0.6s entre picos (frecuencia cardíaca máxima de ~100 lpm).
`height`: umbral dinámico basado en la media.
```
r_times = peaks / fs
rr_intervalos = np.diff(r_times) * 1000  # En milisegundos
```
Calcula los tiempos y los intervalos RR (diferencias entre picos R), en milisegundos.

##### Análisis HRV en dominio del tiempo
```
rr_media = np.mean(rr_intervalos)
rr_desv_standar = np.std(rr_intervalos)
print(f"\nAnálisis HRV - Dominio del tiempo:")
print(f"Media RR: {rr_media:.2f} ms")
print(f"Desviación estándar (SDNN): {rr_std:.2f} ms")
```
Calcula la media y desviación estándar de los intervalos RR, que son medidas comunes de HRV.
##### Transformada Wavelet Continua Morlet
```
widths = np.arange(1, 128)  # escala de wavelet
wavelet = lambda M, s: morlet2(M, s, w=6)  # Define wavelet Morlet (es una funcion abreviada)
cwtmatr = cwt(rr_intervalos - np.mean(rr_intervalos), wavelet, widths)
```
Realiza análisis tiempo-frecuencia con wavelets sobre la serie RR (que es tiempo).
Se resta la media para centrar la señal.

### Visualizacion de la señal 
para corroborar que la toma de datos si fue exitosa se extrae de la señal de 5 minutos un segmento de 20 segundos, de esta forma confirmando que es una señal ECG.
```
duracion_segundos = 20
inicio_segundo = 110
inicio_muestra = int(inicio_segundo * fs)
fin_muestra = inicio_muestra + int(duracion_segundos * fs)
t = np.arange(len(ecg)) / fs
t_segmento = t[inicio_muestra:fin_muestra]
ecg_segmento = ecg[inicio_muestra:fin_muestra]
```
este segmento se saco a partir del segundo 110 buscando una muestra central de la señal ECG.
```
plt.figure(figsize=(10, 4))
plt.plot(t_segmento, ecg_segmento, label='ECG (segmento)')
plt.title(f'Segmento de la señal ECG: {inicio_segundo}s a {inicio_segundo + duracion_segundos}s')
plt.xlabel('Tiempo [s]')
plt.ylabel('mV')
plt.grid(True)
plt.legend()
plt.tight_layout()
plt.show()
```
Muestra gráficamente el segmento ECG.


![image](https://github.com/user-attachments/assets/6c8b4706-32d0-47cd-95f0-8e780422c0f7)


#####Señal no filtrada vs filtrada
```
plt.figure(figsize=(12, 4))
plt.plot(ecg, label='ECG crudo', alpha=0.4)
plt.plot(ecg_filt, label='ECG filtrado', linewidth=1)
plt.title('ECG no flitrado vs Filtrado')
plt.xlabel('Muestras')
plt.ylabel('Amplitud')
plt.legend()
plt.grid(True)
```
Comparación visual entre señal original y señal filtrada.


![image](https://github.com/user-attachments/assets/f0fa5d41-d817-4e4d-a7f4-e172f374f0fa)



##### Picos R detectados
```
plt.figure(figsize=(12, 4))
plt.plot(ecg_filt, label='ECG filtrado')
plt.plot(peaks, ecg_filt[peaks], 'ro', label='Picos R')
plt.title('Picos R detectados')
plt.xlabel('Muestras')
plt.ylabel('mV')
plt.legend()
plt.grid(True)
```
Muestra los picos R detectados sobre la señal filtrada. los cuales se pueden ver por puntos rojos.


![image](https://github.com/user-attachments/assets/e0165272-49e8-464a-b729-521f1bc872c5)


#####  Espectrograma (Transformada Wavelet)
```
plt.figure(figsize=(10, 5))
plt.imshow(np.abs(cwtmatr), aspect='auto', cmap='jet',
           extent=[0, len(rr_intervals), widths[-1], widths[0]])
plt.colorbar(label='Magnitud')
plt.title('Espectrograma (Wavelet Continua - Morlet)')
plt.xlabel('RR intervalo index')
plt.ylabel('Escala (relacionada con frecuencia)')
plt.tight_layout()
plt.show()
```
Visualiza un espectrograma usando la transformada wavelet.
El eje Y (escalas) está invertido para que escalas más grandes estén abajo (como frecuencias bajas).


![image](https://github.com/user-attachments/assets/2f9f5766-0303-4205-8327-a447f645e7dc)



## conclusiones
El análisis en el dominio del tiempo proporciona una visión general de la frecuencia cardíaca mediante parámetros como la media y la desviación estándar de los intervalos R-R. Es útil para obtener una idea global del comportamiento del corazón, pero no permite observar cómo cambian a lo largo del tiempo (frecuencia). Por el contrario, el análisis en el dominio tiempo-frecuencia, como el realizado con la transformada wavelet, permite visualizar cómo varían las frecuencias de los intervalos R-R a travez del tiempo. Esto lo hace más útil para detectar cambios en la actividad cardíaca que el análisis temporal no puede identificar, en otras palabras el funcionamiento y/o activacion del sistema simpatico como parasimpatico.

El tipo de wavelet utilizado influye en la resolución y precisión del análisis. La Morlet(que es una funcion wavelet), son ideales para señales fisiológicas porque ofrecen un buen equilibrio entre resolución temporal y frecuencia.

Este tipo de análisis se aplica en medicina para detectar arritmias y evaluar la variabilidad de la frecuencia cardíaca (HRV), la cual es un indicador del estado del sistema nervioso autónomo. También es útil en dispositivos portátiles como relojes inteligentes y bandas deportivas, en investigaciones sobre estrés, sueño y emociones.

Gracias a el analisis frecuencia tiempo, se pudo identificar con la frecuencia cambiaba a travez del tiempo, viendo su intencidad, gracias a este analisis pudimos ver como dependiendo de que actividad se haga nuestro cuerpo puede responder al activar o desactivar los sistemas simpaticos y parasimpaticos, esto influenciando de forma directa la frecuencia cardiaca y la HRV. En nuestro paciente se pudo evidenciar que el estudio empezo con la activacion del sistema simpatico para luego de relarse pasar a activar o usar el sistema parasimpatico, la importancia de reconocer estos cambios en el HRV, es describir que no siempre vamos a estar 100% activos llenos de adrenalina y con alta frecuencia cardiaca ya que reconociendo en que actividad estamos que sistema poderse activar.

Realizado por: Sara Vasquez y Eduard Alarcon
