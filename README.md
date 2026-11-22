# mcp-redzilla

Servidor MCP de ejemplo con **20 usuarios en memoria** y cuatro herramientas para crear, listar, actualizar y eliminar usuarios. Está hecho en **TypeScript** con el SDK oficial de MCP v2, pensado para leerse y entender qué ocurre detrás de una tool.

---

## 1. Qué es MCP (en 30 segundos)

MCP (Model Context Protocol) es un protocolo abierto que permite a un cliente (Claude Code, OpenCode, Cursor, el MCP Inspector...) descubrir y ejecutar funciones en un servidor externo.

- El **host** lanza el cliente.
- El **cliente** habla JSON-RPC con el servidor.
- El **servidor** expone capacidades: **tools** (funciones), **resources** (datos legibles) y **prompts** (plantillas).

Este proyecto solo expone **tools**. El flujo es:

```
Cliente  --tools/list-->       Servidor   (descubre nombre, descripción y esquema de cada tool)
Cliente  --tools/call-->       Servidor   (ejecuta una tool con argumentos)
Servidor --CallToolResult-->   Cliente    (bloques de contenido: texto JSON, o isError)
```

Usamos **stdio**: el cliente arranca el servidor como un proceso y escribe/lee mensajes por stdin/stdout. Es el transporte más simple para un servidor local.

> Regla de oro: `stdout` es el canal del protocolo. Nunca escribas `console.log` en el servidor; todos los logs van a `console.error` (stderr).

---

## 2. Requisitos

- Node.js **>= 20** (SDK v2). El MCP Inspector necesita Node >= 22.23.4 o >= 24.22.1.
- No hay base de datos ni servicios externos.

---

## 3. Instalar y ejecutar

```bash
npm install
npm run dev          # arranca el servidor en stdio y espera a un cliente
```

Al arrancar verás solo el mensaje de stderr:

```
[mcp-redzilla] Servidor MCP iniciado en stdio
```

Es normal que no pase nada más: un servidor stdio está a la espera de que un cliente abra la conversación.

 Otros scripts:

```bash
npm run typecheck    # comprobación de tipos con TypeScript (no genera archivos)
npm run inspect      # abre el MCP Inspector apuntando a este servidor
```

En el Inspector, pulsa **Connect**, abre la pestaña **Tools** y ejecuta cualquiera de las cuatro herramientas. Es la forma más rápida de ver exactamente qué recibe el modelo.

---

## 4. Registrarlo en un cliente (OpenCode)

Añade esto a `opencode.json` (del proyecto) o a `~/.config/opencode/opencode.json` (global):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "redzilla-users": {
      "type": "local",
      "command": [
        "npx", "-y", "tsx",
        "C:/Users/camil/Desktop/Proyectos/mcp-redzilla/src/index.ts"
      ],
      "enabled": true
    }
  }
}
```

Después reinicia OpenCode. Si usas otro cliente, la idea es la misma: indicar el comando que lanza el servidor por stdio.

> En Windows usa barras `/` en la ruta del JSON. Si el comando no arranca, prueba con la ruta absoluta a `node.exe` y al script `node_modules/tsx/dist/cli.mjs`.

---

## 5. Herramientas expuestas

| Tool | Argumentos | Resultado |
|---|---|---|
| `listar_usuarios` | ninguno | `{ "total": 20, "usuarios": [...] }` |
| `crear_usuario` | `nombre`, `apellido`, `telefono`, `direccion`, `correo`, `estado?` | `{ "mensaje", "usuario" }` con id nuevo (21, 22, ...) y fechas actuales |
| `actualizar_usuario` | `id` + cualquier campo editable | `{ "mensaje", "usuario" }`; conserva `fechaCreacion`, actualiza `fechaModificacion` |
| `eliminar_usuario` | `id` | `{ "mensaje", "usuario" }` con el usuario borrado |

Reglas de negocio:

- `estado` solo puede ser `"activo"` o `"inactivo"`; al crear, el valor por defecto es `"activo"`.
- `correo` tiene formato válido y es **único** (la comparación ignora mayúsculas).
- Actualizar sin campos, usar un id inexistente o repetir un correo devuelve un resultado con `isError: true` y un mensaje legible.
- Los argumentos inválidos los rechaza el SDK antes de ejecutar el handler, usando el esquema de Zod.

Ejemplo de llamada `tools/call` a `crear_usuario`:

```json
{
  "name": "crear_usuario",
  "arguments": {
    "nombre": "Laura",
    "apellido": "Torres",
    "telefono": "+57 300 111 2233",
    "direccion": "Calle 8 #4-15, Bogotá",
    "correo": "laura.torres@example.com",
    "estado": "activo"
  }
}
```

---

## 6. Estructura del código

```
src/
├─ index.ts    → arranque: serveStdio(crearServidor)
├─ server.ts   → capa MCP: registro de tools y formato de respuestas
├─ types.ts    → tipos: Usuario + esquemas Zod de entrada
├─ store.ts    → Map<number, Usuario> y operaciones CRUD
└─ seed.ts     → los 20 usuarios iniciales
```

La separación importante es entre `server.ts` y `store.ts`:

- `store.ts` **no sabe nada de MCP**. Es un `Map` con copias defensivas para que nadie modifique el estado sin actualizar `fechaModificacion`. Mañana puedes cambiarlo por SQLite y las tools no se enteran.
- `server.ts` solo traduce entre JSON-RPC/MCP y el store. Aquí se decide qué es un error de negocio (`isError`) frente a un error de validación (Zod).
- `types.ts` contiene los esquemas de Zod. El SDK los convierte en JSON Schema, los publica en `tools/list` y valida cada llamada con ellos. De paso infieren el tipo del argumento del handler: no hay duplicidad entre validación y tipos.

Detalles que merecen leerse con calma:

1. `registerTool(nombre, config, handler)` registra cada herramienta. `config.inputSchema` es lo que ve el modelo; `description` es lo que lee para decidir cuándo usarla.
2. El handler devuelve `CallToolResult`: una lista de bloques `content` y, opcionalmente, `isError`.
3. `serveStdio(crearServidor)` crea una instancia del servidor por conexión y gestiona el transporte. Al cerrar stdin, el proceso termina.
4. Los 20 usuarios son datos fijos para que las pruebas sean reproducibles; el siguiente id se calcula solo (21).

---

## 7. Modelo de datos

```ts
interface Usuario {
  id: number;
  nombre: string;
  apellido: string;
  telefono: string;
  direccion: string;
  correo: string;
  estado: "activo" | "inactivo";
  fechaCreacion: string;      // ISO 8601
  fechaModificacion: string;  // ISO 8601
}
```

---

## 8. Limitaciones (importante si validas una idea)

- **Todo está en memoria.** Al reiniciar el proceso vuelven los 20 usuarios semilla. No hay persistencia.
- **Con stdio, cada cliente lanza su propio proceso.** Dos clientes distintos ven dos listas independientes. Para estado compartido habría que usar el transporte HTTP o guardar en disco/base de datos.
- **Concurrencia:** una sola conexión por proceso. Suficiente para aprender y probar; no es un servidor multiusuario.

---

## 9. Nota sobre la versión del SDK

Este proyecto usa **SDK v2**: paquetes `@modelcontextprotocol/server` y `@modelcontextprotocol/server/stdio`, API `registerTool` con `z.object(...)` y `serveStdio`.

Casi todos los tutoriales que encontrarás usan el **SDK v1** (`@modelcontextprotocol/sdk`, `new McpServer(...).tool(...)` y `new StdioServerTransport()`). Es la generación anterior: el protocolo es el mismo, cambian los nombres del paquete y un par de llamadas. No mezcles ejemplos de una versión con otra.

---

## 10. Comprobación rápida sin Inspector

Si quieres ver el protocolo crudo:

```bash
node -e "const {spawn}=require('node:child_process');const p=spawn(process.execPath,['node_modules/tsx/dist/cli.mjs','src/index.ts'],{stdio:['pipe','pipe','inherit']});p.stdout.on('data',d=>process.stdout.write(d));p.stdin.write(JSON.stringify({jsonrpc:'2.0',id:1,method:'initialize',params:{protocolVersion:'2025-11-25',capabilities:{},clientInfo:{name:'prueba',version:'1.0.0'}}})+'\n');setTimeout(()=>p.kill(),2000)"
```

Verás la respuesta JSON-RPC del `initialize` en stdout.
