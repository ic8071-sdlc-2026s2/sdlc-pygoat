# Mapa de la app · OWASP PyGoat

Este documento es andamiaje para la fase 1. Describe qué hace PyGoat, cómo está organizada y qué datos guarda, para que su equipo pueda dibujar el DFD y escribir los casos de abuso. No es una lista de debilidades: esas las encuentra usted con el análisis y las herramientas de las fases siguientes.

## Qué es

PyGoat es una plataforma de aprendizaje de seguridad web hecha con Django. Una persona se registra, lee lecciones sobre el OWASP Top 10, abre laboratorios vulnerables para practicar, resuelve retos con bandera (*flag*) y acumula puntos y logros en un tablero.

Para el laboratorio del curso, trate a PyGoat como lo que es por dentro: un **sistema** con usuarios, autenticación, datos persistentes, un rol de administración y servicios externos. Las lecciones son el «producto» de la plataforma, igual que los pedidos son el producto de una tienda. Sus casos de abuso deben atacar a la plataforma y a sus usuarios, no resolver las lecciones.

Stack: Python 3.11, Django 4.2, django-allauth 0.52 para cuentas y gunicorn como servidor. La base de datos es SQLite, en un archivo dentro del contenedor.

## Cómo corre en este laboratorio

La versión 3.0.0 está pensada para Docker Compose: la app Django, un proxy Traefik y un contenedor aparte por cada laboratorio que abre cada persona. En este curso solo corre la app Django, sin Traefik y sin acceso a Docker. Por eso:

| Funciona | No funciona en el laboratorio |
|:---|:---|
| Registro, inicio y cierre de sesión con usuario y contraseña | Abrir o detener laboratorios: los botones del menú lateral muestran un error |
| Página de inicio y menú lateral con el catálogo de 29 entradas | Iniciar el reto `do-it-fast` y, por lo tanto, enviar su bandera y ganar puntos |
| Páginas de teoría de cada lección (lista abajo) | Inicio de sesión con Google: no hay credenciales de Google configuradas |
| Página del reto, tablero de puntaje y logros (en cero) | Restablecer la contraseña por correo: no hay servidor de correo |
| Administración de Django en `/admin/` | Listar los laboratorios en ejecución de cada persona |

`lab/app.sh` arranca la app con `migrate` y tres comandos que cargan los datos iniciales: `seed_scoring` (niveles de dificultad, logros y ajustes de puntaje), `populate_labs` (catálogo de laboratorios y menú lateral) y `populate_challenges` (el reto). Para el DFD dibuje también lo que no corre aquí (Traefik, Docker y los contenedores de laboratorio): es parte de la arquitectura real de PyGoat y del análisis de riesgo.

## Roles

| Rol | Cómo se obtiene | Qué puede hacer, según el diseño |
|:---|:---|:---|
| Visitante anónimo | Nadie inicia sesión | Ver el login, registrarse, pedir el restablecimiento de contraseña |
| Estudiante | Se registra en `/register` | Ver lecciones y el catálogo, abrir laboratorios y retos, enviar banderas, ver su puntaje y sus logros |
| Personal (*staff*) | Un administrador le marca `is_staff` | Además: registrar, editar y borrar laboratorios personalizados desde el botón «Custom Labs» de la barra superior; entrar a `/admin/` según sus permisos |
| Superusuario | `createsuperuser` en el contenedor | Todo lo anterior y todos los modelos de `/admin/`: usuarios, retos, laboratorios, dificultades, puntajes y logros |
| Traefik (sistema, solo en la arquitectura completa) | Configuración de Docker Compose | Consulta `/auth/verify/` para decidir si una petición puede llegar al laboratorio de una persona |

## Cuentas de prueba

PyGoat no trae cuentas creadas ni documenta cuentas de prueba para la plataforma. Cree las suyas:

- Estudiantes: regístrese en `/register` con usuario, correo y contraseña.
- Superusuario, con la app levantada en el Codespace:

  ```bash
  docker exec -it lab-app python manage.py createsuperuser
  ```

Las cuentas viven en la base SQLite del contenedor. Al detener la app con `bash lab/app.sh detener` se pierden y hay que crearlas de nuevo.

## Pantallas

| Ruta | Acceso previsto | Qué muestra |
|:---|:---|:---|
| `/login/` | anónimo | Formulario de inicio de sesión, enlaces a registro y a Google |
| `/register` | anónimo | Formulario de registro: usuario, correo y contraseña dos veces |
| `/` | estudiante | Introducción a PyGoat; sin sesión redirige al login |
| `/logout/` | estudiante | Cierra la sesión |
| `/password_change/` | estudiante | Cambio de contraseña |
| `/password_reset/` y `/reset/<uidb64>/<token>/` | anónimo | Restablecimiento de contraseña por correo |
| `/accounts/...` | según la página | Páginas de django-allauth: login, registro, correos, cambio y restablecimiento de contraseña, cuentas de Google |
| `/challenge/<nombre>` | estudiante | Página de un reto, con el botón para iniciarlo y el campo para enviar la bandera |
| `/scoring/dashboard/` | estudiante | Puntaje total, retos resueltos, logros y su progreso |
| `/admin/` | personal o superusuario | Administración de Django |

Páginas de teoría de las lecciones, una por tema: `/xss`, `/sql`, `/insec_des`, `/xxe`, `/auth`, `/ba`, `/data_exp`, `/cmd`, `/bau`, `/sec_mis`, `/a9`, `/a10`, `/insecure-design`, `/broken_access_control`, `/ssrf`, `/injection`, `/ssti`, `/cryptographic_failure`, `/A03`, `/auth_failure` y `/2021/A8`. El menú lateral no enlaza todas: escriba la ruta en el navegador.

## Endpoints JSON

Los usa el JavaScript de `introduction/static/js/lab_manager.js` y de la página del reto.

| Método | Ruta | Acceso previsto | Qué hace |
|:---|:---|:---|:---|
| GET | `/challenge/start-lab/<lab>/` | estudiante | Construye si hace falta y arranca el contenedor del laboratorio para esa persona |
| POST | `/challenge/stop-lab/<lab>/` | estudiante | Detiene y borra un laboratorio propio |
| POST | `/challenge/stop-labs/` | estudiante | Detiene todos los laboratorios propios |
| GET | `/challenge/list-labs/` | estudiante | Lista los laboratorios propios en ejecución |
| GET | `/challenge/custom-labs/list/` | estudiante | Lista los laboratorios personalizados (alimenta el menú lateral) |
| POST | `/challenge/custom-labs/create/` | personal | Registra un laboratorio personalizado: nombre, carpeta de construcción y puerto |
| POST | `/challenge/custom-labs/update/<id>/` | personal | Edita un laboratorio personalizado |
| POST | `/challenge/custom-labs/delete/<id>/` | personal | Borra un laboratorio personalizado |
| POST | `/challenge/<nombre>` | estudiante | Inicia el contenedor del reto |
| PUT | `/challenge/<nombre>` | estudiante | Envía una bandera; si es correcta, suma puntos |
| DELETE | `/challenge/<nombre>` | estudiante | Detiene el contenedor del reto |
| GET | `/scoring/scores/` | estudiante | Puntaje total y los últimos 20 puntajes |
| GET | `/scoring/achievements/` | estudiante | Logros obtenidos |
| GET | `/auth/verify/` | Traefik | Responde 200, 401 o 403 según si la persona es dueña del laboratorio pedido |

La columna «Acceso previsto» describe la intención del diseño. Parte de su trabajo en las fases 1, 2 y 4 es comprobar si la app de verdad la respeta.

## Datos que guarda

Todo vive en `db.sqlite3`, dentro del contenedor.

| Dato | Modelo o tabla | Dónde se define |
|:---|:---|:---|
| Cuentas: usuario, correo, hash de la contraseña, marcas `is_staff` e `is_superuser`, último acceso | `auth_user` | Django |
| Sesiones activas | `django_session`; el navegador guarda la cookie `sessionid` | Django |
| Token anti-CSRF | cookie `csrftoken` | Django |
| Correos verificados y cuentas de Google enlazadas, credenciales de la app de Google | `EmailAddress`, `SocialAccount`, `SocialToken`, `SocialApp` | django-allauth |
| Catálogo de laboratorios: nombre, carpeta de construcción, puerto, si es personalizado | `Lab`, `LabCategoryMapping` | `challenge/models.py` |
| Retos: imagen de Docker, puertos, puntos y la bandera guardada como hash SHA-256 | `Challenge` | `challenge/models.py` |
| Progreso de cada persona en cada reto: contenedor, puerto, intentos, resuelto | `UserChallenge` | `challenge/models.py` |
| Puntaje: inicio y fin de cada intento, duración, puntos calculados | `LabCompletion`, `LabScore` | `scoring/models.py` |
| Reglas de puntaje: dificultades, multiplicadores, bonificación por tiempo | `DifficultyLevel`, `ScoringSettings` | `scoring/models.py` |
| Logros definidos y logros obtenidos | `AchievementMilestone`, `UserAchievement` | `scoring/models.py` |
| Acciones hechas desde `/admin/` | `django_admin_log` | Django |

## Integraciones externas

| Integración | Para qué | Estado en el laboratorio |
|:---|:---|:---|
| Motor de Docker (SDK `docker`, socket `/var/run/docker.sock`) | Construir y arrancar los contenedores de laboratorios y retos | No disponible |
| Traefik | Publicar cada laboratorio en `lab-<usuario>-<lab>.<dominio>` y pedir permiso a `/auth/verify/` | No corre |
| Docker Hub | Descargar la imagen del reto `do-it-fast` | No se usa |
| Google (OAuth 2.0 vía django-allauth) | Iniciar sesión con una cuenta de Google | Declarado, sin credenciales |
| Servidor de correo (SMTP) | Enviar el enlace de restablecimiento de contraseña | No configurado |
| CDN públicas (Bootstrap, jQuery, Font Awesome, cdnjs) | Hojas de estilo y scripts que carga el navegador | Las carga el navegador del usuario |

## Dónde vive cada parte en `app/`

| Carpeta o archivo | Qué contiene |
|:---|:---|
| `pygoat/settings.py`, `pygoat/urls.py` | Configuración de Django y rutas principales |
| `introduction/` | Registro, inicio, páginas de teoría, `/auth/verify/`, plantillas base (`templates/introduction/`) y estáticos (`static/`) |
| `introduction/static/js/lab_manager.js` | JavaScript que abre y detiene laboratorios y administra los personalizados |
| `challenge/` | Modelos de laboratorios y retos, vistas de `/challenge/...`, comandos `populate_labs`, `populate_challenges` y `startuplabs`, datos del reto en `challenge.json` |
| `scoring/` | Puntaje, logros, tablero y comando `seed_scoring` |
| `dockerized_labs/` | Los 21 laboratorios como apps Flask o Django independientes, cada una con su Dockerfile; no corren en este laboratorio |
| `challenge/labs/` | Los tres retos nuevos de la v3 como apps Flask independientes; tampoco corren aquí |
| `Dockerfile`, `requirements.txt` | Imagen de la app Django que construye `lab/app.sh` |
| `docker-compose.yml`, `traefik.yml` | Despliegue completo del proyecto original; este laboratorio no los usa |
| `Solutions/` | Soluciones de los ejercicios de las lecciones, publicadas por el proyecto |

Una pregunta útil para el DFD: ¿cuáles de estas carpetas forman parte del sistema que se despliega y cuáles son el contenido que la plataforma ofrece?
