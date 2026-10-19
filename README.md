# MIDNIGHT DINER — reservas de restaurantes

Aplicación web para reservar mesa en restaurantes, con dos roles de uso (**cliente** y **administrador**), construida para el **Parcial CR-2 — Aplicación de reservas con patrones creacionales** del curso *Patrones de Diseño de Software* (Universidad Sergio Arboleda, G02).

Stack: **FastAPI + Jinja2 + SQLite** (`sqlite3`, sin ORM), organizada en **tres capas** (presentación → negocio → datos) con **DAO** y **6 patrones de diseño** (3 creacionales y 3 estructurales). «MIDNIGHT DINER» es una marca ficticia (`BRAND_NAME` en `app/config.py`).

| | |
|---|---|
| **Aplicación en línea** | https://midnight-diner.onrender.com/ |
| **Video explicativo** | Enlace de Google Drive indicado en la entrega de la plataforma |
| **Documento (PDF, APA)** | `parcial_cr2_reservas.pdf` (se entrega por la plataforma) |

> La aplicación está en el plan gratuito de Render: tras 15 min sin visitas se «duerme» y la primera carga tarda cerca de 1 minuto. Además el disco es efímero: al reiniciarse, la base se recrea con los datos de demostración (ver [Despliegue](#despliegue)).

## Contenido

1. [Cómo ejecutarlo](#cómo-ejecutarlo)
2. [Usuarios de demostración](#usuarios-de-demostración)
3. [Roles y administradores](#roles-y-administradores)
4. [Qué puede hacer cada rol](#qué-puede-hacer-cada-rol)
5. [Arquitectura en tres capas](#arquitectura-en-tres-capas)
6. [Patrones de diseño](#patrones-de-diseño)
7. [Modelo de datos](#modelo-de-datos)
8. [Reglas de negocio](#reglas-de-negocio)
9. [Seguridad](#seguridad)
10. [Pruebas](#pruebas)
11. [Despliegue](#despliegue)
12. [Cumplimiento de la rúbrica del parcial](#cumplimiento-de-la-rúbrica-del-parcial)
13. [Atribución de las cartas](#atribución-de-las-cartas) y [Créditos](#créditos)

---

## Cómo ejecutarlo

Probado con Python 3.13 (el mismo que usa Render).

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m app.datos.seed             # crea reservas.db con los datos de demostración
uvicorn app.main:app --reload        # http://127.0.0.1:8000
```

Variables de entorno (todas opcionales en desarrollo):

| Variable | Para qué sirve |
|---|---|
| `SECRET_KEY` | Firma de la cookie de sesión. El valor por defecto es solo para desarrollo. |
| `RESERVAS_DB` | Ruta del archivo SQLite (por defecto `reservas.db` junto a la carpeta `app/`). |
| `HTTPS_ONLY=1` | Cookie de sesión `Secure` y cabecera HSTS (producción). |
| `AUTOSEMBRAR=1` | Siembra los datos de demostración al arrancar si la base está vacía (Render). |

## Usuarios de demostración

Los crea `python -m app.datos.seed` (datos ficticios). Cada administrador gestiona **un** restaurante.

| Rol | Correo | Contraseña | Restaurante |
|---|---|---|---|
| Cliente | `cliente@demo.com` | `Cliente123!` | — |
| Administrador | `admin@demo.com` | `Admin123!` | 1 · Slice Palace Medianoche (Bogotá, pizzería) |
| Administrador | `admin2@demo.com` | `Admin123!` | 2 · La Cocina del Turno Noche (Bogotá, colombiana) |
| Administrador | `admin3@demo.com` | `Admin123!` | 3 · Pasillo Este Sushi (Cali, japonesa) |
| Administrador | `admin4@demo.com` | `Admin123!` | 4 · Cámara 04 Parrilla (Medellín, parrilla) |
| Administrador | `admin5@demo.com` | `Admin123!` | 5 · Backstage Pasta & Vino (Cali, italiana) |

## Roles y administradores

Hay tres situaciones de acceso: **visitante** (sin sesión), **cliente** y **administrador**.

### Cómo entra un administrador

1. Abrir `/login` (o el botón «Ingresar» de la barra superior).
2. Escribir el correo y la contraseña de un administrador (por ejemplo `admin@demo.com` / `Admin123!`).
3. El sistema lo redirige solo a **`/admin`**, el panel de su restaurante (un cliente, en cambio, va a `/restaurantes`).

El mismo formulario de `/login` sirve para clientes y administradores; lo que cambia es el **rol guardado en la base**, y el rol decide adónde llega la persona y qué rutas puede abrir.

### Cómo se asigna el rol de administrador (decisión de seguridad)

`/registro` es público y **siempre crea un cliente**: el rol no viaja en el formulario, lo fija el servidor. Es una decisión de seguridad deliberada: si cualquiera pudiera elegir ser administrador, podría adueñarse del panel de un restaurante, ver sus reservas y cambiar su carta. Los administradores solo se crean de dos formas: los **5 de la siembra** (`app/datos/seed.py`) o con el **script de alta** de más abajo. Al iniciar sesión, la fábrica de usuarios (Factory Method) construye un `Administrador`, y es ese objeto el que decide el destino con `ruta_inicio()` → `/admin`.

### Alcance del rol

- **Un administrador ↔ un restaurante** (`restaurante.admin_id` es `NOT NULL`): solo ve y modifica lo suyo. Si cambia un identificador en la URL para mirar otro restaurante, el Proxy de protección lo rechaza y deja la línea `DENEGADO` en el registro.
- **Rol de gestión**: recibe, confirma y cancela las reservas de los clientes. El formulario de reserva (`/reservar`) es del cliente; el administrador que lo abre recibe 403 «Esta sección es solo para clientes».
- Un **cliente** que intente abrir `/admin` recibe 403 «ACCESO DENEGADO».

### Cómo crear más administradores

El alta de administradores es una operación **controlada por el operador del sistema**, no por el público: se ejecuta en el servidor con un script de apoyo que usa las mismas funciones de la aplicación (misma conexión, mismo hash `scrypt`) y crea el usuario junto con su restaurante. Guárdalo como `nuevo_admin.py` (fuera del repositorio o bórralo al terminar), cambia los datos de las primeras líneas y ejecútalo **desde la carpeta del proyecto**:

```python
from app.datos.conexion import ConexionBD
from app.seguridad import hashear_password

NOMBRE, EMAIL, PASSWORD = "Ana Pérez", "ana@resto.com", "Clave2026"
RESTAURANTE = dict(nombre="La Esquina", direccion="Cra 5 #10-20", ciudad="Bogotá",
                   tipo_cocina="Italiana", descripcion="")

ConexionBD().inicializar_esquema()
with ConexionBD().sesion() as con:        # una sola transacción: se crean los dos registros o ninguno
    uid = con.execute(
        "INSERT INTO usuario (nombre, email, password_hash, rol) VALUES (?,?,?, 'administrador')",
        (NOMBRE, EMAIL.lower(), hashear_password(PASSWORD))).lastrowid
    con.execute(
        "INSERT INTO restaurante (nombre, direccion, ciudad, tipo_cocina, descripcion, admin_id) "
        "VALUES (?,?,?,?,?,?)", (*RESTAURANTE.values(), uid))
print("Administrador creado:", EMAIL)
```

```bash
python nuevo_admin.py
```

Después, la persona entra por `/login` con ese correo y contraseña y **configura su restaurante desde el panel**: mesas, horarios y carta. Notas:

- Usa un correo único (en minúsculas) y una contraseña de al menos 8 caracteres con letras y números, la misma regla del registro.
- Un restaurante nuevo empieza sin mesas ni horarios: el administrador los carga desde su panel y desde ese momento el restaurante queda abierto a reservas.
- El script apunta a la base indicada por `RESERVAS_DB` (o `reservas.db` por defecto).
- En la demostración en línea (Render, plan gratuito) los administradores son los 5 de la siembra, que se recrean en cada arranque. Para dejar uno adicional de forma permanente en un despliegue, se agrega a `app/datos/seed.py`.

## Qué puede hacer cada rol

| Rol | Puede | No puede |
|---|---|---|
| **Visitante** | Ver inicio, lista de restaurantes y cartas; registrarse (`/registro`) o ingresar (`/login`). | Reservar ni ver reservas. |
| **Cliente** | Elegir restaurante, fecha, hora y mesa en el asistente de reserva (`/reservar`); añadir platos al pre-pedido y ver el total a pagar; ver, filtrar y cancelar **sus** reservas; imprimir el comprobante y descargar el `.ics` para el calendario. | Entrar a `/admin` (403) ni ver reservas de otras personas. |
| **Administrador** | Ver indicadores y gráficos de **su** restaurante (ocupación, reservas por estado, mapa de calor, top de platos); **gestionar reservas** (buscar, filtrar, confirmar, cancelar con confirmación, exportar CSV, paginar); **gestionar mesas** (crear, editar, activar, eliminar); **gestionar horarios** semanales; **gestionar la carta** (categorías y platos, imagen JPG/PNG/WebP ≤ 2 MB, marcar agotado). | Usar `/reservar` (403) ni tocar datos de otro restaurante. |

## Arquitectura en tres capas

```
app/
├── presentacion/   CONTROLADORES (auth, cliente, admin, api) + VISTAS (templates Jinja2, static)
├── negocio/        servicios · patrones/creacionales · patrones/estructurales · excepciones
├── datos/          MODELOS (dataclasses) · dao/ · conexion.py (Singleton) · schema.sql · seed.py
├── dependencias.py raíz de composición: conecta DAO → servicios → proxy → fachada
└── seguridad.py    hash de contraseñas (scrypt, librería estándar)
```

- **Presentación (MVC: vista + controlador)**: los controladores solo reciben la petición, llaman a `ReservaFacade` y renderizan una plantilla Jinja2. Nunca hablan con el servicio real ni con SQL.
- **Negocio**: reglas, servicios y patrones. La fachada recibe `ReservaServiceProxy`, nunca el servicio real.
- **Datos (modelo)**: dataclasses + `BaseDAO[T]` con un DAO por entidad. Un DAO no importa negocio: `UsuarioDAO` recibe la fábrica de usuarios por inyección.

Los diagramas (arquitectura, modelo relacional, DAO y control con patrones) están en `docs/*.mmd` y como PNG en `docs/diagramas/`.

## Patrones de diseño

| Tipo | Patrón | Clase(s) | Archivo | Para qué se usa |
|---|---|---|---|---|
| Creacional | **Singleton** | `ConexionBD` | `app/datos/conexion.py` | Una sola conexión/configuración SQLite (WAL, `busy_timeout`), con doble verificación y candado. |
| Creacional | **Factory Method** | `FabricaUsuario`, `FabricaCliente`, `FabricaAdministrador`, `UsuarioFactory` | `app/negocio/patrones/creacionales/usuario_factory.py` | Crea el tipo de usuario según el rol sin `if` dispersos. |
| Creacional | **Builder** | `ReservaBuilder` (+ `ReservaDirector`) | `app/negocio/patrones/creacionales/reserva_builder.py` | Arma paso a paso una reserva válida (mesa, fecha, personas, pre-pedido). |
| Estructural | **Facade** | `ReservaFacade` | `app/negocio/patrones/estructurales/reserva_facade.py` | Interfaz única para los controladores sobre 7 servicios. |
| Estructural | **Proxy** (protección) | `ReservaServiceProxy` | `app/negocio/patrones/estructurales/reserva_proxy.py` | Verifica rol y propiedad del recurso antes de llegar al servicio real; registra `PERMITIDO`/`DENEGADO`. |
| Estructural | **Decorator** | `Cobro`, `CobroBase`, `CobroDecorador`, `ConDescuento`, `ConImpuestoConsumo`, `ConPropinaSugerida` | `app/negocio/patrones/estructurales/cobro_decorator.py` | Total a pagar: descuento → impuesto → propina opcional. |

**Total a pagar (Decorator).** Todo en pesos colombianos enteros, con una única regla de redondeo (al peso más cercano, empates hacia arriba). Orden fijo: descuento 10 % desde $150.000 → impuesto al consumo 8 % → propina sugerida 10 % (opcional). Ejemplo: subtotal $170.000 → descuento −$17.000 → impuesto +$12.240 → **$165.240**; con propina +$16.524 → **$181.764**. Las tasas son **supuestos de demostración** (`app/config.py`) y no se verificaron contra la normativa vigente. La reserva guarda una instantánea del cobro (`subtotal_cop`, `descuento_*`, `impuesto_*`, `total_pagar_cop`) para que un cambio posterior de las tasas no altere lo ya cobrado.

## Modelo de datos

Ocho tablas (`app/datos/schema.sql`): `usuario`, `restaurante` (con `admin_id`), `mesa`, `horario`, `reserva`, `categoria`, `producto` y `reserva_item`. Una reserva pertenece a un cliente y a una mesa; sus ítems congelan el precio unitario del momento. Diagrama: `docs/modelo_relacional.mmd`.

## Reglas de negocio

1. Fecha y hora futuras y dentro del horario del restaurante; la reserva dura **120 min**, así que la última hora válida es `cierre − 120 min`.
2. La mesa debe estar activa y su capacidad ≥ número de personas.
3. **Sin solapamientos** en una mesa (las canceladas no cuentan). La comprobación y la inserción ocurren en una sola transacción `BEGIN IMMEDIATE`, de modo que dos personas que eligen la misma mesa y hora a la vez no pueden quedarse ambas con ella: la segunda recibe un aviso de que ya está ocupada.
4. El cliente solo ve y cancela lo suyo; el administrador solo gestiona su restaurante.
5. Estados: `PENDIENTE → CONFIRMADA | CANCELADA` (además `CONFIRMADA → CANCELADA`).
6. El precio de cada plato sale siempre de la base de datos, nunca del navegador.

## Seguridad

Contraseñas con **scrypt** (sal propia); **token CSRF** por sesión en todos los POST; cookie de sesión firmada (8 h, `SameSite=Lax`, `Secure` con `HTTPS_ONLY`); **límite de intentos de login** (5 fallos por IP y correo, bloqueo de 5 min, respuesta 429); cabeceras de seguridad y CSP; subida de imágenes validada por su cabecera real (JPG/PNG/WebP, ≤ 2 MB, nombre aleatorio); consultas SQL siempre parametrizadas; control de acceso por rol en los controladores y por propiedad del recurso en el Proxy.

## Pruebas

```bash
pytest -q          # 154 pruebas, todas pasan
```

Cubren cada patrón (`tests/test_patrones.py`, `tests/test_fase4.py` para el Decorator), las reglas de reserva (`tests/test_reglas_negocio.py`), la concurrencia y la seguridad (`tests/test_fase5.py`), el cobro guardado, las cartas y la capa web de punta a punta (login, reserva, doble reserva, confirmación y permisos en `tests/test_web.py`). Capturas de la interfaz: `docs/capturas/` (se regeneran con `python tools/e2e_capturas.py http://localhost:8765 docs/capturas`, ver `docs/ESTADO.md`).

## Despliegue

Render, plan gratuito, definido en `render.yaml` (Blueprint): instala `requirements-prod.txt`, arranca `uvicorn` con `--proxy-headers` y se actualiza solo con cada `push` a la rama `main`. La base vive en `/tmp/reservas.db` y se siembra al arrancar (`AUTOSEMBRAR=1`). Detalles, límites del plan gratuito y por qué no se usa Vercel: `docs/DESPLIEGUE.md`.

## Cumplimiento de la rúbrica del parcial

| Requisito del parcial | Dónde está |
|---|---|
| Aplicación web de reservas con login y formulario de reserva para **cliente y administrador** | Login común en `/login` (cliente → `/restaurantes`, administrador → `/admin`). El cliente reserva en `/reservar`; el administrador recibe, confirma y cancela esas reservas en `/admin/reservas`. |
| Los restaurantes gestionan **mesas, horarios y reservas** | Panel de administración: `/admin/mesas`, `/admin/horarios`, `/admin/reservas` (más la carta en `/admin/carta`). |
| **≥ 2 patrones creacionales** | Singleton, Factory Method y Builder (3). |
| **≥ 2 patrones estructurales** | Facade, Proxy y Decorator (3). |
| Diseño orientado a objetos en **3 capas / MVC** | Ver [Arquitectura](#arquitectura-en-tres-capas). |
| Modelo relacional | `docs/modelo_relacional.mmd` y `app/datos/schema.sql`. |
| Diagrama de clases **DAO** y de **control con patrones** | `docs/diagrama_dao.mmd` y `docs/diagrama_control_patrones.mmd` (PNG en `docs/diagramas/`). |
| Documento en **APA** con análisis, diseño, código y capturas de ejecución | `parcial_cr2_reservas.pdf` (se entrega aparte). |
| Código fuente | Este repositorio. |
| Aplicación desplegada | https://midnight-diner.onrender.com/ |
| Video (código, capas/MVC, diagrama con patrones creacionales, aplicación en ejecución) | Enlace de Google Drive indicado en la entrega de la plataforma. |

---

## Atribución de las cartas

Las cartas de los restaurantes 1 y 5 parten del menú de un repositorio de GitHub de un tercero, usado con su autorización; solo se tomaron nombres, descripciones y precios (sin imágenes ni diseño). Las cartas 2, 3 y 4 son propias.

## Créditos

- Tipografías: Google Fonts (VT323, Press Start 2P, Bebas Neue, Inter; licencia OFL). Ilustraciones: SVG/CSS originales.
- Marca ficticia, sin material de terceros salvo las fotos de platos, con sus créditos.
- Fotos de los platos: descargadas de **Wikimedia Commons** con `tools/descargar_fotos.py` (licencias CC0 / dominio público / CC BY / CC BY-SA, verificadas por la API). Recortadas a 16:10 centrando el plato, con auto-contraste y nitidez suaves, a 800×500 (< 250 KB) más una versión `@400` para `srcset`. Al reutilizarlas se debe mantener esta atribución; las CC BY-SA obligan a compartir las adaptaciones con la misma licencia.

| Plato (archivo) | Foto en Wikimedia Commons | Autor | Licencia |
|---|---|---|---|
| `bruschetta-clasica.jpg` | [Bruschetta at The Bar at MacArthur Place - Sarah Stierch.jpg](https://commons.wikimedia.org/wiki/File:Bruschetta_at_The_Bar_at_MacArthur_Place_-_Sarah_Stierch.jpg) | Missvain | CC BY 4.0 |
| `ensalada-caprese.jpg` | [Caprese salad in homemade.jpg](https://commons.wikimedia.org/wiki/File:Caprese_salad_in_homemade.jpg) | spiralmushroom | CC BY 2.0 |
| `carpaccio-di-manzo.jpg` | [Beef Carpaccio (3466985953).jpg](https://commons.wikimedia.org/wiki/File:Beef_Carpaccio_(3466985953).jpg) | Mack Male from Edmonton, AB, Canada | CC BY-SA 2.0 |
| `calamares-fritti.jpg` | [Calamari fritti in Izola Isola (52976884396).jpg](https://commons.wikimedia.org/wiki/File:Calamari_fritti_in_Izola_Isola_(52976884396).jpg) | domjisch from Graz, Austria | CC BY 2.0 |
| `arancini-di-risotto.jpg` | [Arancini 002.jpg](https://commons.wikimedia.org/wiki/File:Arancini_002.jpg) | No machine-readable author provided. Gmelfi assumed (based on copyright claims). | Public domain |
| `focaccia-al-rosmarino.jpg` | [Rosemary Focaccia.jpg](https://commons.wikimedia.org/wiki/File:Rosemary_Focaccia.jpg) | Parkerman &amp; Christie from San Diego, USA | CC BY 2.0 |
| `zuppa-minestrone.jpg` | [Minestrone soup.jpg](https://commons.wikimedia.org/wiki/File:Minestrone_soup.jpg) | Katrin Morenz from Aachen, Deutschland | CC BY-SA 2.0 |
| `burrata-con-tomates-cherry.jpg` | [Burrata salad with cherry tomatoes and rocket from La Mensa Grill and Bar Plaza88.jpg](https://commons.wikimedia.org/wiki/File:Burrata_salad_with_cherry_tomatoes_and_rocket_from_La_Mensa_Grill_and_Bar_Plaza88.jpg) | Peachyeung316 | CC BY-SA 4.0 |
| `antipasto-misto.jpg` | [Antipasto Misto Con Bruschette.jpg](https://commons.wikimedia.org/wiki/File:Antipasto_Misto_Con_Bruschette.jpg) | AlMare (usuario de Commons) | CC BY 2.0 |
| `pizza-margherita.jpg` | [Margherita Originale.JPG](https://commons.wikimedia.org/wiki/File:Margherita_Originale.JPG) | Mario56 | CC BY-SA 3.0 |
| `pizza-pepperoni.jpg` | [Pepperoni Pizza from Fellini’s Pizza.jpg](https://commons.wikimedia.org/wiki/File:Pepperoni_Pizza_from_Fellini%E2%80%99s_Pizza.jpg) | BullDawg2021 | CC BY 4.0 |
| `pizza-quattro-formaggi.jpg` | [Four Cheese - Pizza 500 2023-11-10.jpg](https://commons.wikimedia.org/wiki/File:Four_Cheese_-_Pizza_500_2023-11-10.jpg) | Andy Li | CC0 |
| `spaghetti-alla-carbonara.jpg` | [Espaguetis carbonara.jpg](https://commons.wikimedia.org/wiki/File:Espaguetis_carbonara.jpg) | Javier Somoza | CC BY-SA 4.0 |
| `fettuccine-alfredo.jpg` | [Fettuccine Alfredo.jpg](https://commons.wikimedia.org/wiki/File:Fettuccine_Alfredo.jpg) | Robert Banh | CC BY 2.0 |
| `lasagna-alla-bolognese.jpg` | [Meaty Lasagna 8of8 (8736299782).jpg](https://commons.wikimedia.org/wiki/File:Meaty_Lasagna_8of8_(8736299782).jpg) | Breville USA | CC BY 2.0 |
| `gnocchi-al-pesto.jpg` | [Gnocchi with pesto (3269919059).jpg](https://commons.wikimedia.org/wiki/File:Gnocchi_with_pesto_(3269919059).jpg) | Karen and Brad Emerson | CC BY 2.0 |
| `risotto-ai-funghi.jpg` | [Risotto Champignons.jpg](https://commons.wikimedia.org/wiki/File:Risotto_Champignons.jpg) | PixelBrûlé | CC BY-SA 4.0 |
| `pollo-alla-parmigiana.jpg` | [Tony's of North Beach - December 2022 - Sarah Stierch 05.jpg](https://commons.wikimedia.org/wiki/File:Tony%27s_of_North_Beach_-_December_2022_-_Sarah_Stierch_05.jpg) | Missvain | CC BY 4.0 |
| `tiramisu.jpg` | [Tiramisu - Bistro Don Giovanni - Sarah Stierch 2018 02.jpg](https://commons.wikimedia.org/wiki/File:Tiramisu_-_Bistro_Don_Giovanni_-_Sarah_Stierch_2018_02.jpg) | Missvain | CC BY 4.0 |
| `panna-cotta.jpg` | [Panna Cotta with cream and garnish.jpg](https://commons.wikimedia.org/wiki/File:Panna_Cotta_with_cream_and_garnish.jpg) | hiro | CC BY-SA 2.0 |
| `cannoli-siciliani.jpg` | [Tiny sicilian cannoli at lunch (5071817409).jpg](https://commons.wikimedia.org/wiki/File:Tiny_sicilian_cannoli_at_lunch_(5071817409).jpg) | judywitts | CC BY 2.0 |
| `affogato-al-caffe.jpg` | [Vinoteca, Smithfield, London (4485849609).jpg](https://commons.wikimedia.org/wiki/File:Vinoteca,_Smithfield,_London_(4485849609).jpg) | Ewan Munro from London, UK | CC BY-SA 2.0 |
| `torta-caprese.jpg` | [Tortacapresespot.jpeg](https://commons.wikimedia.org/wiki/File:Tortacapresespot.jpeg) | Mgdongarra | CC0 |
| `semifreddo-al-limone.jpg` | [Semifreddo nocciola.jpg](https://commons.wikimedia.org/wiki/File:Semifreddo_nocciola.jpg) | Daniele2399 | CC BY-SA 4.0 |
| `zeppole.jpg` | [Zeppole stacked on a plate.jpg](https://commons.wikimedia.org/wiki/File:Zeppole_stacked_on_a_plate.jpg) | Luca Sartoni | CC BY-SA 2.0 |
| `torta-della-nonna.jpg` | [Torta della nonna.jpg](https://commons.wikimedia.org/wiki/File:Torta_della_nonna.jpg) | fugzu from Pistoia, Italia | CC BY 2.0 |
| `empanadas-colombianas.jpg` | [Empanada colombiana.jpg](https://commons.wikimedia.org/wiki/File:Empanada_colombiana.jpg) | Francesc Fort | CC BY-SA 4.0 |
| `arepa-con-queso.jpg` | [Arepa de chocolo.jpg](https://commons.wikimedia.org/wiki/File:Arepa_de_chocolo.jpg) | Nelson Fernando Sotelo Castro | CC BY 2.0 |
| `patacones.jpg` | [Patacones - Barranquilla.jpg](https://commons.wikimedia.org/wiki/File:Patacones_-_Barranquilla.jpg) | Jdvillalobos | CC BY 3.0 |
| `ajiaco-santafereno.jpg` | [Ajiaco la puerta falsa.jpg](https://commons.wikimedia.org/wiki/File:Ajiaco_la_puerta_falsa.jpg) | Cesar.cely | CC BY-SA 4.0 |
| `bandeja-paisa.jpg` | [Bandeja paisa, plato Colombiano.jpg](https://commons.wikimedia.org/wiki/File:Bandeja_paisa,_plato_Colombiano.jpg) | Wilfredor | CC0 |
| `sancocho-de-gallina.jpg` | [Sancocho cruzado de gallina, rabo y costilla con arepa.jpg](https://commons.wikimedia.org/wiki/File:Sancocho_cruzado_de_gallina,_rabo_y_costilla_con_arepa.jpg) | Rodolfo pimentel | CC BY-SA 4.0 |
| `brevas-en-almibar.jpg` | [Incir Kompostosu (Fig in Syrup).JPG](https://commons.wikimedia.org/wiki/File:Incir_Kompostosu_(Fig_in_Syrup).JPG) | Miansari66 | CC0 |
| `limonada-de-coco.jpg` | [Coconut Drink (Unsplash).jpg](https://commons.wikimedia.org/wiki/File:Coconut_Drink_(Unsplash).jpg) | Alexander Mils alexandermils | CC0 |
| `cafe-colombiano.jpg` | [A small cup of coffee.JPG](https://commons.wikimedia.org/wiki/File:A_small_cup_of_coffee.JPG) | Julius Schorzman | CC BY-SA 2.0 |
| `agua-de-panela.jpg` | [Aguapanela-Colombiana-3-2.jpg](https://commons.wikimedia.org/wiki/File:Aguapanela-Colombiana-3-2.jpg) | Mappa Stellae | CC BY-SA 4.0 |
| `chocolate-con-queso.jpg` | [Colombian classic - hot chocolate with arepa and cheese (in porcelain with pink flowers).jpg](https://commons.wikimedia.org/wiki/File:Colombian_classic_-_hot_chocolate_with_arepa_and_cheese_(in_porcelain_with_pink_flowers).jpg) | Quinn Comendant | CC BY-SA 2.0 |
| `edamame.jpg` | [Edamame by jark.jpg](https://commons.wikimedia.org/wiki/File:Edamame_by_jark.jpg) | jark | CC BY-SA 2.0 |
| `gyozas-de-cerdo.jpg` | [Pork gyoza - Massachusetts.jpg](https://commons.wikimedia.org/wiki/File:Pork_gyoza_-_Massachusetts.jpg) | Daderot | CC0 |
| `tempura-de-camaron.jpg` | [Tempura Prawns at Southport Surf Life Saving Club, Main Beach, Queensland.jpg](https://commons.wikimedia.org/wiki/File:Tempura_Prawns_at_Southport_Surf_Life_Saving_Club,_Main_Beach,_Queensland.jpg) | Kgbo | CC BY-SA 4.0 |
| `sopa-miso.jpg` | [Miso Soup 001.jpg](https://commons.wikimedia.org/wiki/File:Miso_Soup_001.jpg) | Ocdp | CC0 |
| `ramen-tonkotsu.jpg` | [Tonkotsu Ramen - Goemon Ramen Bar 2023-06-06.jpg](https://commons.wikimedia.org/wiki/File:Tonkotsu_Ramen_-_Goemon_Ramen_Bar_2023-06-06.jpg) | Andy Li | CC0 |
| `california-roll.jpg` | [California Sushi mit Kaviar (26545022496).jpg](https://commons.wikimedia.org/wiki/File:California_Sushi_mit_Kaviar_(26545022496).jpg) | Tim Reckmann from Hamm, Deutschland | CC BY 2.0 |
| `sashimi-de-salmon.jpg` | [Salmon sashimi Yuichiro Haga.jpeg](https://commons.wikimedia.org/wiki/File:Salmon_sashimi_Yuichiro_Haga.jpeg) | Yuichiro Haga | CC BY 2.0 |
| `nigiri-de-salmon.jpg` | [Nigiri Sushi (25966163204).jpg](https://commons.wikimedia.org/wiki/File:Nigiri_Sushi_(25966163204).jpg) | Tim Reckmann from Hamm, Deutschland | CC BY 2.0 |
| `yakisoba-de-pollo.jpg` | [Chicken yakisoba Merelymel13.jpg](https://commons.wikimedia.org/wiki/File:Chicken_yakisoba_Merelymel13.jpg) | Merelymel13 | CC BY-SA 2.0 |
| `mochi-helado.jpg` | [Mochi Ice Cream.jpg](https://commons.wikimedia.org/wiki/File:Mochi_Ice_Cream.jpg) | Charles Nguyen | Public domain |
| `dorayaki.jpg` | [Dorayaki 004.jpg](https://commons.wikimedia.org/wiki/File:Dorayaki_004.jpg) | Ocdp | CC0 |
| `cheesecake-de-matcha.jpg` | [Olea Matcha Cheesecake with olive oil - Olea 2025-07-03.jpg](https://commons.wikimedia.org/wiki/File:Olea_Matcha_Cheesecake_with_olive_oil_-_Olea_2025-07-03.jpg) | Andy Li | CC0 |
| `te-verde-japones.jpg` | [2018 Uji lightly steamed sencha - first infusion.jpg](https://commons.wikimedia.org/wiki/File:2018_Uji_lightly_steamed_sencha_-_first_infusion.jpg) | Difference engine | CC BY-SA 4.0 |
| `provoleta-a-la-parrilla.jpg` | [Provoleta con cazuela (porción).jpg](https://commons.wikimedia.org/wiki/File:Provoleta_con_cazuela_(porci%C3%B3n).jpg) | Horacio Cambeiro | CC BY 3.0 |
| `chorizo-criollo.jpg` | [Chorizo a la parrilla de Argentina.jpg](https://commons.wikimedia.org/wiki/File:Chorizo_a_la_parrilla_de_Argentina.jpg) | Horacio Cambeiro | CC BY 3.0 |
| `mazorca-asada.jpg` | [Grilled Corn on the cob.JPG](https://commons.wikimedia.org/wiki/File:Grilled_Corn_on_the_cob.JPG) | Namitakhaire | CC BY-SA 4.0 |
| `churrasco-de-res.jpg` | [DFC 2244 Juicy grilled steak smothered in savory brown gravy served with roasted potatoes sautéed vegetables and a crisp mixed salad.jpg](https://commons.wikimedia.org/wiki/File:DFC_2244_Juicy_grilled_steak_smothered_in_savory_brown_gravy_served_with_roasted_potatoes_saut%C3%A9ed_vegetables_and_a_crisp_mixed_salad.jpg) | PattayaPatrol | CC BY-SA 4.0 |
| `picana-a-la-brasa.jpg` | [Picanha (14759358544).jpg](https://commons.wikimedia.org/wiki/File:Picanha_(14759358544).jpg) | Christian Benseler | CC BY 2.0 |
| `costillas-bbq.jpg` | [BBQ RIBS pork side ribs.jpg](https://commons.wikimedia.org/wiki/File:BBQ_RIBS_pork_side_ribs.jpg) | WorldwrestlingfederationVKM | CC BY-SA 3.0 |
| `pollo-a-las-brasas.jpg` | [Pollo a la brasa peruano.jpg](https://commons.wikimedia.org/wiki/File:Pollo_a_la_brasa_peruano.jpg) | Manuel González Olaechea | CC BY-SA 3.0 |
| `salmon-a-la-parrilla.jpg` | [Grilled plated salmon fillet.jpg](https://commons.wikimedia.org/wiki/File:Grilled_plated_salmon_fillet.jpg) | DanaTentis | CC0 |
| `brownie-con-helado.jpg` | [Walnut brownie with ice-cream.jpg](https://commons.wikimedia.org/wiki/File:Walnut_brownie_with_ice-cream.jpg) | Sirsendu.mohanta | CC BY-SA 4.0 |
| `flan-de-caramelo.jpg` | [Crème caramel at NB Steak JK.jpg](https://commons.wikimedia.org/wiki/File:Cr%C3%A8me_caramel_at_NB_Steak_JK.jpg) | Oleg Yunakov | CC BY-SA 4.0 |
| `copa-de-vino-tinto.jpg` | [A glass of red wine.jpg](https://commons.wikimedia.org/wiki/File:A_glass_of_red_wine.jpg) | Davide Restivo from Lenzburg, Aarau, Switzerland | CC BY-SA 2.0 |
| `limonada-de-hierbabuena.jpg` | [Homemade Mint Lemonade,Bangladesh.jpg](https://commons.wikimedia.org/wiki/File:Homemade_Mint_Lemonade,Bangladesh.jpg) | Evening42 | CC BY-SA 4.0 |
