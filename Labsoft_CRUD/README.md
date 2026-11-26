# LabSoft EPN · CRUD funcional en memoria (Taller 05)

> **Taller 05 · Aplicaciones Móviles · Escuela Politécnica Nacional**
> CRUD completo de **solicitudes de instalación de software** en la app LabSoft EPN, hecho con **NativeScript 9 + TypeScript**. Los datos se guardan **en memoria**, sin base de datos. Usa diálogos y Toast **nativos de Android** llamados directamente desde TypeScript.
>
> ✏️ **Autor(es):** Maldonado Shirley· **Fecha:** 28 noviembre de 2026

---

## 1. Objetivo

Construir una aplicación móvil que permita **crear, ver, editar y eliminar** (CRUD) solicitudes de instalación de programas en los laboratorios de la EPN. Toda la información vive en memoria mientras la app está abierta. Además, se demuestra el uso directo de clases nativas de Android (`AlertDialog`, `Toast`, `DatePickerDialog`) desde TypeScript, sin plugins.

## 2. El problema

Para instalar un programa en una computadora del laboratorio hace falta la contraseña de administrador, y solo la tiene el ayudante. Hoy el estudiante tiene que buscarlo en persona, no sabe si su pedido fue aceptado y a veces el programa ya estaba instalado en otro laboratorio.

**LabSoft EPN** ordena ese proceso: el estudiante envía una **solicitud**, el ayudante la **aprueba o la rechaza** y luego la marca como **instalada**. El estudiante recibe un **aviso** en cada paso.

## 3. ¿Qué significa "CRUD en memoria"?

**CRUD** son las 4 operaciones básicas sobre un dato:

| Letra | Operación | En LabSoft |
|---|---|---|
| **C** | Create (crear) | Enviar una nueva solicitud |
| **R** | Read (leer) | Ver la lista de mis solicitudes y el detalle de una |
| **U** | Update (actualizar) | Editar una solicitud que todavía está *Pendiente* |
| **D** | Delete (eliminar) | Borrar una solicitud que no esté instalada |

**"En memoria"** quiere decir que los datos se guardan en arreglos de TypeScript (`app/datos/store.ts`). Es como escribir en una pizarra: mientras la app está abierta todo se conserva, pero si se cierra por completo la pizarra se borra y vuelven los datos de ejemplo.

## 4. La entidad principal: Solicitud

```ts
interface Solicitud {
  id: number            // se genera solo (132, 133, …)
  programa: string      // ej. "Android Studio"
  lab: number           // laboratorio elegido
  pc: number            // computadora elegida (0 = "la primera libre")
  fecha: Date           // día en que se necesita
  motivo: string        // materia o razón
  urgente: boolean      // hay clase o prueba hoy
  estado: 'Pendiente' | 'Aprobada' | 'Instalada' | 'Rechazada'
  email: string         // quién la pidió
  enviada: Date
  motivoRechazo: string
}
```

Ciclo de vida de una solicitud:

```
 [Crear] → Pendiente ──Aprobar──→ Aprobada ──Marcar instalada──→ Instalada
               │  ▲                    
               │  └──── Deshacer (Snackbar, 4 s)
               └──Rechazar (con motivo)──→ Rechazada
```

## 5. Las operaciones CRUD, paso a paso

| Operación | Pantalla | Qué hace el usuario | Función en `store.ts` |
|---|---|---|---|
| **Create** | E5 Nueva solicitud | Escribe el programa, elige laboratorio, PC (E5d) y fecha (calendario E5b), y toca **Enviar solicitud** | `crear(datos)` |
| **Read** (lista) | E3 Mis solicitudes | Ve sus solicitudes con colores por estado y las filtra con chips | `listarMias(filtro)` |
| **Read** (una) | E4 Detalle | Toca una tarjeta y ve la línea de pasos | `obtener(id)` |
| **Update** | E6 Editar solicitud | En el detalle toca **Editar**, cambia datos y toca **Guardar cambios** | `actualizar(id, datos)` |
| **Delete** | E7 + E8 | Mantiene presionada una tarjeta (o toca **Eliminar**) → **ELIMINAR** → Toast | `eliminar(id)` |

### Reglas de negocio (validaciones)

- El **programa** no puede quedar vacío y no puede estar **ya instalado** en ese laboratorio. Ejemplo: "Python 3.13" en el Lab. 3 da error.
- Hay que elegir una **computadora** o marcar "la primera libre". Las PCs ocupadas aparecen desactivadas.
- La **fecha** no puede ser anterior a hoy: el calendario nativo bloquea los días pasados.
- El **motivo** es obligatorio.
- Solo se puede **editar** una solicitud *Pendiente*.
- No se puede **eliminar** una solicitud *Instalada*. La app avisa con el Toast "Ya está instalada: no se puede eliminar".
- Los errores se muestran en rojo **debajo de cada campo**.

## 6. Otros CRUD y acciones de la app

Además del CRUD principal, la app maneja otros datos en memoria:

| Quién | Qué datos | Operaciones |
|---|---|---|
| Estudiante (registro) | Cuenta de usuario | **Create**: R1 → R2, validado con un registro EPN simulado → R4 con un código de correo |
| Ayudante | Estado de las solicitudes de su laboratorio | Aprobar (con **Deshacer**), rechazar con motivo y marcar instalada. Cada acción crea un **aviso** para el estudiante |
| Ayudante | Estado del laboratorio | **Update**: presente, PCs libres, clase en curso, mantenimiento |
| Jefa | Laboratorios | **CRUD completo**: crear, listar, editar y eliminar (mantener presionado) |
| Jefa | Catálogo de software | **Create / Read / Update**: nombre, versión, categoría, permitido y laboratorios donde está instalado |
| Jefa | Usuarios | **Update** del rol con un diálogo de opciones de radio |

## 7. Componentes nativos de Android usados desde TypeScript

Todo está en `app/nativo/dialogos.ts`. Son clases de Android llamadas directamente desde TypeScript, sin plugins:

```ts
// E7 · ¿Eliminar solicitud?
const b = new android.app.AlertDialog.Builder(actividad)
b.setTitle('¿Eliminar solicitud?')
b.setMessage('Se cancelará la instalación…')
b.setPositiveButton('ELIMINAR', new android.content.DialogInterface.OnClickListener({ onClick: () => alAceptar() }))
b.setNegativeButton('CANCELAR', null)
b.create().show()

// E8 · Toast
android.widget.Toast.makeText(contexto, 'Solicitud eliminada', android.widget.Toast.LENGTH_SHORT).show()
```

| Clase nativa | Dónde aparece |
|---|---|
| `android.app.AlertDialog.Builder` | E7 eliminar, C4 cerrar sesión, R3 error de registro |
| `AlertDialog` + `EditText` (`setView`) | A4 escribir el motivo de rechazo |
| `AlertDialog` + `setSingleChoiceItems` | AD7 cambiar rol con opciones de radio |
| `android.app.DatePickerDialog` | E5b calendario con días pasados bloqueados |
| `android.widget.Toast` | E8 y todos los mensajes cortos |

En iOS esas clases no existen. Por eso cada función revisa `if (isAndroid)` y, si no es Android, usa los diálogos propios de NativeScript.

## 8. Arquitectura

```
Pantalla (.xml)  ──tap──▶  Código de la pantalla (.ts)  ──llama──▶  store.ts (arreglos en memoria)
      ▲                              │
      └──── Observable (binding) ◀───┘   y para avisar: nativo/dialogos.ts (AlertDialog / Toast)
```

- **Vista:** archivos `.xml` con los componentes.
- **Lógica de pantalla:** archivos `.ts`. Cada pantalla crea un `Observable` y la vista se actualiza sola con `{{ }}`.
- **Datos:** `datos/store.ts` es la única "base de datos": arreglos y funciones `crear`, `obtener`, `actualizar` y `eliminar`.
- **Navegación:** `comun/ui.ts` maneja el menú inferior, la flecha ←, el botón 🏠 y el modo oscuro.

```
app/
├── datos/store.ts        ← arreglos en memoria + funciones CRUD
├── nativo/dialogos.ts    ← AlertDialog, DatePickerDialog, Toast nativos
├── comun/                ← ui.ts (navegación) y perfil
├── entrada/  registro/   ← login y creación de cuenta
├── estudiante/           ← E1–E10 (aquí está el CRUD principal)
├── ayudante/             ← A1–A6
└── jefe/                 ← AD1–AD8
```

## 9. Capturas

| | | | | |
|:-:|:-:|:-:|:-:|:-:|
| ![Registro](capturas/01-registro.png) | ![Verificar](capturas/02-verificar.png) | ![Inicio](capturas/03-inicio-cuenta-nueva.png) | ![Calendario](capturas/04-create-calendario.png) | ![Enviada](capturas/05-create-enviada.png) |
| R1 Registro | R4 Verificar | E1 Cuenta nueva | **C**: calendario | **C** + **R**: enviada |
| ![Editar](capturas/06-update-editar.png) | ![Eliminar](capturas/07-delete-dialogo.png) | ![Toast](capturas/08-delete-toast.png) | ![Regla](capturas/09-regla-instalada.png) | ![Rechazar](capturas/10-rechazar.png) |
| **U**: editar | **D**: AlertDialog | **D**: Toast | Regla: instalada | A4 Rechazar |

## 10. Video de demostración

🎬 [**Ver el video del CRUD**](video/LabSoft_CRUD.mp4) (3 min 18 s)

Qué se ve en el video, en orden:
1. **Crear cuenta** (R1 → R2 → R4) y entrar como estudiante nuevo.
2. **Create:** nueva solicitud con laboratorio, PC, calendario y urgente → Toast "Solicitud enviada".
3. **Read:** la solicitud aparece en *Mis solicitudes* y se abre su detalle.
4. **Update:** editar fecha y motivo → Toast "Cambios guardados".
5. **Delete:** diálogo nativo **¿Eliminar solicitud?** → Toast "Solicitud eliminada". Una solicitud *Instalada* no se puede eliminar.
6. **Ayudante:** bandeja, aprobar, rechazar con motivo e historial.
7. **Cerrar sesión** con diálogo nativo.

## 11. Cómo ejecutarlo

Requisitos: Node.js, NativeScript CLI 9, Android Studio con un emulador (probado en *Medium Phone API 37*).

```bash
npm install
ns run android        # en Windows PowerShell usar: ns.cmd run android
```

| Dato de prueba | Valor |
|---|---|
| Estudiante | andy.m@epn.edu.ec |
| Ayudante (Lab. 3) | jorge.l@epn.edu.ec |
| Jefa | rosa.p@epn.edu.ec |
| Contraseña | cualquiera de 4 o más caracteres |
| Código único (registro) | cualquiera de 9 números que empiece con el año, ej. 202312345 (con 202010101 + FIS sale el error R3) |
| Código del correo (R4) | 482719 |

## 12. Conclusiones

- Con NativeScript se puede programar en TypeScript y aun así usar **componentes nativos reales** de Android. Se comprueba en **Layout Inspector**, donde aparecen `RecyclerView`, `EditText` y `AlertDialog`.
- Guardar los datos en memoria permite probar todo el CRUD **sin servidor ni base de datos**. La desventaja es que los datos se pierden al cerrar la app. El siguiente paso sería guardarlos en SQLite o en una API.
- Separar los **datos** (`store.ts`), lo **nativo** (`dialogos.ts`) y las **pantallas** hace que el código sea más fácil de leer y de cambiar.


