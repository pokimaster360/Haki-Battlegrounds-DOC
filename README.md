# Roblox Battlegrounds — Documentación de Arquitectura

## 1. Descripción del proyecto

Juego de combate estilo Battlegrounds inspirado en One Piece, desarrollado en Roblox utilizando Luau.

	El objetivo es crear un sistema de combate modular que permita desarrollar habilidades, estilos de combate, Haki, progresión y entrenamiento sin que cada nuevo sistema requiera modificar toda la arquitectura.

	### Principios fundamentales

- El servidor es la autoridad sobre el combate.
	- El cliente comunica intenciones, no resultados.
	- Las habilidades son modulares y configurables.
	- Los sistemas de combate y progresión son independientes.
	- Los datos del jugador son administrados centralizadamente.
	- La progresión debe beneficiar al jugador sin generar diferencias de poder absurdas.
	- Los sistemas deben permitir añadir nuevos estilos, habilidades y actividades sin rehacer los existentes.

	---

	# 2. Arquitectura general

La arquitectura está dividida en cuatro grandes áreas:

	1. **Client:** input, animaciones, VFX y presentación.
	2. **Combat:** ejecución de habilidades, hitboxes, estados, daño y movimiento.
	3. **Player:** datos del jugador y estados.
	4. **Progression / Training:** recompensas, experiencia y actividades de entrenamiento.

	El flujo general:

	```text
```
CLIENT
│
├── InputController
│       │
│       ▼
├── CombatManager (Client)
│       │
│       ▼
├── Remotes
│
└─────────────────────────────────────────
│
▼
SERVER
│
├── CombatManager (Server)
│       │
│       ▼
├── SkillService
│       │
│       ▼
├── Skill
│       │
│       ├── HitboxService
│       ├── DamageService
│       ├── StateService
│       ├── MovementService
│       ├── RagdollService
│       └── Animation_Service
│
├── PlayerDataService
│
└── ProgressionService
│
├── TrainingService
├── ImpactTrainingService
└── Futuras actividades
```

---

# 3. Estructura de carpetas

## ServerScriptService

```text
ServerScriptService
│
├── Combat
│   │
│   ├── CombatManager.lua
│   │
│   ├── SkillRegistry
│   │
│   ├── Core
│   │   ├── SkillService.lua
│   │   ├── CooldownService.lua
│   │   ├── AutoAttackService.lua
│   │   ├── DamageService.lua
│   │   ├── StateService.lua
│   │   ├── MovementService.lua
│   │   ├── RagdollService.lua
│   │   └── Animation_Service.lua
│   │
│   └── Hitbox
│       └── HitboxService.lua
│
├── Player
│   │
│   ├── PlayerDataService.lua
│   │
│   └── Templates
│       ├── PlayerData.lua
│       └── Skill.lua
│
├── Progression
│   │
│   └── ProgressionService.lua
│
└── Training
│
├── TrainingService.lua
└── ImpactTrainingService.lua
```

Esta estructura representa la arquitectura propuesta/actual a partir de lo que fuimos desarrollando. Algunos módulos o carpetas pueden haber cambiado de nombre en Roblox Studio.

	---

	# 4. CombatManager

## Ubicación

ServerScriptService/Combat/CombatManager.lua

Es el punto de entrada principal del servidor para inicializar el sistema de combate.

	Actualmente es uno de los pocos Scripts que ejecutan código directamente. El resto son ModuleScripts.

	## Responsabilidades

- Inicializar los módulos de combate.
	- Conectar RemoteEvents.
	- Recibir solicitudes de ataque.
	- Delegar la ejecución al SkillService.
	- Inicializar servicios que necesitan comenzar a escuchar eventos.

	Ejemplo conceptual:

	```lua
SkillService:Init()

AttackRemote.OnServerEvent:Connect(function(player, slot, action)
	SkillService:Execute(player, slot, action)
end)
```

CombatManager no debería:

	- Calcular daño directamente.
	- Crear hitboxes directamente.
	- Ejecutar las timelines de habilidades.
	- Gestionar experiencia.
	- Gestionar animaciones individuales.

	Su función es conectar los sistemas.

	---

	# 5. SkillService

## Ubicación

ServerScriptService/Combat/Core/SkillService.lua

Es el encargado de validar y ejecutar las habilidades.

	## Responsabilidades

- Verificar si el jugador puede atacar.
	- Comprobar estados.
	- Comprobar cooldowns.
	- Determinar qué habilidad está equipada.
	- Obtener la habilidad del SkillRegistry.
	- Ejecutar la habilidad.
	- Gestionar habilidades activas.
	- Manejar liberación de habilidades Hold.
	- Cancelar habilidades cuando corresponda.

	## Flujo de ejecución

```text
CombatManager
│
▼
SkillService:Execute(player, slot, action)
│
├── Validar jugador
├── Validar Character
├── Validar estados
├── Validar cooldown
├── Obtener skill
│
▼
Skill.Execute()
│
▼
Skill execution
```

El SkillService no debería contener la lógica específica de cada habilidad.

	Por ejemplo, el SkillService no debería saber cómo funciona un Gattling Punch o un Ultimate Punch.

	---

	# 6. SkillRegistry

## Responsabilidad

Registrar y encontrar habilidades automáticamente.

	Las habilidades están organizadas mediante carpetas y cada una contiene:

	```text
SkillFolder
│
├── Skill.lua
└── Config.lua
```

Ejemplo:

	```text
Combat
└── Skills
└── Fist
└── Basic
├── Punch
│   ├── Skill.lua
│   └── Config.lua
│
├── Gattling Punch
│   ├── Skill.lua
│   └── Config.lua
│
└── Ultimate Punch
├── Skill.lua
└── Config.lua
```

## Path

Cada habilidad tiene un identificador:

	```lua
Fist/Basic/Punch
```

Este Path permite identificar la habilidad independientemente de dónde se encuentre equipada.

	El SkillRegistry se encarga de localizar y registrar los módulos.

	---

	# 7. Sistema de habilidades

Cada habilidad tiene dos componentes principales.

	## Skill.lua

Contiene la lógica de ejecución.

	Puede utilizar:

	- HitboxService
- DamageService
- StateService
- MovementService
- RagdollService
- Animation_Service
- PlayFX Remote

## Config.lua

Contiene parámetros configurables.

	Ejemplo:

	```lua
return {
	Path = "Fist/Basic/Punch",

	Cooldown = 1,

	AttackType = "OneShot",

	Damage = 10,

	Hitbox = {
		Size = Vector3.new(5, 5, 5),
		Offset = CFrame.new(0, 0, -5),
	}
}
```

La configuración debe permitir modificar una habilidad sin tener que reescribir su lógica.

	---

	# 8. Tipos de habilidades

## OneShot

Una habilidad que se ejecuta una vez y termina independientemente de si el jugador sigue presionando el botón.

	Ejemplo:

	Punch.

	Características:

	- No necesita Release.
	- No se cancela porque el jugador suelte el input.
	- Tiene una duración definida.
	- El servidor controla cuándo termina.

	## Hold

Una habilidad que continúa mientras el jugador mantiene presionado el input.

	Ejemplo:

	Gattling Punch.

	Características:

	- El cliente envía Execute.
	- El cliente envía Release cuando deja de presionar.
	- El servidor controla los golpes.
	- Puede cancelarse al soltar.
	- La secuencia no depende del cliente.

	## Habilidades con timeline

Las habilidades pueden utilizar una secuencia de eventos.

	Ejemplo:

	```text
Start
│
▼
Dash
│
▼
Grab
│
▼
Combo
│
▼
Throw
│
▼
End
```

El servidor determina cuándo sucede cada evento importante.

	Las animaciones y efectos visuales deben sincronizarse con esos eventos.

	---

	# 9. HitboxService

## Ubicación

ServerScriptService/Combat/Hitbox/HitboxService.lua

## Responsabilidad

Detectar colisiones durante una habilidad.

	No aplica daño.

	No decide si el objetivo está invulnerable.

	No decide si el jugador gana experiencia.

	Solo detecta qué partes o personajes entraron en la hitbox.

	## Funcionamiento

Utiliza:

	```lua
Workspace:GetPartBoundsInBox()
```

Y realiza consultas periódicas mediante:

	```lua
RunService.Heartbeat
```

La frecuencia utilizada actualmente es aproximadamente cada 0.05 segundos.

	## API

```lua
local hitbox = HitboxService:Create({
	Origin = character,
	Size = Vector3.new(5, 5, 5),
	Offset = CFrame.new(0, 0, -5),
	Attacker = character,
})
```

### Start

```lua
hitbox:Start()
```

Comienza a detectar impactos.

	### Stop

```lua
hitbox:Stop()
```

Detiene la detección.

	### Destroy

```lua
hitbox:Destroy()
```

Detiene y limpia la hitbox.

	## Callbacks

### OnHit

```lua
hitbox.OnHit = function(character)
	-- Objetivo detectado
end
```

Detecta personajes que tienen un Humanoid.

	### OnHitPart

Se agregó como extensión para detectar partes individuales.

	```lua
hitbox.OnHitPart = function(part)
	-- Parte detectada
end
```

Este callback permite que sistemas externos detecten objetos de entrenamiento, sin modificar la lógica principal del combate.

	---

	# 10. DamageService

## Responsabilidad

Centralizar el cálculo y aplicación del daño.

	El daño no debe aplicarse directamente desde el cliente.

	La habilidad solicita daño y el DamageService calcula el resultado.

	## Datos que puede recibir

```lua
{
	Amount = 10,

	Skill = "Fist/Basic/Punch",

	Attacker = player,

	Targets = targets,
}
```

## Escalado

El daño puede depender de:

	- Strength
- Armament Haki
- Style Level
- Defense
- Blocking
- Otros modificadores futuros

## Strength

El máximo de Strength establecido fue:

	```text
20,000
```

Se desea un crecimiento lineal hasta el máximo, evitando que el escalado haga que los jugadores de bajo nivel sean irrelevantes.

	## Style Level

Los estilos tienen:

	```text
Level: 1–300
```

El aumento máximo de daño por Style Level acordado anteriormente fue aproximadamente:

	```text
	+25% de daño final
```

El Style Level debe tener un impacto relativamente moderado en el daño.

	Su mayor utilidad es desbloquear habilidades.

	## Armament

El daño adicional por Armament solo debe aplicarse cuando el Haki está activo.

	El sistema consulta el estado del personaje mediante StateService.

	---

	# 11. StateService

## Ubicación

ServerScriptService/Player/StateService.lua

Es el sistema que registra los estados temporales o permanentes de un personaje.

	Actualmente almacena estados utilizando el Character como clave.

	## API

### Set

```lua
StateService:Set(
	character,
	"Attacking",
	true,
	duration
)
```

Para eliminar un estado:

	```lua
StateService:Set(
	character,
	"Attacking",
	false
)
```

## Has

```lua
StateService:Has(
	character,
	"Armament"
)
```

Devuelve true o false.

	## Estados utilizados o previstos

```text
Attacking
Stun
Blocking
Ragdoll
Invulnerable
CanStun
Armament
```

Los estados son independientes.

	Un personaje puede tener múltiples estados simultáneamente.

	## Tokens

Los estados con duración utilizan tokens para evitar que una duración antigua elimine un estado nuevo.

	Esto evita problemas cuando, por ejemplo, un Stun de 1 segundo es reemplazado por otro Stun de 3 segundos.

	---

	# 12. PlayerDataService

## Ubicación

ServerScriptService/Player/PlayerDataService.lua

Es el único módulo responsable de administrar y modificar los datos de los jugadores durante la sesión.

	## Responsabilidades

- Crear datos al entrar.
	- Eliminar datos al salir.
	- Obtener datos.
	- Modificar propiedades.
	- Sumar valores.
	- Aplicar límites.
	- Gestionar niveles de progresión.

	## API

### Add

```lua
PlayerDataService:Add(player)
```

Crea una copia profunda del template.

	### Remove

```lua
PlayerDataService:Remove(player)
```

Elimina los datos de la sesión.

	### Get

```lua
local playerData = PlayerDataService:Get(player)
```

Devuelve los datos actuales.

	### Update

```lua
PlayerDataService:Update(
	player,
	"Stats/Strength",
	100
)
```

Establece un valor.

	### SumTo

```lua
PlayerDataService:SumTo(
	player,
	"Stats/Strength",
	10
)
```

Incrementa un valor.

	---

	# 13. Data Template

## PlayerData

La estructura del jugador contiene:

	```lua
{
	SelectedSlot = 0,

	Hotbar = {},

	EquippedStyles = {},

	Styles = {},

	Stats = {},

	Haki = {},

	Items = {},
}
```

## Hotbar

Ejemplo:

	```lua
Hotbar = {
	[0] = "Fist/Basic/Punch",
	[1] = "Fist/Basic/Gattling Punch",
}
```

## EquippedStyles

```lua
EquippedStyles = {
	Fist = "Fist/Basic",

	Sword = nil,

	Gun = nil,

	DevilFruit = nil,
}
```

## Stats

```lua
Stats = {
	Strength = 1,

	Defense = 0,

	Agility = 1,
}
```

La incorporación de Endurance se había planteado para el entrenamiento físico, pero debe verificarse que exista actualmente en el template.

	## Haki

```lua
Haki = {
	Armament = {
		Level = 0,
		Experience = 0,
	},

	Observation = {
		Level = 0,
		Experience = 0,
	},

	Conqueror = {
		Level = 0,
		Experience = 0,
	},
}
```

## Styles

```lua
Styles = {
	["Fist/Basic"] = {
		Level = 1,
		Experience = 0,
		MaxLevel = 300,
	},
}
```

---

# 14. Path System

PlayerDataService utiliza paths para acceder a propiedades.

	Ejemplo:

	```lua
"Stats/Strength"
```

O:

	```lua
"Haki/Armament/Experience"
```

Para tablas con claves que contienen `/`:

	```lua
"Styles/[Fist/Basic]/Level"
```

Los corchetes permiten tratar `Fist/Basic` como una única clave.

	El sistema utiliza ParsePath para dividir el path y encontrar la propiedad.

	---

	# 15. Max Values

PlayerDataService tiene una función de limitación:

	```lua
ClampToMax(parent, property, value)
```

Busca automáticamente:

	```lua
parent["Max" .. property]
```

Por ejemplo:

	```lua
Strength = 100

MaxStrength = 20000
```

Cuando se utiliza SumTo:

	```lua
PlayerDataService:SumTo(
	player,
	"Stats/Strength",
	100000
)
```

El resultado nunca debería superar MaxStrength.

	Para Experience, el límite debe calcularse utilizando MaxLevel.

	---

	# 16. ProgressionService

## Ubicación

ServerScriptService/Progression/ProgressionService.lua

Este módulo centraliza el otorgamiento de recompensas de progresión.

	No detecta acciones.

	No crea hitboxes.

	No escucha inputs.

	No decide si un golpe acertó.

	Recibe una recompensa de otro sistema que ya validó la acción.

	## Flujo

```text
Actividad
│
▼
Validación
│
▼
ProgressionService
│
▼
PlayerDataService
│
▼
Experience
│
▼
Level
```

## Ejemplo

```lua
ProgressionService:AwardHakiTraining(
	player,
	"Armament",
	5
)
```

El ProgressionService solicita al PlayerDataService que sume experiencia.

	---

	# 17. Experiencia y niveles

Se decidió que los niveles sean enteros.

	El jugador no puede superar MaxLevel.

	La experiencia tampoco puede superar la cantidad necesaria para alcanzar MaxLevel.

	## Curva

El requisito de experiencia aumenta ligeramente con cada nivel.

	No es exponencial.

	Configuración propuesta:

	```lua
BASE_XP = 100

XP_GROWTH = 2
```

El requisito individual aumenta linealmente:

	```text
Nivel 0 → 1: 100 XP
Nivel 1 → 2: 102 XP
Nivel 2 → 3: 104 XP
Nivel 3 → 4: 106 XP
Nivel 4 → 5: 108 XP
```

La experiencia acumulada es cuadrática.

	## MaxLevel

Cada progresión tiene su propio límite.

	Ejemplo:

	```lua
Armament = {
	Level = 0,

	Experience = 0,

	MaxLevel = 300,
}
```

El sistema calcula automáticamente el XP máximo necesario.

	## Actualización

Cuando se modifica Experience:

	```lua
PlayerDataService:SumTo(
	player,
	"Haki/Armament/Experience",
	100
)
```

El PlayerDataService:

	1. Suma experiencia.
	2. Limita la experiencia.
	3. Calcula el nivel.
	4. Actualiza Level.

	El resto de los atributos no debería ejecutar cálculos de nivel.

	---

	# 18. TrainingService

## Ubicación

ServerScriptService/Training/TrainingService.lua

Es el sistema encargado del entrenamiento físico.

	El jugador utiliza V para comenzar o detener el entrenamiento.

	## Ejercicios

```text
PushUps → Strength
Squats → Agility
SitUps → Endurance
```

El entrenamiento rota entre ejercicios.

	## Repeticiones

La implementación propuesta:

	```text
2 repeticiones por ejercicio
```

Cada repetición dura aproximadamente:

	```text
1 segundo
```

## Flujo

```text
InputController
│
▼
TrainingController
│
▼
TrainingRemote
│
▼
TrainingService
│
▼
Repetición validada
│
▼
ProgressionService
│
▼
PlayerDataService
```

El servidor controla la repetición.

	El cliente solamente reproduce la animación.

	## Animaciones

Ubicación:

	```text
ReplicatedStorage
└── Assets
└── Animations
└── Training
└── Basic
├── PushUps
├── Squats
└── SitUps
```

---

# 19. ImpactTrainingService

## Estado

Sistema en desarrollo.

	Su objetivo es entrenar Armament Haki golpeando objetos.

	## Requisitos

- Armament debe estar activado.
	- El objeto debe ser válido.
	- El jugador debe cumplir los requisitos de la categoría.
	- El impacto debe haber sido detectado por el servidor.
	- Debe respetarse el cooldown.

	## Categorías

```text
Basic
Intermediate
Advanced
```

Los objetos tienen un Tag:

	```text
HakiTrainingObject
```

Y un Attribute:

	```text
Category
```

Ejemplo:

	```lua
Category = "Basic"
```

## Configuración propuesta

```lua
Basic = {
	RequiredStrength = 0,
	RequiredArmament = 0,

	Reward = 1,
	Damage = 0,
	Cooldown = 1,
}
```

```lua
Intermediate = {
	RequiredStrength = 500,
	RequiredArmament = 50,

	Reward = 3,
	Damage = 10,
	Cooldown = 1.5,
}
```

```lua
Advanced = {
	RequiredStrength = 2000,
	RequiredArmament = 200,

	Reward = 8,
	Damage = 30,
	Cooldown = 2,
}
```

Estos valores son configuraciones de prueba, no valores definitivos de balance.

	## Flujo

```text
Skill
│
▼
HitboxService
│
▼
OnHitPart
│
▼
ImpactTrainingService
│
├── Comprobar objeto
├── Comprobar Armament
├── Comprobar cooldown
├── Comprobar requisitos
│
▼
ProgressionService
│
▼
PlayerDataService
```

## Anti-exploit

El cliente no debe enviar:

	```text
Reward
Experience
Category
Target
```

El servidor debe determinar todas esas cosas.

	El cliente únicamente comunica que quiere ejecutar una habilidad.

	---

	# 20. Armament Haki

Armament tiene dos aspectos:

	## Datos

```lua
Haki.Armament.Level

Haki.Armament.Experience
```

## Estado activo

```lua
StateService:Has(
	character,
	"Armament"
)
```

El nivel de Armament representa la progresión.

	El estado Armament representa si está activado.

	No son lo mismo.

	## Activación

La activación se gestiona mediante el sistema de input y el Remote correspondiente.

	El servidor valida y cambia el estado.

	El cliente recibe información para reproducir los efectos visuales.

	## VFX

Ubicación aproximada:

	```text
ReplicatedStorage
└── Assets
├── Animations
├── Sounds
└── VFX
```

Los VFX de Haki estaban organizados bajo:

	```text
VFX/Haki/Armament
```

El sistema debe reproducir el efecto al activar y quitarlo al desactivar.

	---

	# 21. Client Architecture

## InputController

Detecta input del jugador.

	Por ejemplo:

	```text
MouseButton1 → Execute
T → Toggle Armament
V → Toggle Training
1–9 → Hotbar
```

El InputController no ejecuta combate directamente.

	Emite eventos a otros controladores.

	## CombatManager (Client)

Recibe input y organiza la interacción con el servidor.

	## VFXService

Se encarga de reproducir efectos visuales.

	## Animation_Service

Se encarga de reproducir animaciones.

	Las animaciones no deben decidir si un golpe acertó.

	El servidor decide el resultado del combate.

	---

	# 22. Remotes

Ubicación:

	```text
ReplicatedStorage
└── Remotes
```

Remotes utilizados o previstos:

	```text
Attack

PlayFX

ToggleArmament

TrainingRemote
```

## Attack

Client → Server.

	Comunica la acción de combate.

	Ejemplo:

	```lua
AttackRemote:FireServer(slot, action)
```

## PlayFX

Server → Client.

	Comunica eventos visuales.

	Puede incluir:

	```lua
{
	Skill = "Fist/Basic/Punch",

	Attacker = player,

	Targets = targets,
}
```

Los datos visuales deben representar un evento validado por el servidor.

	## TrainingRemote

Client → Server.

	Solicita iniciar o detener entrenamiento.

	No envía recompensas.

	---

	# 23. Progresión y balance

## Strength

Máximo establecido:

	```text
20,000
```

Debe crecer linealmente hasta el máximo.

	## Agility

Agility debería afectar ligeramente la velocidad de ejecución de ataques.

	Idea inicial:

	```text
Máximo Agility = 20,000

Máximo bonus de velocidad = 15%
	```

La velocidad debe afectar correctamente:

	- Animación.
	- Timeline.
	- Hitboxes.
	- Duraciones.
	- Estados.

	No solamente la velocidad de la animación.

	## Style Level

Máximo:

	```text
300
```

Bonus máximo de daño previsto:

	```text
25%
	```

Su principal beneficio debería ser desbloquear habilidades.

	## Haki

El Haki debería progresar mediante actividades apropiadas.

	No otorgar experiencia indiscriminadamente por cada hit de cualquier habilidad.

	Las habilidades MultiHit y AoE son particularmente vulnerables a explotación.

	---

	# 24. Decisiones de arquitectura

Estas decisiones son importantes y conviene mantenerlas.

	### 1. Servidor autoritativo

El cliente no determina:

	- Daño.
	- Targets válidos.
	- Experiencia.
	- Cooldowns.
	- Estados.
	- Recompensas.

	### 2. Separación de responsabilidades

HitboxService detecta.

	DamageService calcula daño.

	StateService gestiona estados.

	PlayerDataService gestiona datos.

	ProgressionService gestiona recompensas.

	TrainingService valida actividades.

	### 3. Habilidades modulares

Cada habilidad tiene su propio Skill.lua y Config.lua.

	### 4. No centralizar todo en CombatManager

CombatManager inicializa y conecta.

	No contiene toda la lógica.

	### 5. Progresión independiente del combate

El combate no necesita saber cómo se calcula un nivel.

	Simplemente solicita una recompensa cuando ocurre una acción válida.

	### 6. Configuración antes que hardcoding

Requisitos, recompensas, cooldowns y daños deben ser configurables.

	---

	# 25. Estado actual del proyecto

## Sistemas que ya existían o habían sido desarrollados

- CombatManager.
	- SkillService.
	- SkillRegistry.
	- HitboxService.
	- DamageService.
	- StateService.
	- PlayerDataService.
	- InputController.
	- Skills modulares.
	- Hotbar.
	- Sistema de estilos.
	- Sistema de Haki.
	- PlayFX.
	- Animaciones.
	- Cooldowns.
	- Sistema de estados.

	## Sistemas en desarrollo

- Progresión automática de Level a partir de Experience.
	- Curva de XP.
	- Entrenamiento físico.
	- ImpactTrainingService.
	- Entrenamiento de Armament Haki mediante objetos.
	- Balance de daño por Strength.
	- Balance de Style Level.
	- Agility como modificador de velocidad.

	## Problema reciente

Se observó un comportamiento extraño al incorporar la actualización automática de Level:

	- La progresión aparentemente funcionaba una vez.
	- En impactos posteriores no parecía ejecutarse nuevamente.
	- La activación/desactivación de Armament presentaba problemas visuales.
	- Al retirar la modificación, aparentemente todo volvía a funcionar.

	La causa todavía no fue identificada.

	Debe revisarse el Output y comprobar si existe algún error dentro de PlayerDataService, ProgressionService o la lógica de activación de Armament.

	---

	# 26. Próximos pasos recomendados

## Prioridad 1 — Estabilizar PlayerDataService

Confirmar que:

	- Add funciona.
	- Get devuelve los datos.
	- Update modifica correctamente.
	- SumTo modifica correctamente.
	- Experience no supera su máximo.
	- Level se actualiza.
	- Level es entero.
	- MaxLevel funciona.
	- No hay errores.

	## Prioridad 2 — Estabilizar Armament

Confirmar:

	- Activación.
	- Desactivación.
	- StateService.
	- VFX.
	- Daño.
	- No se interrumpe cuando se modifica Experience.

	## Prioridad 3 — Progresión

Completar:

	- ProgressionService.
	- Curva de XP.
	- AwardHakiTraining.
	- AwardPhysicalTraining.

	## Prioridad 4 — Entrenamiento físico

Completar:

	- TrainingService.
	- Animaciones.
	- Repeticiones.
	- Recompensas.
	- Cancelación.

	## Prioridad 5 — ImpactTrainingService

Completar:

	- Tags.
	- Categorías.
	- Requisitos.
	- Cooldowns.
	- Daño.
	- Recompensas.
	- Conexión con HitboxService.

	## Prioridad 6 — Balance

Probar con diferentes valores de:

	- Strength.
	- Defense.
	- Agility.
	- Style Level.
	- Armament Level.

	---

	# 27. Regla para futuras implementaciones

Cuando se implemente un nuevo sistema:

	1. Identificar quién detecta la acción.
	2. Identificar quién valida la acción.
	3. Identificar quién ejecuta el resultado.
	4. Identificar quién modifica los datos.
	5. Identificar quién presenta los efectos visuales.

	No crear módulos innecesarios si una responsabilidad ya está cubierta por un sistema existente.

	El objetivo es mantener una arquitectura modular, pero no fragmentarla excesivamente.
