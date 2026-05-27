# Refactor: roles, estados y reportes

Este documento resume el diseño de base de datos normalizado, la lógica de negocio y el módulo de reportes.

## Base de datos

### Tabla `roles`

Catálogo de roles (`admin`, `empleado`, `cliente`). La tabla `usuarios` usa `id_rol` como clave foránea; el campo de texto `rol` fue eliminado.

### Tabla `estados`

Catálogo unificado con `nombre` y `tipo` (`usuario`, `producto`, `pedido`, `pago`). Permite reutilizar nombres lógicos en distintos dominios (por ejemplo `activo` para usuario y para producto) sin mezclar significados.

Las tablas `usuarios`, `productos`, `pedidos` y `pagos` referencian `id_estado` en lugar de columnas enum o texto.

### Usuarios y autenticación

La tabla `usuarios` incluye `email_verified_at` y `remember_token` para compatibilidad con el flujo estándar de Laravel (verificación de correo, “recordarme”, restablecimiento de contraseña). El modelo `User` implementa `MustVerifyEmail`.

### Integridad referencial

Las FK usan `restrictOnDelete` en tablas de negocio hacia catálogos, evitando borrar roles/estados mientras existan filas dependientes, y `cascadeOnUpdate` donde aplica para mantener IDs alineados.

## Flujo de pedidos y estados

Estados de pedido sembrados (tipo `pedido`): pendiente → confirmado → preparando → listo → entregado; y `cancelado` como estado terminal alternativo.

- Al **crear** un pedido se valida stock y se descuenta; se registran movimientos de stock según la lógica del `PedidoController` y servicios asociados.
- La **entrega** exige pago en estado confirmado (reglas en controladores de pedido/pago).

### Cancelación de pedidos (detalle)

Toda la lógica vive en `App\Services\PedidoCancelacionService::cancelar()` dentro de una **transacción** (`DB::transaction`):

1. Se bloquea el registro del pedido (`lockForUpdate`). Si ya está **cancelado**, el método termina sin efecto (**idempotente**: no duplica devoluciones de stock).
2. Si el pedido está **entregado**, se lanza `DomainException` y no se modifica nada.
3. Se actualiza el pedido a estado **cancelado** (no se borra la fila; queda historial).
4. Por cada fila de `detalle_pedido` se incrementa el stock del producto (bloqueo por fila) y se inserta en `movimientos_stock`: **tipo** `entrada`, **motivo** `cancelacion`, **fecha** actual.
5. Si existe pago asociado: **pendiente** → estado pago **cancelado** y `fecha_pago` en `null`; **confirmado** → **requiere_devolucion** (sin borrar el pago).
6. Se registra un `Log::info` con `pedido_id`, `codigo` y usuario autenticado.

**Entrada HTTP**

- `POST /pedidos/{pedido}/cancelar` → `PedidoController::cancelar` (solo **admin** y **empleado**). Botones en listado y detalle de pedido.
- Sigue vigente cambiar el select a **cancelado** y guardar (`PUT /pedidos/{pedido}/estado`), que delega en el mismo servicio.

**Reportes**

- Ventas e ingresos: solo pedidos con `id_estado` distinto de cancelado (ya aplicado en `ReporteController`, `DashboardController` y ranking de productos).
- En **reporte de ventas** (admin) se muestra además el monto **devoluciones pendientes (caja)**: suma de `pagos.monto` en estado `requiere_devolucion` ligados a pedidos cancelados en el rango de fechas del reporte (obligación de egreso en caja). El CSV de ventas incluye una fila final con ese total.

## Pagos

Los pagos inician en pendiente; solo personal con rol adecuado confirma mediante la ruta de confirmación. No hay confirmación automática al crear el pedido.

## Control de acceso

Middleware `role:...` resuelve el rol por `id_rol` y el nombre en `roles`. Las rutas de reportes, cambio de estado de pedidos, **cancelación explícita de pedidos**, confirmación de pagos y CRUD restringido se agrupan por rol en `routes/web.php`. Los clientes no pueden invocar la cancelación (no tienen la ruta).

## Módulo de reportes

Rutas bajo el prefijo `/reportes` y nombres `reportes.*`:

| Ruta / nombre | Quién | Contenido |
|---------------|-------|-----------|
| `reportes.index` | Autenticados | Índice de reportes disponibles según rol |
| `reportes.ventas`, `ventas.csv`, `reportes.pagos`, `reportes.usuarios` | Admin | Ventas por rango, export CSV de ventas, pagos, usuarios |
| `reportes.pedidos`, `productos-vendidos`, `stock` | Admin y empleado | Pedidos filtrables, ranking de productos, stock y movimientos (empleado con alcance acotado en vistas/controlador) |
| `reportes.cliente` | Cliente | Mi actividad / historial de pedidos |

Las consultas usan Eloquent y relaciones (`Pedido`, `DetallePedido`, `Producto`, `Pago`, `MovimientoStock`, `User`, `Estado`).

## Seeders

- `RolSeeder`: roles base.
- `EstadoSeeder`: estados por tipo, incluyendo pedidos y pagos (incl. `requiere_devolucion`).
- `DatabaseSeeder`: usuario admin de demo, categoría y producto de ejemplo (ajustar credenciales en entorno real).

## Comando de verificación

```bash
php artisan migrate:fresh --seed
php artisan test
```

**Nota:** Si las rutas no coinciden con el código (por ejemplo falta `reportes.*`), ejecutar `php artisan route:clear` para eliminar caché de rutas obsoleta en `bootstrap/cache`.
