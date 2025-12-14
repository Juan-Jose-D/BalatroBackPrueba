# Backend Balatro Multiplayer

Backend para el juego multijugador Balatro, implementado con Spring Boot y WebSocket. Actúa como **intermediario de comunicación en tiempo real** entre jugadores, con lógica de progreso y detección de victoria.

## 🎯 Funcionalidades Principales

### Comunicación en Tiempo Real
- **WebSocket con STOMP**: Comunicación bidireccional entre jugadores
- **Reenvío de mensajes**: Todos los mensajes se reenvían automáticamente a ambos jugadores
- **Señalización WebRTC**: Soporte para chat de voz entre jugadores

### Emparejamiento
- **Matchmaking automático**: Cola FIFO que empareja jugadores automáticamente
- **Salas privadas**: Creación de salas con código único (6 caracteres) para jugar con amigos

### Lógica de Progreso y Victoria ⭐
- **Seguimiento de progreso**: El backend mantiene el progreso de cada jugador (ante y blind)
- **Detección de victoria automática**: 
  - Detecta cuando un jugador supera al oponente que se quedó sin manos
  - Envía `GAME_WON` y `GAME_LOST` automáticamente
- **Manejo de empates**: Detecta cuando ambos jugadores se quedan sin manos en el mismo ante/blind
- **Comparación de progresos**: Compara ante y blind para determinar quién está más adelante
- **Timeout**: Maneja cuando un jugador se queda sin tiempo (15 segundos)

### Autenticación
- **AWS Cognito**: Autenticación JWT integrada
- **Validación de tokens**: Verificación automática en cada conexión WebSocket

## 🏗️ Arquitectura

```
Cliente 1 ←→ [Backend Spring Boot] ←→ Cliente 2
              ├─ Matchmaking
              ├─ Room Service
              ├─ Game Service (progreso + victoria)
              ├─ WebRTC Signaling
              └─ Session Management
```

## 📡 Endpoints WebSocket

### Matchmaking
- `/app/matchmaking/join` - Unirse a cola
- `/app/matchmaking/leave` - Salir de cola
- Suscripción: `/user/queue/matchmaking`

### Salas Privadas
- `/app/room/create` - Crear sala
- `/app/room/join` - Unirse a sala con código
- Suscripción: `/user/queue/room`

### Mensajes de Juego
- `/app/game/{gameId}` o `/app/game/{gameId}/message` - Enviar mensaje
- `/app/game/{gameId}/chat` - Enviar chat
- `/app/game/{gameId}/emote` - Enviar emote
- Suscripción: `/topic/game/{gameId}`

### WebRTC
- `/app/webrtc/signal` - Señalización WebRTC
- Suscripción: `/user/queue/webrtc/{gameId}`

## 🔄 Tipos de Mensajes Importantes

### ROUND_COMPLETE
Cuando un jugador completa una ronda, el backend:
1. Actualiza el progreso del jugador (ante, blind)
2. Verifica condiciones de victoria
3. Si hay victoria, envía `GAME_WON` y `GAME_LOST` automáticamente
4. Reenvía el mensaje a ambos jugadores

**Formato:**
```json
{
  "type": "ROUND_COMPLETE",
  "gameId": "...",
  "playerId": "...",
  "payload": {
    "action": "ROUND_COMPLETE",
    "data": {
      "ante": 2,
      "blind": "small",
      "score": 500
    }
  }
}
```

### GAME_LOST (no_hands)
Cuando un jugador se queda sin manos:
1. El backend registra el ante/blind donde se quedó sin manos
2. Verifica si el oponente ya está más adelante (victoria inmediata)
3. Si no, espera a que el oponente avance para verificar victoria
4. Reenvía el mensaje a ambos jugadores

**Formato:**
```json
{
  "type": "GAME_LOST",
  "gameId": "...",
  "playerId": "...",
  "payload": {
    "action": "GAME_LOST",
    "data": {
      "reason": "no_hands",
      "ante": 1,
      "blind": "big"
    }
  }
}
```

### GAME_WON / GAME_LOST
El backend envía estos mensajes automáticamente cuando detecta victoria:
- `GAME_WON` al ganador con `reason: "opponent_no_hands"` o `"opponent_timeout"`
- `GAME_LOST` al perdedor con `reason: "no_hands"` o `"timeout"`

### TIME_OUT
Cuando un jugador se queda sin tiempo (cronómetro de 15s):
- El backend reenvía el mensaje a ambos jugadores
- El oponente recibe la notificación de victoria

### Empate
Cuando ambos jugadores se quedan sin manos en el mismo ante/blind:
- El backend detecta el empate automáticamente
- Envía `GAME_LOST` con `reason: "tie"` a ambos jugadores

## 🚀 Instalación Rápida

### Requisitos
- Java 17+
- Maven 3.8+
- AWS Cognito configurado

### Pasos

1. **Clonar y configurar**
```bash
git clone <repo-url>
cd ARSW-Proyecto-Backend
```

2. **Configurar AWS Cognito** en `src/main/resources/application.properties`:
```properties
aws.cognito.userPoolId=tu-user-pool-id
aws.cognito.region=us-east-1
aws.cognito.jwkUrl=https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json
```

3. **Ejecutar**
```bash
mvn spring-boot:run
```

Servidor disponible en `http://localhost:8080`, WebSocket en `ws://localhost:8080/ws`

## 🔐 Autenticación

El backend requiere token JWT de AWS Cognito en el header `Authorization` del mensaje STOMP CONNECT:

```javascript
const client = new Client({
    webSocketFactory: () => new SockJS('http://localhost:8080/ws'),
    connectHeaders: {
        'Authorization': `Bearer ${token}` // Token de Cognito
    }
});
```

## 🧪 Pruebas

```bash
# Ejecutar pruebas
mvn test

# Generar reporte de cobertura
mvn clean verify
# Reporte en: target/site/jacoco/index.html
```

## 📊 Características Técnicas

### Configuración

**Archivo:** `application.properties`

```properties
# Puerto del servidor
server.port=8080

# Orígenes permitidos para WebSocket y CORS
spring.websocket.servlet.allowed-origins=http://localhost:3000,http://localhost:5173
cors.allowed-origins=http://localhost:3000,http://localhost:5173

# Tamaño máximo de mensajes WebSocket (64KB)
spring.websocket.message-size-limit=65536

# Nivel de logging
logging.level.com.arsw.balatro=DEBUG
```

**Modificar orígenes permitidos:**
Agregar los puertos donde correrá tu cliente al campo `allowed-origins`.

## 🧪 Prueba del Sistema Simplificado

### Cliente de Prueba en JavaScript

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test Balatro Backend (Simplificado)</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7/bundles/stomp.umd.min.js"></script>
</head>
<body>
    <h1>Balatro Backend Test - Intermediario Puro</h1>
    <button onclick="connect()">Conectar</button>
    <button onclick="joinQueue()">Unirse a Cola</button>
    <button onclick="sendTestMessage()">Enviar Mensaje de Prueba</button>
    <pre id="log"></pre>

    <script>
        let stompClient = null;
        let currentGameId = null;
        const playerId = 'test-' + Math.random().toString(36).substr(2, 9);

        function connect() {
            const socket = new SockJS('http://localhost:8080/ws');
            stompClient = Stomp.over(socket);

            stompClient.connect({}, function(frame) {
                log('✅ Conectado al servidor');

                // Suscribirse a notificaciones de matchmaking
                stompClient.subscribe('/user/queue/matchmaking', function(message) {
                    log('📬 Matchmaking: ' + message.body);
                    const data = JSON.parse(message.body);
                    if (data.type === 'MATCH_FOUND') {
                        currentGameId = data.gameId;
                        subscribeToGame(data.gameId);
                    }
                });
            });
        }

        function joinQueue() {
            stompClient.send('/app/matchmaking/join', {}, JSON.stringify({
                type: 'JOIN_QUEUE',
                playerId: playerId
            }));
            log('📤 Enviado JOIN_QUEUE con ID: ' + playerId);
        }

        function subscribeToGame(gameId) {
            log('🎮 Suscrito al juego: ' + gameId);
            stompClient.subscribe(`/topic/game/${gameId}`, function(message) {
                const data = JSON.parse(message.body);
                log('📥 Mensaje recibido: ' + JSON.stringify(data, null, 2));
            });
        }

        function sendTestMessage() {
            if (!currentGameId) {
                log('❌ No estás en un juego. Une a la cola primero.');
                return;
            }
            
            // Enviar mensaje genérico al oponente
            stompClient.send(`/app/game/${currentGameId}/message`, {}, JSON.stringify({
                type: 'GAME_MESSAGE',
                playerId: playerId,
                gameId: currentGameId,
                payload: {
                    action: 'TEST_ACTION',
                    data: 'Mensaje de prueba desde ' + playerId,
                    timestamp: Date.now()
                }
            }));
            log('📤 Mensaje de prueba enviado');
        }

        function log(msg) {
            const logEl = document.getElementById('log');
            logEl.textContent += new Date().toISOString() + ': ' + msg + '\n';
            logEl.scrollTop = logEl.scrollHeight;
        }
    </script>
</body>
</html>
```

**Cómo probar:**
1. Abre el archivo en **DOS navegadores diferentes**
2. Haz clic en "Conectar" en ambos
3. Haz clic en "Unirse a Cola" en ambos
4. Deberías ver "MATCH_FOUND" en ambos navegadores
5. Haz clic en "Enviar Mensaje de Prueba" en uno
6. **Ambos navegadores** deberían recibir el mensaje (el que envía y el que recibe)
7. Verifica que el backend **solo reenvía** el mensaje sin modificarlo

## 💡 Decisiones de Diseño Simplificado

### ¿Por qué un intermediario puro?

Este backend fue **drásticamente simplificado** para eliminar toda lógica de juego:

1. **Separación de responsabilidades:**
   - Backend: Solo comunicación y matchmaking
   - Cliente: TODA la lógica del juego, validaciones, estado
   - Ventaja: Los clientes pueden implementar variaciones del juego sin cambiar el backend

2. **Escalabilidad:**
   - El backend no procesa nada complejo, solo reenvía bytes
   - Puede manejar miles de partidas simultáneas
   - El cuello de botella está en el cliente, no en el servidor

3. **Desarrollo desacoplado:**
   - Frontend y backend pueden desarrollarse independientemente
   - Cambios en reglas del juego no requieren deploy del backend
   - Facilita pruebas y debugging (cada cliente es autónomo)

### ¿Por qué sin base de datos?

1. **No hay nada que persistir:**
   - El backend solo guarda `gameId + player1Id + player2Id`
   - No hay cartas, puntajes, ni estado de juego
   - Las partidas son sesiones temporales de comunicación

2. **Simplicidad extrema:**
   - No requiere configuración de base de datos
   - Despliegue trivial (un solo JAR)
   - Menos latencia (todo en memoria)

**Limitación:** Si el servidor se reinicia, las partidas activas se pierden. Para producción, agregar Redis solo para el registro de partidas.

### ¿Por qué STOMP sobre WebSocket puro?

STOMP simplifica enormemente el desarrollo:

1. **Routing automático:** `/topic/game/{gameId}` enruta mensajes automáticamente
2. **Suscripciones:** Los clientes se suscriben y reciben solo mensajes relevantes
3. **Broadcasts:** Un mensaje a `/topic/game/{gameId}` llega a ambos jugadores
4. **Compatibilidad:** Librerías en todos los lenguajes (JS, Java, Python, etc.)

### ¿Por qué thread-safe collections?

Múltiples jugadores conectándose simultáneamente generan:

- Múltiples threads procesando matchmaking
- Accesos concurrentes al registro de partidas
- Mensajes simultáneos de diferentes partidas

`ConcurrentHashMap` y `ConcurrentLinkedQueue` garantizan thread-safety sin locks manuales.

## 📊 Flujo de Mensajes Simplificado

### Tipos de Mensajes (MessageType) - Simplificados

```java
// Matchmaking
JOIN_QUEUE          // Unirse a cola
LEAVE_QUEUE         // Salir de cola
MATCH_FOUND         // Match encontrado

// Mensajes de juego (genéricos)
GAME_MESSAGE        // Mensaje genérico que el backend reenvía sin procesar

// Comunicación
CHAT_MESSAGE        // Mensaje de chat
PLAYER_EMOTE        // Emote

// Eventos
PLAYER_CONNECTED    // Jugador conectado
PLAYER_DISCONNECTED // Jugador desconectado

// Keep-alive
PING                // Ping de cliente
PONG                // Respuesta del servidor

// Errores
ERROR               // Error genérico
```

**Eliminados:**
- ❌ `GAME_START`, `GAME_END`, `ROUND_START`, `ROUND_END` - Ya no se manejan lifecycle events
- ❌ `PLAY_HAND`, `DISCARD_CARDS`, `BUY_ITEM`, `SELL_ITEM`, `REROLL_SHOP` - Ya no se procesan acciones específicas
- ❌ `GAME_STATE_UPDATE`, `PLAYER_STATE_UPDATE`, `OPPONENT_ACTION`, `SHOP_UPDATE` - Ya no se gestiona estado
- ❌ `INVALID_ACTION` - Ya no se validan acciones

**Nuevo:**
- ✅ `GAME_MESSAGE` - Mensaje universal que los clientes usan para comunicarse

### Fases del Juego (GamePhase) - ELIMINADO

El enum `GamePhase` fue completamente eliminado. El backend ya no rastrea fases del juego.

**Los clientes manejan sus propias fases/estados localmente.**

## 🔐 Manejo de Errores (Simplificado)

### Validaciones Implementadas (Mínimas)

El backend solo valida lo mínimo necesario para funcionar como intermediario:

1. **Jugador no en el juego:**
   - Al intentar enviar un mensaje, se verifica que el playerId pertenezca a la partida
   - Si no pertenece, se envía error y no se reenvía el mensaje

2. **Partida no encontrada:**
   - Si el gameId no existe, se envía error al cliente

**Eliminadas todas las validaciones de lógica de juego:**
- ❌ Ya no se valida manos restantes
- ❌ Ya no se valida descartes restantes
- ❌ Ya no se valida dinero suficiente
- ❌ Ya no se valida fase del juego
- ❌ Ya no se valida turno del jugador

**Los clientes son responsables de validar sus propias reglas de juego.**

### Manejo de Desconexiones (Simplificado)

1. **Desconexión detectada:**
   - `WebSocketEventListener` captura el evento
   - Remueve jugador de cola de matchmaking
   - Notifica al oponente con `PLAYER_DISCONNECTED`

2. **Limpieza de memoria:**
   - Después de 60 segundos, se limpia el juego de memoria
   - **No se declara ganador** - el cliente decide qué hacer

**El cliente decide:**
- Si espera reconexión del oponente
- Si declara victoria por abandono
- Cuánto tiempo esperar
- Cómo notificar al usuario

## 🎯 Extensiones Futuras (Para el Backend Simplificado)

Aunque el backend actual es ultra-ligero, se podrían agregar:

### 1. Persistencia con Redis
- Mantener registro de partidas en Redis
- Permite reiniciar el servidor sin perder partidas activas
- Habilita escalado horizontal (múltiples instancias del backend)

### 2. Autenticación
- Integrar Spring Security con JWT
- Validar tokens en la conexión WebSocket
- Asociar sesiones a usuarios autenticados

### 3. Salas Privadas
- Matchmaking con código de sala
- Permitir que amigos se conecten directamente
- No requiere lógica de juego adicional

### 4. Métricas y Monitoreo
- Tiempo promedio de matchmaking
- Número de partidas activas
- Duración promedio de partidas
- **Sin procesar datos de juego**, solo métricas de infraestructura

### 5. Reconexión Inteligente
- Permitir reconexión a partida activa
- Mantener gameId en sesión
- Notificar al oponente sobre reconexión

**NO se recomienda agregar:**
- ❌ Validación de reglas de juego (rompe el propósito de cliente grueso)
- ❌ Procesamiento de lógica de juego (debe estar en el cliente)
- ❌ Almacenamiento de estado detallado del juego (solo en cliente)

---

## Despliegue

Nuestro despliegue está realizado en AWS

![instancias](/img/instancias.png)

![targetGroupsALB](/img/targetGroupALB.png)

![alb](/img/alb.png)

![albHealthCheck](/img/alb-healthCheck.png)

### Impedimentos

Al estar usando una cuenta  de aws instructure, una cuenta de aprendizaje; no nos es posible hacer el CI/CD correctamente, ya que cada vez que abrimos el laboratorio se crean credenciales nuevas, y toca actualizar las variable de etorno usadas en el proceso de despliegue del proyecto.

### Pruebas de carga

```sh
load-tests/
├── pom.xml                          # Configuración Maven con Gatling
├── README.md                        # Documentación completa
├── run-all-tests.ps1               # Script para ejecutar todas las pruebas
└── src/test/
    ├── scala/simulations/
    │   ├── BasicLoadTest.scala     # Prueba básica (50 usuarios)
    │   ├── StressTest.scala        # Prueba de estrés (300 usuarios pico)
    │   └── LoadBalancerTest.scala  # Prueba de balanceo
    └── resources/
        └── gatling.conf            # Configuración de Gatling
```


## 📝 Conclusión

Este backend implementa un **sistema ultra-ligero y eficiente** como **intermediario puro** de comunicación en tiempo real.

**Características clave:**
- ✅ WebSocket con STOMP para comunicación bidireccional
- ✅ Matchmaking automático FIFO
- ✅ **Solo reenvío de mensajes** sin procesamiento de lógica
- ✅ Thread-safe para múltiples conexiones simultáneas
- ✅ Sin base de datos (todo en memoria)
- ✅ Limpieza automática de partidas inactivas
- ✅ Manejo robusto de desconexiones
- ✅ Logging detallado para diagnóstico

## 💡 Lógica de Progreso y Victoria

El backend mantiene estado mínimo pero crítico:

### Estado por Partida
- Progreso actual de cada jugador (ante, blind)
- Punto donde cada jugador se quedó sin manos (si aplica)
- Estado del juego (terminado, empate, ganador)

### Comparación de Progresos
- Compara ante primero, luego blind si el ante es igual
- Orden de blinds: `small` < `big` < `boss`

### Detección de Victoria
1. **Victoria inmediata**: Si el oponente ya está más adelante cuando un jugador se queda sin manos
2. **Victoria por progreso**: Si un jugador supera el ante/blind donde el oponente se quedó sin manos
3. **Empate**: Si ambos se quedan sin manos en el mismo ante/blind

---

**Proyecto ARSW - Arquitecturas de Software**  
Universidad Escuela Colombiana de Ingeniería Julio Garavito

**Equipo:**
- Samuel Alejandro Prieto Reyes
- Josué David Hernández Martínez
- Juan José Díaz Gómez
