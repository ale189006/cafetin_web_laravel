# Hallazgos de Seguridad (Priorizados)

---

## 1) 🔴 Crítico — Escalada de privilegios en cambio de estado de pedidos

Cualquier usuario autenticado (incluido cliente) puede invocar la ruta de cambio de estado de pedidos, porque está fuera del middleware de rol.

**`web.php` — Líneas 25-37**

```php
Route::resource('pedidos', PedidoController::class)->except(['edit', 'update', 'destroy']);
Route::put('/pedidos/{pedido}/estado', PedidoEstadoController::class)->name('pedidos.estado');
Route::middleware('role:admin,empleado')->group(function () {
    Route::resource('categorias', CategoriaController::class)->except(['show']);
    Route::resource('productos', ProductoController::class)->except(['show']);
});
```

**Método de vulnerabilidad (explotación):**

- Usuario cliente autenticado envía `PUT /pedidos/{id}/estado` con `estado=listo|cancelado|entregado`.
- Puede alterar estados operativos sin permiso (**Broken Access Control / Privilege Escalation**).

---

## 2) 🟠 Alto — IDOR / Exposición de pedidos de otros usuarios

No hay control de ownership en `index`/`show` de pedidos para clientes; se listan y consultan pedidos globales.

**`PedidoController.php` — Líneas 20-30**

```php
public function index(Request $request): View
{
    $query = Pedido::with(['usuario', 'pago'])->latest();
    if ($request->filled('fecha_inicio') && $request->filled('fecha_fin')) {
        $query->whereBetween('fecha', [$request->fecha_inicio.' 00:00:00', $request->fecha_fin.' 23:59:59']);
    }
    return view('pedidos.index', [
        'pedidos' => $query->paginate(12)->withQueryString(),
    ]);
}
```

**`PedidoController.php` — Líneas 125-129**

```php
public function show(Pedido $pedido): View
{
    $pedido->load(['usuario', 'detalles.producto', 'pago']);
    return view('pedidos.show', compact('pedido'));
}
```

**Método de vulnerabilidad:**

- Cliente prueba IDs secuenciales (`/pedidos/1`, `/pedidos/2`, ...).
- Accede a datos de otros clientes (nombre, total, estado, pago).

---

## 3) 🟠 Alto — Usuario inactivo puede seguir iniciando sesión

Se valida contraseña/email, pero no se restringe `estado=activo` durante autenticación.

**`LoginRequest.php` — Líneas 41-54**

```php
public function authenticate(): void
{
    $this->ensureIsNotRateLimited();
    if (! Auth::attempt($this->only('email', 'password'), $this->boolean('remember'))) {
        RateLimiter::hit($this->throttleKey());
        throw ValidationException::withMessages([
            'email' => trans('auth.failed'),
        ]);
    }
    RateLimiter::clear($this->throttleKey());
}
```

**Método de vulnerabilidad:**

- Admin marca usuario como inactivo, pero ese usuario aún puede autenticarse si conoce su contraseña.
- Control de estado queda sólo "visual", no de seguridad.

---

## 4) 🟡 Medio — Falta de rate limiting en acciones sensibles

Sólo login tiene limitación por intentos. Operaciones de negocio críticas no tienen throttling (crear pedido, cambiar estado, confirmar pago).

**Método de vulnerabilidad:**

- Script automatizado dispara cientos de requests para saturar operaciones, generar ruido operativo o manipular estados en ráfaga.

---

## 5) 🟡 Medio — Hardening incompleto de seguridad HTTP / sesión

No hay evidencia de cabeceras defensivas (CSP, HSTS, X-Frame-Options, etc.) ni cookies forzadas `Secure` para producción. Además hay script inline en dashboard (dificulta CSP estricta).

**`dashboard.blade.php` — Líneas 56-58**

```html
<script>
    setTimeout(() => window.location.reload(), 30000);
</script>
```

**`.env` — Líneas 30-33**

```env
SESSION_DRIVER=database
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
```

**`.env` — Líneas 4-5**

```env
APP_DEBUG=true
APP_URL=http://localhost
```

**Método de vulnerabilidad:**

- Sin cabeceras: mayor superficie ante clickjacking y endurecimiento insuficiente del navegador.
- En despliegue real, `APP_DEBUG=true` puede filtrar internals ante errores.

---

## Riesgos Frontend Observados

- **XSS reflejado/almacenado:** no se detectó uso de `{!! !!}` en las vistas principales; Blade escapado reduce el riesgo.
- **Déficit principal frontend:** falta de política CSP y cabeceras de seguridad (defensa en profundidad), más que XSS directo en templates actuales.

---

## Recomendaciones de Mitigación (Accionables)

### 1. Autorizar ruta de estado de pedidos por rol
- Mover `pedidos.estado` a `role:admin,empleado` o usar `Policy`/`Gate`.

### 2. Aplicar autorización por objeto (ownership)
- En `PedidoController@index/show`:
  - **cliente:** sólo sus pedidos (`where('id_usuario', auth()->id())`)
  - **admin/empleado:** todos.
- Ideal: `PedidoPolicy@view`.

### 3. Bloquear login de usuarios inactivos
- Cambiar `Auth::attempt(...)` a credenciales con `estado => 'activo'`.

### 4. Rate limiting de operaciones sensibles
- Aplicar `throttle` en rutas de creación, cambio de estado y confirmación de pago.

### 5. Hardening de seguridad web
- En producción: `APP_DEBUG=false`, HTTPS, `SESSION_SECURE_COOKIE=true`.
- Agregar middleware de security headers: **CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy**.
- Mover JS inline del dashboard a archivo JS externo para permitir CSP estricta.

---

> **Nota:** El análisis fue estructurado tomando como referencia los criterios de revisión de seguridad backend/frontend (autenticación, autorización, validación, sesiones, cabeceras y superficie de ataque).
