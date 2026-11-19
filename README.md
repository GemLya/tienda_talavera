# Talavera Dolores — tienda móvil con TDD (`tienda_talavera`)

App Flutter de comercio electrónico para vender artesanías de **Dolores Hidalgo, Gto.**: catálogo, carrito, pago (simulado), perfil de usuario y **"Ver en tu espacio"** con realidad aumentada (RA).

Proyecto de la materia **Desarrollo Móvil Integral** — Universidad Tecnológica del Norte de Guanajuato (UTNG). Construido siguiendo el *Manual paso a paso con TDD — Talavera Dolores: tienda móvil completa* del Prof. Anastacio Rodríguez García.

> ⚠️ **Importante:** la pasarela de pagos y el servidor de cuentas son **SIMULADOS**. La app **nunca procesa dinero real** ni envía datos de tarjeta a ningún lado.

---

## 1. ¿Qué hace la app?

| Pieza | Qué hace |
|---|---|
| **Catálogo** | 8 productos (talavera, dulces, textiles, decoración). Búsqueda, filtro por categoría y filtro "Solo AR". |
| **Detalle** | Precio, medidas, existencias, favorito y botón "Ver en tu espacio (AR)" si el producto tiene modelo 3D. |
| **Carrito** | Máximo 10 piezas por producto, nunca más que las existencias, cupones `DOLORES10` (10 %) y `TALAVERA15` (15 %), envío de **$99.00** gratis desde **$800.00**, IVA incluido desglosado. Se guarda en el teléfono. |
| **Checkout** | 3 pasos: dirección → pago → confirmación. Revalida precios y existencias **antes** de cobrar. Usa clave de idempotencia para no cobrar dos veces. |
| **Perfil** | Registro / inicio de sesión, editar dirección, favoritos, "Mis compras" y cerrar sesión. |
| **Ver en tu espacio** | Visor 3D con `model_viewer_plus` y botón de AR (Scene Viewer en Android, AR Quick Look en iPhone) a escala real (`ArScale.fixed`). |

---

## 2. Estructura de carpetas

```
tienda_talavera/
  lib/
    dominio/                 <- reglas del negocio (Dart puro, SIN Flutter)
      dinero.dart              centavos, IVA incluido, porcentaje con redondeo
      producto.dart            Producto, Categoria, Dimensiones, tieneModelo3d
      carrito.dart             Carrito, LineaCarrito, Cupon
      excepciones.dart         ErrorDeApp + mensajeSeguro()
      validadores.dart         correo, contraseña, CP, teléfono...
      direccion.dart
      usuario.dart
      orden.dart
      tarjeta.dart             Luhn, marca, vigencia, CVV, toString seguro
      bloqueo_intentos.dart    5 fallos -> 15 min bloqueado
      contratos.dart           interfaces (repositorios, pasarela, auth)
      servicio_checkout.dart   orquesta el pago
    datos/                   <- implementaciones de los contratos
      catalogo_memoria.dart    catálogo inicial de 8 productos
      almacenes.dart           carrito (memoria / shared_preferences),
                               almacén seguro (memoria / flutter_secure_storage),
                               órdenes en memoria
      pasarela_simulada.dart   tarjetas de prueba + idempotencia
      auth_simulada.dart       sal + SHA-256, token Random.secure(), bloqueo
    presentacion/
      vm/                    <- ViewModels (ChangeNotifier)
        catalogo_vm.dart  carrito_vm.dart  sesion_vm.dart  checkout_vm.dart
      pantallas/             <- widgets "tontos": solo dibujan
        pantalla_catalogo.dart  pantalla_detalle.dart  pantalla_carrito.dart
        pantalla_checkout.dart  pantalla_perfil.dart
        pantalla_ver_en_tu_espacio.dart
    dependencias.dart        <- raíz de composición (único lugar que elige implementaciones)
    main.dart
  test/
    dominio_test.dart        dinero, producto, carrito, validadores, tarjeta
    servicios_test.dart      pasarela, checkout, bloqueo, auth
    viewmodels_test.dart     los 4 ViewModels
```

### Regla de dependencia de capas

```
presentacion  ──►  dominio  ◄──  datos
        (y dependencias.dart conecta todo)
```

- Las flechas apuntan **siempre hacia el dominio**.
- `lib/dominio/` **no importa** `package:flutter/...` ni paquetes de terceros: solo conoce *contratos* (clases abstractas).
- `datos/` implementa esos contratos (memoria, `shared_preferences`, `flutter_secure_storage`).
- Cambiar la pasarela simulada por una real (Stripe, Mercado Pago, Conekta) se hace tocando **solo** `lib/dependencias.dart`.

Puedes comprobar que el dominio está limpio con:

```bash
grep -rn "package:flutter" lib/dominio/    # no debe imprimir nada
```

---

## 3. Requisitos

- Flutter 3.10 o superior (Dart 3). Comprueba con `flutter --version` y `flutter doctor`.
- Android Studio con un emulador o un teléfono Android con depuración USB.
- Para probar la **RA real**: un teléfono físico compatible con ARCore.

---

## 4. Instalación paso a paso

Este paquete trae `lib/`, `test/`, `pubspec.yaml`, `analysis_options.yaml`, el `AndroidManifest.xml` y este README. Las carpetas de plataforma (`android/`, `ios/`, etc.) las genera Flutter:

```bash
# 1) Crear el proyecto base (genera android/, ios/, web/...)
flutter create tienda_talavera
cd tienda_talavera

# 2) Copiar ENCIMA los archivos de este paquete (reemplazar cuando pregunte):
#    lib/  test/  pubspec.yaml  analysis_options.yaml  README.md
#    android/app/src/main/AndroidManifest.xml

# 3) Borrar la prueba de ejemplo de "flutter create" (busca MyApp, que ya no existe)
#    Windows (PowerShell):  Remove-Item test\widget_test.dart
#    macOS / Linux:         rm test/widget_test.dart

# 4) Descargar dependencias
flutter pub get
```

Si `flutter pub get` se queja de alguna versión, borra esa línea en `pubspec.yaml` y ejecuta:

```bash
flutter pub add model_viewer_plus shared_preferences flutter_secure_storage crypto
```

---

## 5. Correr las pruebas (TDD)

```bash
flutter test                              # todas
flutter test test/dominio_test.dart       # solo dominio
flutter test test/servicios_test.dart     # pasarela, checkout, bloqueo y auth
flutter test test/viewmodels_test.dart    # ViewModels
```

Deben salir en verde: **`All tests passed!`**

Qué verifican, en resumen:

- **Dinero:** `$1,250.00`, IVA incluido de `$116.00` = `$16.00`, `porcentaje(9999, 10) == 1000` (redondeo `(c * pct + 50) ~/ 100`).
- **Carrito:** límite de 10, existencias, envío gratis desde `80000` centavos, cupones insensibles a mayúsculas/espacios, un error deja el carrito **intacto**, `aMapas()` guarda solo `id` y `cantidad`.
- **Tarjeta:** Luhn, marcas (Visa, Mastercard, Amex), vigencia hasta el último día del mes, CVV de 4 dígitos solo en Amex, `toString()` sin el número completo.
- **Checkout:** no cobra si cambió el precio o la tarjeta es inválida; si se rechaza el pago no crea orden ni toca inventario.
- **Pasarela:** misma clave de idempotencia = un solo cobro.
- **Auth:** mensaje genérico, bloqueo tras 5 fallos (aun con la clave correcta), token guardado en almacén seguro, no se puede cambiar correo/id.
- **ViewModels:** reusa la clave tras error de red, la renueva tras rechazo, ignora el doble toque.

Análisis estático (debe decir `No issues found!`):

```bash
flutter analyze
```

---

## 6. Ejecutar la app

```bash
flutter devices          # lista emuladores/teléfonos conectados
flutter run              # en modo debug
```

Recorrido sugerido: Catálogo → Detalle → Agregar al carrito → Perfil (crear cuenta) → Carrito → Ir a pagar → Dirección → Pago → Confirmación → Perfil › "Mis compras".

### Tarjetas de prueba (pasarela simulada)

| Número | Resultado |
|---|---|
| `4242 4242 4242 4242` | ✅ Aprobada |
| `4000 0000 0000 0002` | ❌ Rechazada por el banco (no descuenta inventario) |
| `4000 0000 0000 9995` | ❌ Fondos insuficientes |

Usa cualquier fecha futura (por ejemplo mes `12`, año `30`) y CVV `123` (Amex: 4 dígitos).
Montos mayores a $50,000.00 se rechazan. **No se procesa dinero real.**

### Cupones

`DOLORES10` (10 %) y `TALAVERA15` (15 %). Puedes escribirlos en minúsculas o con espacios.

### Nota sobre la sesión simulada

El token de sesión sí se guarda en `flutter_secure_storage`, pero el "servidor" de cuentas (`AuthSimulada`) y las órdenes viven en memoria. Al cerrar por completo la app, las cuentas se borran y el token guardado se descarta como inválido (es justo lo que prueba "token inválido en el almacén se descarta"). El **carrito** sí se conserva porque va en `shared_preferences`. Con un backend real la sesión también se conservaría.

---

## 7. Realidad aumentada: permisos y notas

### Android

En `android/app/src/main/AndroidManifest.xml` (ya incluido en este paquete):

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

- `android:usesCleartextTraffic="true"` **solo** si pruebas modelos desde un servidor local por `http`. En producción usa `https`.
- El modo AR usa **Scene Viewer**, que requiere un teléfono **compatible con ARCore** y la app **"Google Play Services for AR"** instalada/actualizada desde Play Store.
- `minSdkVersion`: el proyecto usa `flutter.minSdkVersion` (21 o más). Si algún paquete pide más, edita `android/app/build.gradle` (o `build.gradle.kts`) y pon `minSdk = 21` (o el que indique el error).
- En el **emulador** el visor 3D sí se ve, pero el botón de AR no aparece o no abre la cámara.

### iOS

- AR Quick Look necesita el modelo en formato **`.usdz`** (campo `modeloUsdz`, que se pasa como `iosSrc`). Se convierte desde `.glb` con *Reality Converter* de Apple.
- No requiere permiso de cámara en `Info.plist` para el visor. Si luego usas la cámara directamente, agrega `NSCameraUsageDescription`.
- El simulador de iOS **no** soporta AR: prueba en un **iPhone físico**.

### Modelos 3D

- Solo se aceptan rutas `https://` o `assets/` (`Producto.tieneModelo3d`). Nunca `http://` ni `file://`.
- Los modelos del catálogo (astronauta y caballo de `modelviewer.dev`) son **solo de práctica**. Para publicar usa modelos propios o con licencia comercial.
- Constrúyelos a escala real en metros (un jarrón de 30 cm mide 0.30 en Y) y procura que pesen menos de 5 MB.

---

## 8. Generar el ejecutable

### APK de release (para instalar directo en un teléfono Android)

```bash
flutter clean
flutter pub get
flutter analyze
flutter test
flutter build apk --release
```

El archivo queda en:

```
build/app/outputs/flutter-apk/app-release.apk
```

Para instalarlo en un teléfono conectado por USB:

```bash
flutter install
# o bien:
adb install build/app/outputs/flutter-apk/app-release.apk
```

APK más ligeros (uno por arquitectura de procesador):

```bash
flutter build apk --release --split-per-abi
# genera app-armeabi-v7a-release.apk, app-arm64-v8a-release.apk, app-x86_64-release.apk
```

### App Bundle (para subir a Google Play)

```bash
flutter build appbundle --release
```

Queda en:

```
build/app/outputs/bundle/release/app-release.aab
```

> **Firma:** el proyecto que crea `flutter create` firma el release con la llave de *debug*. Sirve para entregar la práctica e instalar el APK, pero **Google Play exige una llave propia**. Ver <https://docs.flutter.dev/deployment/android#signing-the-app>.

---

## 9. Seguridad aplicada (resumen)

| Dato | Dónde se guarda | ¿Por qué? |
|---|---|---|
| Carrito (id, cantidad) | `shared_preferences` | No es secreto. El precio se vuelve a leer del catálogo. |
| Token de sesión | `flutter_secure_storage` (Keychain / Keystore) | Es una llave. |
| Contraseña | En ningún lado del teléfono | Se guarda sal + hash en el "servidor". |
| Tarjeta completa / CVV | En ningún lado | PCI DSS. Se limpian del formulario al pagar. |
| Últimos 4 dígitos | En la orden | Referencia segura. |

Otras defensas: mensajes de error genéricos (`mensajeSeguro`), cálculo de hash aunque el correo no exista, comparación de hash en tiempo constante, bloqueo por intentos, revalidación de precios antes de cobrar, idempotencia y bloqueo del doble toque.

> SHA-256 se usa porque es didáctico. Un backend real debe usar **bcrypt, scrypt o Argon2**.

---

## 10. Correcciones respecto al manual

Para que compile limpio y sin advertencias se hicieron estos ajustes (la lógica y las pruebas son las mismas):

1. **Emojis de productos:** en el PDF salían en blanco; se asignaron emojis reales (🏺 🍽️ ☕ 🐴 🍯 🍬 🧣 🧵).
2. **Líneas partidas del PDF:** cadenas y comentarios cortados a la mitad (por ejemplo `'pedido-` / `${...}'` en `checkout_vm.dart`) se unieron. La clave por defecto ahora la genera un método estático `_claveAleatoria()`.
3. **Funciones `async =>` que devolvían un valor en métodos `Future<void>`** (almacenes en memoria): ahora usan cuerpo con llaves.
4. **`builder: (_, __)`** se cambió por `(context, _)` para evitar la advertencia `unnecessary_underscores` en versiones nuevas de Dart.
5. **Diálogo de dirección del perfil:** ahora es un `StatefulWidget` (`_DialogoDireccion`) que libera sus controladores en `dispose()`. En el manual se liberaban justo al cerrar, mientras la animación aún los usaba (error *"TextEditingController was used after being disposed"*).
6. **`AlmacenCarritoPrefs.cargar()`** revisa cada elemento guardado; si el archivo está dañado devuelve `[]` en vez de tronar al restaurar. `Carrito.restaurar` también ignora entradas mal formadas.
7. **`AuthSimulada`:** se agregó comparación de hash en tiempo constante y se quitó un `!` innecesario en `actualizarPerfil`.
8. **`CatalogoMemoria.cambiarPrecio`** ya no truena si el id no existe.
9. **`test/widget_test.dart`** de `flutter create` se debe borrar (usa `MyApp`, que no existe).
10. Se agregó el texto "Disponible en teléfonos compatibles con AR" en el detalle y la lista de favoritos por nombre en el perfil, como sugiere el manual.

---

## 11. Comandos finales (copiar y pegar)

```bash
flutter pub get
flutter analyze
flutter test
flutter build apk --release
# -> build/app/outputs/flutter-apk/app-release.apk
flutter build appbundle --release
# -> build/app/outputs/bundle/release/app-release.aab
```
