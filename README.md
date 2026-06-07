# amalgame-net-socketio

Serveur **Socket.IO** (Engine.IO v4 / Socket.IO v5) côté serveur pour
[Amalgame](https://github.com/amalgame-lang/Amalgame), par-dessus les
WebSockets d'`amalgame-net-http`. **Pur-AM**, aucun runtime C.

Un `socket.io-client@4` (navigateur) se connecte **sans aucune
modification** et reçoit les events broadcastés par le serveur.

## v0.1 — périmètre

- Transport **WebSocket uniquement** (pas de polling HTTP).
- **Broadcast serveur → client** : `io.Emit("event", json)`.
- Suivi connect/disconnect (registre de connexions, mutex).
- Handshake Engine.IO/Socket.IO complet (`0{open}` → `40{connect}` → `42[...]`).

Hors périmètre v0.1 : rooms, acks, events client→serveur, namespaces multiples
(prévus, l'API est conçue pour grandir).

## Pourquoi WebSocket-only + port dédié

Le upgrade WebSocket d'`amalgame-net-http` ne donne pas accès à l'URL ni au
`sid` de la requête. Le handshake Engine.IO se fait donc entièrement sur la
WebSocket (le client `socket.io-client@4` tente `websocket` en premier avec
`transports: ['websocket','polling']`, donc il se connecte directement).
Le serveur tourne sur un **port dédié** ; en production, un reverse-proxy
route `/socket.io/` vers ce port :

```nginx
location /socket.io/ {
    proxy_pass http://127.0.0.1:3001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

## Usage

```amalgame
import Amalgame.Net.SocketIO

let io = SocketIO.New()
io.ServeBackground(3001)        // thread dédié (non bloquant)

// ... ailleurs, quand l'état change :
io.Emit("ranking:update", "{\"sub_tournament_id\":1,\"ranking\":[...]}")
```

`Emit` est thread-safe (registre gardé par mutex) : il peut être appelé
depuis le thread HTTP principal pendant que les connexions vivent sur le
thread du serveur WS.

## API

| Méthode | Description |
|---|---|
| `SocketIO.New()` | Construit une instance. |
| `.WithPingInterval(ms)` / `.WithPingTimeout(ms)` | Réglages Engine.IO (défaut 25000 / 20000). |
| `.Emit(event, jsonPayload) : int` | Broadcast `42["event",payload]` ; renvoie le nb de clients atteints. |
| `.Serve(port) : int` | Serveur WS bloquant (à lancer sur un thread dédié). |
| `.ServeBackground(port)` | Lance `Serve` sur un thread de fond. |

`SioProtocol` expose les builders de frames (`OpenPacket`, `ConnectPacket`,
`EventFrame`) — purs et testables.

## Dépendances

- `amalgame-net-http` (serveur WebSocket : `Ws.ServeMt`, `WsConn.*`)
- `amalgame-threading` (`Mutex`, `Thread.Spawn`)

## Licence

Apache-2.0.
