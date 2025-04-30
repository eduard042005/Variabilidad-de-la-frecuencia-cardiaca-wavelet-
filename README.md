# INFORME 5
## Variabilidad de la Frecuencia Cardiaca usando la Transformada Wavelet 
![image](https://github.com/user-attachments/assets/a400a889-006e-4346-91fb-de1d16a94f33)

En este informe de laboratorio se plantea como objetivo analizar la variabilidad de la frecuencia cardíaca (HRV) utilizando la transformada wavelet para identificar cambios en las frecuencias características y analizar la dinámica de esta.

El laboratorio se realizó tomando a un paciente en el cual se le realizara un electrocardiograma (ECG), la señal cardiaca seria procesada por un sensor ECG y esta señal será adquirida a través de una blu phil (microcontrolador), sirviendo como un sistema de adquisición de datos.
En esta toma de datos el paciente se verá sometido a tres tipos de actividades distintas, la primera es estar una actividad de estrés o activiad fisica esto con el fin de activar el sistema simpático, aumentando la frecuencia. A continuación, se dispondría a el paciente en un estado de “normalidad” escuchando el ambiente y hablando con personas. Y por último en reposo, con la intención que el sistema parasimpático del cuerpo humano, bajara la frecuencia cardiaca. Esta medicion se dispondra de 5 minutos para cada tipo de actividad.
Todo esto con el fin de poder saber cómo estímulos externos activan los sistemas (simpático y parasimpático) y gracias a esta activación como afecta directamente a la variabilidad de la frecuencia cardiaca (HRV).
La HRV (Heart Rate Variability) o  en español la variabilidad de la frecuencia cardiaca, es la medida de las fluctuaciones en el tiempo entre latidos sucesivos, específicamente entre los picos R del complejo QRS en un electrocardiograma (intervalo R–R). Cuanto más variable es ese intervalo, más flexible y adaptativo es el sistema nervioso autónomo.

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
