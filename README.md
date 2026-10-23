# Análisis multiespectral de imágenes Sentinel-2 para la detección automática de áreas irrigadas por pivotes centrales

Trabajo Final de la **Licenciatura en Ciencias de la Computación** (FaMAF, Universidad Nacional de Córdoba), presentado en febrero de 2026.

- **Autora:** María Victoria López Amuchástegui
- **Directores:** Mg. Axel Waldemar Elesser y Dr. Juan Bautista Cabral

📄 Informe completo: [`TFC_Vicky_Lopez.pdf`](TFC_Vicky_Lopez.pdf)

## De qué se trata

Los **pivotes centrales** son sistemas de riego que giran alrededor de un punto fijo y dejan en el campo círculos fáciles de reconocer desde el satélite. Detectarlos automáticamente sirve para saber cuánta superficie se riega y dónde.

La tesis compara **cuatro combinaciones de bandas de Sentinel-2** para ver cuál permite detectar y delimitar mejor los pivotes con un modelo de segmentación por deep learning. El área de estudio es la provincia de **Córdoba**, en la campaña primavera–verano **2023–2024**.

| Composición | Bandas | Qué resalta |
|-------------|--------|-------------|
| Color natural | B4, B3, B2 | La imagen como la vería el ojo humano |
| Bordes rojos | B7, B6, B5 | El estado y la actividad de la vegetación |
| Falso color | B8A, B4, B3 | El vigor de la vegetación (infrarrojo cercano) |
| SWIR | B11, B8, B4 | La humedad del suelo y de la vegetación |

## Cómo se hizo

```
Google Earth Engine ──► Grounding DINO ──► SAM ──► QGIS ──► Anotaciones YOLO ──► YOLO11s-seg ──► Curvas Precisión–Recall
  (imágenes y            (encuentra       (máscara   (validación     (polígonos        (un modelo por
   composiciones)         los círculos)    de cada    geoespacial)    por parche)       composición)
                                           pivote)
```

1. **Imágenes.** Con **Google Earth Engine** armé la mediana temporal de la campaña para cada composición y la exporté en parches **GeoTIFF** georreferenciados.
2. **Etiquetado semiautomático.** **Grounding DINO** encuentra los pivotes a partir de una descripción en texto, y **SAM** (*Segment Anything*) genera la máscara de cada uno. Así obtuve las etiquetas sin dibujarlas a mano.
3. **Validación en QGIS.**
   - Revisé que cada máscara coincidiera con su imagen y descarté las detecciones erróneas.
   - Vectoricé las máscaras con GDAL Polygonize y unifiqué todo en el sistema de referencia POSGAR 98, faja 4.
4. **Dataset.**
   - Con scripts en Python convertí los polígonos al formato de anotación de YOLO.
   - En total hay **169 parches**: **19** quedaron fijos como conjunto de prueba y **150** se repartieron en **15 folds** para validación cruzada.
5. **Entrenamiento.** Entrené **YOLO11s-seg** por separado sobre cada composición, con la misma arquitectura, los mismos hiperparámetros y los mismos folds.
6. **Evaluación.**
   - Curvas **Precisión–Recall** con cuatro umbrales de **IoU**: 0.75, 0.80, 0.85 y 0.90. El IoU mide cuánto se superpone la máscara predicha con el pivote real.
   - Además, comparé la precisión de cada composición con el **recall fijo en 0.80**, es decir, encontrando el 80 % de los pivotes reales.

## Resultados

Precisión con recall fijo en 0.80:

| Composición | IoU 0.75 | IoU 0.80 | IoU 0.85 | IoU 0.90 |
|-------------|:--------:|:--------:|:--------:|:--------:|
| Color natural | 0.962 | 0.959 | 0.939 | 0.168 |
| Bordes rojos | 0.975 | 0.974 | 0.964 | 0.214 |
| Falso color | 0.974 | 0.974 | **0.967** | 0.155 |
| **SWIR** | **0.979** | **0.977** | 0.961 | **0.273** |

- **Para detectar, todas sirven.** Con IoU de hasta 0.85, las cuatro composiciones superan el **95 % de precisión** al encontrar el 80 % de los pivotes.
- **Para delimitar con exactitud, conviene SWIR.** Con IoU 0.90 la precisión cae en todos los casos, porque la resolución de las imágenes no alcanza para un ajuste tan fino. Igual aparecen diferencias: **SWIR** es la que mejor se sostiene, seguida por los **bordes rojos**.
- **Conclusión:** la elección de las bandas casi no cambia la capacidad de **detectar** los pivotes, pero sí la precisión con la que se **delimitan** sus contornos.

## Trabajo a futuro

- Probar otras combinaciones de bandas e índices como NDVI, NDWI o NDMI.
- Usar fechas puntuales de la campaña en vez de la mediana, y otras estaciones del año.
- Aplicar el modelo en otras regiones con pivotes (sur de Buenos Aires, San Luis, La Pampa) para ver si generaliza sin reentrenar.
- Clasificar qué cultivo hay en cada pivote.

## Herramientas

Python · Google Earth Engine · Google Colab · Grounding DINO · Segment Anything (SAM) · QGIS · GDAL · Ultralytics YOLO11

## Licencia

El informe se publica bajo licencia [Creative Commons Atribución-CompartirIgual 4.0 Internacional](https://creativecommons.org/licenses/by-sa/4.0/deed.es) (CC BY-SA 4.0).
