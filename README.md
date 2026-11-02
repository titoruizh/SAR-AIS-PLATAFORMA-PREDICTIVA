# SAR-AIS-PLATAFORMA-PREDICTIVA

**Tesis de Magíster: predecir dónde estará un barco para orientar imágenes satelitales SAR**

> 🔒 **Código privado.** Este repositorio es un *showcase* de mi trabajo de grado (Magíster en Ingeniería Informática, Universidad Andrés Bello). El código y los datos de trabajo no se publican. Las cifras corresponden a una simulación académica.

**Estado:** tesis de Magíster defendida (2026). Trabajo de grado realizado en dupla.

## English summary
Master's thesis project: a platform that uses a ship's AIS history to estimate where it will be in the next hours and points the satellite radar (SAR) search there, instead of using only its last known position. In a simulation of 200 cases, the learned model located 163 ships versus 131 with a simple straight-line projection, using fewer images (−39.61% total estimated cost, −51.47% cost per ship located). Results come from a historical, simulated setup, not from a real operation. The SAR ship detector (YOLO11m) was also exported to ONNX and TensorRT; TensorRT FP16 runs about 2.8× faster than PyTorch at batch 1 and 3.2× at batch 8, with a mAP50-95 drop of 0.0005 (see "Deployment: ONNX / TensorRT").

## Problema
Cuando un barco deja de transmitir su posición AIS, sigue moviéndose y su última posición conocida queda desactualizada. Las imágenes de radar satelital (SAR) permiten ver el mar de día, de noche y con nubes, pero cada imagen cuesta dinero y cubre un área acotada. Si la imagen se pide sobre la última posición, muchas veces solo muestra mar vacío.

**La pregunta:** cuando un barco apaga su AIS, ¿dónde conviene pedir la imagen SAR para volver a encontrarlo gastando menos?

## Enfoque
1. **Datos:** registros AIS históricos públicos (NOAA, 2024), revisados y limpiados.
2. **Modelo:** una red neuronal lee 12 horas de historia del barco y estima su posición hasta 36 horas después.
3. **Comparación:** se compara contra una regla simple que proyecta la última posición en línea recta con el mismo rumbo y velocidad.
4. **Evaluación:** 200 casos simulados, con las mismas reglas para ambos métodos, contando cuántos barcos se encuentran y cuántas imágenes se necesitan.
5. **Plataforma:** mapa web donde se ve el historial, la predicción y la zona de búsqueda propuesta, más un apoyo para revisar una imagen SAR ya tomada.

## Arquitectura
```mermaid
flowchart LR
  A[Historial AIS] --> B[Limpieza de datos]
  B --> C[Modelo de predicción<br/>red neuronal]
  C --> D[Zona de búsqueda]
  D --> E[Mapa web para el operador]
  E --> F[Planificación de imagen SAR]
  F --> G[Revisión de la imagen<br/>detección de barcos]
```

## Stack
Python · PyTorch · FastAPI · Astro · React · Leaflet · YOLO11m (revisión de imágenes) · ONNX · TensorRT · Docker con GPU

## Resultados (simulación de 200 casos)
| Indicador | Modelo propuesto | Proyección en línea recta |
|---|---|---|
| Barcos localizados | 163 / 200 | 131 / 200 |
| Imágenes usadas | 2.877 | 4.764 |
| Reducción de costo total estimado | 39,61 % | — |
| Reducción de costo por barco localizado | 51,47 % | — |

<img width="697" height="432" alt="image" src="https://github.com/user-attachments/assets/ee3ac107-9800-4d07-acbf-76826b07b51d" />


Notas: los costos son una estimación técnica y las pasadas del satélite fueron simuladas; no es un ahorro real ni una prueba en operación. "Localizado" significa que la posición real quedó dentro de alguna imagen planificada. La meta inicial era reducir al menos 50 % el costo; se observó una reducción de 51,47 % por barco localizado, pero la muestra no permite afirmarlo con certeza estadística.

## Deployment: ONNX / TensorRT

El detector de barcos en imágenes SAR (YOLO11m) usado para revisar las imágenes se exportó a ONNX y TensorRT para medir cuánto se gana en velocidad y cuánta precisión se pierde.

### Precisión
`model.val()` sobre el split de validación: 4.649 chips, imgsz 640, conf 0,001, IoU 0,7.

| Formato | mAP50 | mAP50-95 | Δ mAP50-95 vs PyTorch |
|---|---|---|---|
| PyTorch FP32 | 0,9681 | 0,7076 | — |
| ONNX FP32 | 0,9681 | 0,7075 | −0,0001 |
| TensorRT FP16 | 0,9675 | 0,7071 | −0,0005 |
| TensorRT INT8 (mixto) | 0,9494 | 0,6520 | −0,0556 |

### Latencia y throughput
"Forward" es solo el modelo; "e2e" es `predict()` completo con preproceso y NMS. En batch 8 la latencia es por imagen (la del lote dividida por 8). Las medias de forward son de 500 medidas tras 50 de calentamiento.

| Formato | Batch | Forward ms/img (media / p95) | Forward img/s | e2e ms/img | e2e img/s | VRAM |
|---|---|---|---|---|---|---|
| PyTorch FP32 | 1 | 7,07 / 9,22 | 142 | 9,04 | 111 | 264 MiB |
| PyTorch FP16 | 1 | 6,88 / 8,76 | 145 | 9,20 | 109 | 170 MiB |
| ONNX FP32 | 1 | 7,70 / 9,63 | 130 | 9,95 | 100 | 560 MiB |
| TensorRT FP16 | 1 | 2,54 / 4,75 | 393 | 4,45 | 225 | 98 MiB |
| TensorRT INT8 (mixto) | 1 | 2,32 / 4,06 | 432 | 4,19 | 239 | 72 MiB |
| PyTorch FP32 | 8 | 2,06 / 2,20 | 485 | 3,97 | 252 | 1.044 MiB |
| PyTorch FP16 | 8 | 1,25 / 1,32 | 800 | 3,36 | 297 | 664 MiB |
| ONNX FP32 | 8 | 2,04 / 2,13 | 491 | 3,79 | 264 | 2.294 MiB |
| TensorRT FP16 | 8 | 0,65 / 0,90 | 1.533 | 2,75 | 364 | 406 MiB |
| TensorRT INT8 (mixto) | 8 | 0,53 / 0,77 | 1.905 | 2,57 | 389 | 320 MiB |

En resumen, TensorRT FP16 es unas 2,8 veces más rápido que PyTorch FP32 en forward a batch 1 y unas 3,2 veces a batch 8, con una baja de 0,0005 en mAP50-95.

### Notas de método
- **INT8:** se construyó con calibración entrópica propia sobre el split de entrenamiento, manteniendo en FP16 la cabeza, los bloques C2PSA y el stem. El INT8 por defecto de Ultralytics bajaba el mAP50-95 a 0,387 y no se usa. Incluso el INT8 mixto pierde 0,0556 de mAP50-95, así que se dejó como alternativa y no como opción principal.
- **Engine recomendado:** TensorRT FP16 con batch 8, por su equilibrio entre velocidad y precisión.
- **Inferencia sobre GeoTIFF:** `infer_trt.py` procesa una escena SAR con el engine TensorRT y entrega las detecciones en GeoJSON.
- **Entorno reproducible:** imagen Docker con TensorRT (base `nvidia/cuda:12.8.0-cudnn-devel-ubuntu24.04`).
- **Variabilidad:** las cifras de PyTorch a batch 1 cambian entre corridas (en una corrida anterior FP32 dio 7,8 ms y FP16 dio 11,7 ms), mientras que TensorRT FP16 se mantuvo dentro de ±8 %. Las tablas son de la corrida final y deben leerse como referencia, no como valores fijos.

### Hardware y software
NVIDIA GeForce RTX 5090 (34,2 GB, compute capability 12.0) · driver 595.79 · Windows 11 con Docker Desktop/WSL2 · CUDA 12.8 · cuDNN 9.20 · PyTorch 2.12 (dev, cu128) · Ultralytics 8.4.32 · TensorRT 10.13.3.9 · ONNX 1.19.1 · ONNX Runtime 1.23.2 (CUDA EP).

> Los pesos, los archivos ONNX y los engines TensorRT no se publican; el código sigue siendo privado.

## Capturas
<!-- TODO: captura del mapa con historial y predicción (sin datos sensibles) -->
<img width="1204" height="569" alt="image" src="https://github.com/user-attachments/assets/769629fc-7b88-4670-b3c6-b385447bb166" />

<!-- TODO: GIF o figura comparando predicción vs línea recta -->
<img width="585" height="329" alt="image" src="https://github.com/user-attachments/assets/a3f951d5-e302-4b4c-bde7-9d17e95f6f82" />

<img width="406" height="224" alt="image" src="https://github.com/user-attachments/assets/dfbb60a4-e0ad-4b20-b559-e1d127683d98" />

## Autor
Tito Ruiz Haros · [github.com/titoruizh](https://github.com/titoruizh)
