# Diagramas Mermaid → Excalidraw → PNG

Cada `.mmd` de esta carpeta es un diagrama listo para pegar en Excalidraw:
**☰ menú → "Mermaid to Excalidraw" → pega el contenido → Insert → estiliza → exporta PNG a `docs/img/`.**

| Archivo .mmd | Exportar como | Tipo | Fase | Nota |
|--------------|---------------|------|------|------|
| `Img_1.mmd` | `docs/img/Img_1.png` | sequence | 3, 4 | reemplaza el flujo síncrono actual |
| `Img_2.mmd` | `docs/img/Img_2.png` | flowchart | 5, 7 | reemplaza el flujo por eventos actual |
| `Img_4.mmd` | `docs/img/Img_4.png` | class | 2, 3 | nuevo (modelo de la feature account) |
| `Img_5.mmd` | `docs/img/Img_5.png` | sequence | 8 | nuevo (patrón outbox) |
| `Img_6.mmd` | `docs/img/Img_6.png` | sequence | 9 | nuevo (coroutines en paralelo) |

`Img_0.png` (arquitectura) e `Img_3.png` (antes/después) se quedan como están.

Todas las etiquetas van en una sola línea a propósito, para evitar el glitch del `\n` en Excalidraw.
Los seis diagramas fueron validados con el parser oficial de Mermaid (0 errores).
