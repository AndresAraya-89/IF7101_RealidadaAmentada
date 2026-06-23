# Documento Técnico — Proyecto Escuela RA

**Curso:** IF-7102 Optativa: Multimedios — *Realidad Aumentada Interactiva*
**Facilitador:** Mag. Wilber Rodríguez Recinos — UCR, Recinto de Limón
**Archivo base:** `escuela_RA.blend`
**Motor de modelado:** Blender 5.1 · **RA:** Aumentaty Autor + Aumentaty Viewer

> Este documento explica el **valor técnico** del proyecto: arquitectura del modelo
> 3D, decisiones de ingeniería, pipeline hacia Realidad Aumentada, gestión de
> recursos descargados, optimización y cumplimiento de licencias.

---

## 1. Resumen técnico

Se desarrolló una escuela 3D de **topología en anillo (patio central)** modelada de
forma **paramétrica y reproducible** en Blender 5.1, complementada con modelos
**glTF** importados desde Sketchfab, y preparada para exhibirse en **Realidad
Aumentada basada en marca** mediante Aumentaty. El proyecto combina **geometría
generada por código** (estructura sólida del edificio) con **assets externos**
(vegetación, mobiliario, vehículo, interiores), integrados en una sola escena.

---

## 2. Arquitectura del modelo 3D

### 2.1 Diseño paramétrico
Toda la edificación se deriva de un conjunto de constantes (alto y grosor de muro,
ancho de pasillo, tamaño de plaza, profundidad de salas). Los límites del anillo se
**calculan dinámicamente** para evitar solapes y huecos:

- `PLAZA 20×30 m`, `CORRIDOR_WIDTH 2.5 m`, `ROOM_DEPTH 7 m`, `WALL_HEIGHT 3.0 m`.
- Borde interior del anillo `COUT = PLAZA/2 + CORRIDOR_WIDTH`; borde exterior
  `ROUT = COUT + ROOM_DEPTH`.

Esto hace el diseño **escalable**: cambiar una constante reconfigura toda la escuela.

### 2.2 Construcción sólida (no piezas sueltas)
El edificio se construye como un **cascarón continuo**:
- Perímetro **exterior** e **interior** continuos + **particiones compartidas**
  (sin muros dobles) + **piso y techo en anillo**.
- Las piezas se **unen** (`join`) y se **sueldan** (merge by distance) en mallas
  únicas: `Edificio_Muros_Solido`, `Edificio_Techo_Solido`, `Edificio_Piso_Solido`.

**Valor técnico:** simula una construcción real (estanca, sin grietas entre módulos)
y reduce el número de objetos, lo que mejora el rendimiento en RA.

### 2.3 Aberturas reales (operaciones booleanas)
Puertas (1.0×2.1 m) y ventanas (1.5×1.2 m con alféizar 1.0 m) se generan con
**modificadores Boolean (DIFFERENCE, solver EXACT)** que recortan huecos auténticos
en los muros; el acceso atraviesa muro exterior e interior. No son texturas
pintadas: son **vanos navegables**.

### 2.4 Organización por colecciones
La escena se ordena en colecciones (`00_Terreno`, `01_PlazaCentral`,
`02_PasilloAnillo`, `03_EdificioSolido`, `04_Ventanas`, `05_Mobiliario`,
`06_Exterior`), lo que facilita selección, edición y exportación selectiva.

### 2.5 Reproducibilidad por código
La estructura se genera con un script Python (`bpy`) por **fases** (layout →
dimensiones → aberturas → mobiliario). Esto aporta **trazabilidad y reproducibilidad**:
la escuela puede regenerarse de cero de forma determinista.

---

## 3. Integración de assets externos (glTF de Sketchfab)

Los modelos descargados vienen en formato **glTF 2.0** (`scene.gltf` + `scene.bin`
+ `textures/`), el estándar recomendado para intercambio 3D porque transporta
geometría, materiales y texturas de forma compacta.

| Modelo | Geometría (`scene.bin`) | Texturas | Observación técnica |
|--------|------------------------:|---------:|---------------------|
| Árbol descarga | ~33.1 MB | 3 | **Alto detalle** — pesado para RA |
| loft(2) FREE interior | ~2.7 MB | 23 | Muchas texturas (interiores) |
| Piedad (escultura) | ~1.6 MB | 1 | Malla escaneada, densa |
| Low Poly Forest Tree Pack | ~0.5 MB | 17 | **Low-poly** — óptimo para RA |
| Classic Muscle car | ~0.24 MB | 0 | Ligero, color por material |
| Tables And Chairs | ~0.16 MB | 6 | Ligero |

**Decisión técnica clave (rendimiento en RA):** conviene priorizar los modelos
**low-poly** (p. ej. *Low Poly Forest Tree Pack*) para el grueso de la vegetación y
**limitar** el uso de modelos muy pesados como *Árbol descarga* (~33 MB) o la
escultura escaneada, que pueden ralentizar o impedir la carga en Aumentaty Viewer
sobre dispositivos móviles. El tamaño total del `.blend` (~119 MB) refleja la
incorporación de estos assets de alta resolución.

---

## 4. Pipeline hacia Realidad Aumentada

```
Blender 5.1  ──►  Exportar COLLADA (.dae) + texturas  ──►  Aumentaty Autor
   (modelo)            (File > Export > Collada)           (vincular a marca)
                                                                   │
                                                                   ▼
                          Aumentaty Viewer  ◄── PDF de marca impresa
                       (cámara apunta a la marca → aparece la escuela 3D)
```

Pasos técnicos:
1. **Empacar recursos** en Blender: `File > External Data > Pack Resources` (evita
   texturas perdidas al mover el proyecto).
2. **Exportar a COLLADA `.dae`** (formato aceptado por Aumentaty) con texturas.
3. En **Aumentaty Autor**: subir el modelo y **vincularlo a la marca**.
4. Generar el archivo para **Aumentaty Viewer**.
5. Imprimir la **marca** (PDF) y apuntar la cámara para visualizar la escena.

**Recomendaciones de marca:** patrón cuadrado, asimétrico, de alto contraste y
borde grueso, para detección estable de posición y orientación.

---

## 5. Optimización para RA (móvil)

- **Conteo de polígonos:** preferir low-poly; *decimar* (Decimate modifier) los
  assets pesados antes de exportar.
- **Texturas:** reducir resolución y cantidad; el interior loft aporta 23 texturas
  que pueden recortarse si no son visibles.
- **Reutilización (instancing):** repetir un mismo árbol como instancias en lugar de
  copias únicas reduce memoria.
- **Escala y origen:** mantener unidades en **metros** y centrar el predio en
  `(0,0,0)` para que la marca quede en el centro del modelo.
- **Reducción de objetos:** la unión/soldado del edificio ya disminuye draw calls.

---

## 6. Validación

Las pruebas de visualización (`imagenes_prueba/`) documentan cuatro ángulos:
- **Vista de frente**, **Vista de pájaro (cenital)**, **Vista trasera** y
  **Vista del plano arquitectónico (wireframe)**.

Sirven como evidencia de: correcta distribución en anillo, integración de
vegetación y complementos, escala coherente (auto vs. edificio) y solidez de la
estructura (la vista wireframe permite inspeccionar la malla).

---

## 7. Cumplimiento de licencias (aspecto técnico-legal)

Todos los modelos externos se usan conforme a su licencia. **Cinco** son
**CC-BY-4.0** (exigen **crédito al autor**) y **uno** es **Sketchfab Standard**:

| Modelo | Autor | Licencia |
|--------|-------|----------|
| Árbol descarga | tamaliteitor123 | CC-BY-4.0 |
| Low Poly Forest Tree Pack | 99.Miles | CC-BY-4.0 |
| Tables And Chairs | OfekDavid | CC-BY-4.0 |
| Classic Muscle car | Lexyc16 | CC-BY-4.0 |
| Piedad – La Virgen con el cuerpo de Cristo | the dragon 3d | CC-BY-4.0 |
| loft(2) FREE interior | dasy444 | Sketchfab Standard |

> Los textos de crédito originales se conservan en cada `license.txt` dentro de
> `objetos_descargadas/` y **deben** incluirse en la entrega final (documentación
> y/o créditos) para cumplir CC-BY-4.0.

---

## 8. Limitaciones y trabajo futuro

- **Peso de assets:** los modelos de alta resolución elevan el tamaño del proyecto;
  optimizar antes del despliegue en RA móvil.
- **Interactividad:** la RA basada en marca muestra el modelo; una iteración futura
  podría añadir **animación o puntos de información** (etiquetas de especies de
  árboles) para reforzar el enfoque biológico.
- **Multimarca:** dividir la escena en varias marcas (edificio, bosque, parqueo)
  para aligerar cada vista.

---

## 9. Cumplimiento de los requisitos del PDF

- ✔ Herramienta de RA interactiva e innovadora (educación / Biología).
- ✔ ≥10 objetos 3D (edificación + vegetación + complementos).
- ✔ ≥3 objetos principales creados por el equipo (aulas, dirección, comedor).
- ✔ Entregables previstos: proyecto en Blender, exportación para Aumentaty,
  archivo para Viewer, PDF de marca y archivos de soporte.
