# Mini Juego Isométrico de Construcción y Colocación de Objetos
**Prueba Técnica — Programador Unreal Engine 5 (Mid-Level)**

---

## Versión de Unreal Engine

Unreal Engine **5.7.4**

---

## Cómo ejecutar el proyecto

1. Clonar o descomprimir el proyecto en una ubicación local.
2. Abrir el archivo `.uproject` con Unreal Engine 5.7.4.
3. Si el editor solicita recompilar módulos, aceptar.
4. Abrir el nivel `Levels/Lvl_IsometricMinigame`.
5. Presionar **Play** en el editor o empaquetar el proyecto para ejecutarlo de forma independiente.

> **Nota:** No se requieren plugins adicionales. El proyecto usa únicamente sistemas nativos de Unreal Engine 5.

---

## Controles

| Acción | Control |
|---|---|
| Seleccionar building | Clic en botón del catálogo (panel inferior) |
| Arrastrar building | Mantener clic izquierdo y mover el mouse |
| Soltar y confirmar posición | Soltar clic izquierdo |
| Confirmar colocación | Botón **Confirm** (verde) |
| Cancelar colocación | Botón **Cancel** (rojo) |
| Seleccionar building colocado | Clic izquierdo sobre el objeto |
| Eliminar building | Botón **Delete** en el widget del objeto |

---

## Descripción del proyecto

Prototipo de juego isométrico de construcción desarrollado en Unreal Engine 5 con Blueprints. El jugador puede seleccionar construcciones desde un catálogo, arrastrarlas al mundo, confirmar su colocación y eliminarlas. El sistema detecta colisiones en tiempo real con feedback visual y persiste el estado del mundo mediante el sistema nativo de SaveGame de Unreal Engine.

---

## Decisiones técnicas relevantes

### 1. Colisiones escalables mediante array de PrimitiveComponent

La clase padre `BP_BuildingBase` define un array de tipo `PrimitiveComponent` para gestionar todos los componentes de colisión del building, independientemente de su forma (Box, Sphere, Capsule). Cada componente se identifica mediante un **Component Tag**, y el array se puebla automáticamente en el `Construction Script` mediante `GetComponentsByClass`.

Esta decisión permite que buildings con geometría compleja, como el arco (`BP_BuildingTypeC`), definan múltiples collision shapes que respetan el espacio vacío de su mesh, sin modificar ninguna lógica en la clase padre. Agregar un nuevo building con cualquier configuración de colisión no requiere cambios en el sistema de detección.

---

### 2. Referencias internas cacheadas en BeginPlay

Todos los Casts a sistemas externos (PlayerController, GameState, Widgets) se resuelven una única vez mediante el evento `GetInternalReferences`, llamado en el `BeginPlay` del building. Los resultados se almacenan en variables locales que se reutilizan durante todo el ciclo de vida del actor.

Esto evita Casts repetidos en eventos que se ejecutan frecuentemente como `StartDragMode` y `StopDragMode`, cumpliendo el criterio de llamadas eficientes sin sacrificar la accesibilidad a sistemas externos.

---

### 3. Detección de obstáculos mediante eventos de overlap, no Tick

La validez de la posición del ghost se determina mediante `OnComponentBeginOverlap` y `OnComponentEndOverlap`, bindeados dinámicamente al iniciar el drag y liberados al detenerlo. El sistema no realiza ninguna verificación en el `Event Tick`.

Cuando el ghost entra en contacto con un obstáculo, `DetectObstacle` actualiza el estado interno y modifica el material dinámico a rojo. Cuando lo abandona, `LeaveObstacle` restaura el material a su estado neutro. El sistema considera cualquier actor físico del mundo como obstáculo potencial, no solo buildings colocados, lo que permite mayor flexibilidad en el diseño de niveles.

---

### 4. Flag BuildingIsBeingLoaded para evitar guardados redundantes

`ConfirmPlacement` es la función que finaliza la colocación de un building, tanto en el flujo normal del jugador como durante la carga del SaveGame. Para evitar que `SaveAllBuildings` se llame repetidamente por cada building que se carga al inicio, el sistema utiliza una variable booleana `BuildingIsBeingLoaded`.

Al cargar, `LoadAllBuildings` activa este flag en el actor antes de llamar `ConfirmPlacement`. La función verifica el flag internamente: si está activo, omite el guardado y limpia el flag. Si está inactivo, ejecuta el guardado normalmente. Esto mantiene una única función de confirmación sin duplicar lógica ni agregar parámetros externos.

---

### 5. Comunicación desacoplada mediante Event Dispatchers

La comunicación entre los buildings y los widgets de UI (`WBP_PlacementConfirmation`, `WBP_DeleteBuilding`) se realiza exclusivamente mediante **Event Dispatchers**. Los buildings exponen dispatchers como `OnPlacementConfirmed`, `OnPlacementCanceled`, `OnBuildingDeleted` y `OnPlacementValidityChanged`. Los widgets y sistemas externos se suscriben a estos eventos en el momento en que son relevantes y se desuscriben cuando dejan de serlo.

Esto garantiza que el building no tenga dependencias directas hacia los widgets ni hacia el PlayerController en su lógica central, respetando la separación de responsabilidades.

---

### 6. Spawn dinámico mediante TSubclassOf en Blueprints

El catálogo de selección pasa directamente la **clase Blueprint** del building al evento `BeginPlacement` mediante un pin de tipo `Class Reference (BP_BuildingBase)`. El sistema hace Spawn de cualquier clase hija de `BP_BuildingBase` sin necesidad de un Switch ni de modificar la lógica de placement al agregar nuevos buildings.

Cada botón del catálogo almacena su clase asociada como variable, lo que permite agregar nuevos tipos de building simplemente creando un Blueprint hijo y asignándolo al botón correspondiente en el widget.

---

### 7. SaveGame reconstruido desde actores vivos en escena

`SaveAllBuildings` no mantiene una lista sincronizada manualmente. En cada llamada, itera todos los actores con el tag `PlacedBuilding` presentes en el mundo y reconstruye el array desde cero. Esto garantiza que el estado guardado siempre refleja exactamente lo que hay en escena, sin posibilidad de inconsistencias entre el mundo y el archivo de guardado.

Al eliminar un building, el actor remueve su tag antes de destruirse como medida defensiva, asegurando que no sea incluido en el guardado independientemente del orden de ejecución de los suscriptores al dispatcher de eliminación.

---

## Estructura del proyecto

```
Content/
├── Blueprints/
│   ├── Buildings/
│   │   ├── BP_BuildingBase         ← Clase padre con lógica compartida
│   │   ├── BP_BuildingTypeA        ← Building tipo A
│   │   ├── BP_BuildingTypeB        ← Building tipo B
│   │   ├── BP_BuildingTypeC        ← Building tipo C (arco)
│   │   └── BPI_Placeable           ← Interface de colocación
│   └── Core/
│       ├── GameData/
│       │   ├── GI_SaveManager      ← Game Instance con lógica de guardado
│       │   ├── SG_PlacedBuildings  ← SaveGame con array de edificios
│       │   └── ST_BuildingData     ← Structure con datos de cada edificio
│       ├── BPP_IsometricPawn       ← Pawn de la cámara isométrica
│       ├── EPlacementState         ← Enum de estados del sistema
│       ├── GM_IsometricMinigame    ← GameMode
│       ├── GS_IsometricMinigame    ← GameState con lógica de placement
│       ├── HUD_IsometricMinigame   ← HUD
│       └── PC_IsometricPlayerController ← PlayerController con input
├── Input/
│   ├── Actions/
│   │   └── IA_Select_Click         ← Input Action del clic
│   └── IMC_IsometricMinigame       ← Input Mapping Context
├── Levels/
│   └── Lvl_IsometricMinigame       ← Nivel principal
├── Materials/
│   ├── MI_BaseBuildingMaterial     ← Material Instance base
│   ├── MM_BaseBuildingMaterial     ← Material base
│   └── MM_GhostBuildingMaterial    ← Material del ghost
├── Textures/
│   ├── Icons/
│   │   └── Delete_Icon             ← Ícono del botón eliminar
│   └── Thumbnails/
│       ├── Thumbnail_BuildingTypeA
│       ├── Thumbnail_BuildingTypeB
│       └── Thumbnail_BuildingTypeC
└── Widgets/
    ├── WBP_BuildingSelection       ← Catálogo de buildings
    ├── WBP_DeleteBuilding          ← Widget de eliminación
    └── WBP_PlacementConfirmation   ← Widget de confirmación
```
