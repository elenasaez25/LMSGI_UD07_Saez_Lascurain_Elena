# Manual Técnico de Explotación
## Sistema ERP/CRM — WillmanTech S.L.

> **Alumno:** Elena Sáez
> **Fecha:** 22/05/2026

---

## Índice de contenidos

1. [Introducción y Arquitectura del Sistema](#1-introducción-y-arquitectura-del-sistema)
2. [Guía de Instalación y Reinstalación](#2-guía-de-instalación-y-reinstalación)
3. [Seguridad y Control de Acceso](#3-seguridad-y-control-de-acceso)
4. [Procedimiento de Backup y Restauración](#4-procedimiento-de-backup-y-restauración)
5. [Flujo Operativo de Facturación e Informes](#5-flujo-operativo-de-facturación-e-informes)
6. [Consultas IA](#6-consultas-ia)

---

## 1. Introducción y Arquitectura

### 1.1 Propósito del documento

Este manual recoge los procedimientos técnicos necesarios para operar, mantener y recuperar el sistema ERP de WillmanTech S.L. Está dirigido a dos perfiles: **administradores de sistemas**, responsables del despliegue y mantenimiento de la infraestructura, y **usuarios avanzados**, que gestionan los procesos de negocio dentro de la aplicación.

### 1.2 Descripción del entorno tecnológico

La plataforma elegida es **Odoo 16 Community**, un ERP de código abierto que integra en una sola aplicación los módulos de ventas, facturación, CRM, inventario y contabilidad.

El sistema se ejecuta en un entorno **contenerizado con Docker**, lo que garantiza que la aplicación funcione de forma idéntica en cualquier máquina, simplifica las actualizaciones y facilita la recuperación ante fallos.

### 1.3 Propósito operativo de los módulos

| Módulo | Propósito operativo |
|---|---|
| **Ventas** | Presupuestos, pedidos y tarifas de clientes |
| **Facturación** | Emisión de facturas, notas de crédito y cobros |
| **CRM** | Gestión del pipeline comercial y oportunidades |
| **Inventario** | Catálogo de productos y control de existencias |
| **Contabilidad** | Plan contable, impuestos y reporting financiero |

### 1.4 Funcionamiento del despliegue

El sistema se compone de **dos servicios principales** orquestados mediante Docker Compose.
Los datos persisten en **volúmenes de Docker gestionados**, por lo que no se pierden al reiniciar o actualizar los contenedores.

---

## 2. Guía de Instalación y Reinstalación

### 2.1 Requisitos del servidor

Antes de comenzar la instalación, verifica que el servidor cumple los siguientes requisitos mínimos:

| Componente | Requisito mínimo | Recomendado |
|---|---|---|
| Sistema operativo | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| RAM | 4 GB | 8 GB |
| Espacio en disco | 20 GB | 50 GB |
| Docker Engine | v24+ | última estable |
| Docker Compose | v2.20+ | última estable |

Comprueba las versiones instaladas con:

```bash
docker --version
docker compose version
```

### 2.2 Organización de ficheros del proyecto

```
willmantech-erp/
├── docker-compose.yml          # Orquestación de servicios
├── .env                        # Variables de entorno (no subir a Git)
├── addons/                     # Módulos Odoo personalizados
│   └── report_invoice_willmantech/
└── backups/                    # Destino de las copias de seguridad
```

### 2.3 Configuración de variables de entorno

Crea el fichero `.env` en la raíz del proyecto con el contenido siguiente. Sustituye los valores de ejemplo por credenciales reales y seguras:

```env
# === PostgreSQL ========================================
POSTGRES_DB=willmantech_db
POSTGRES_USER=odoo_user
POSTGRES_PASSWORD=Clave_Segura_Produccion_2025!

# === Odoo ========================================
# Contraseña maestra del gestor de bases de datos (/web/database/manager)
ODOO_ADMIN_PASSWD=MasterPass_WillmanTech!
```

> **¡Importante!** El fichero `.env` nunca debe subirse a repositorios de código. Añádelo a un repositorio que se llame `.gitignore`.

### 2.4 Contenido del fichero docker-compose.yml

```yaml
version: '3.8'

services:

  db:
    image: postgres:15
    restart: always
    env_file: .env
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - willmantech_net

  odoo:
    image: odoo:16
    restart: always
    depends_on:
      - db
    ports:
      - "8069:8069"
    environment:
      HOST: db
      USER: ${POSTGRES_USER}
      PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - odoo_data:/var/lib/odoo
      - ./addons:/mnt/extra-addons
    networks:
      - willmantech_net

volumes:
  postgres_data:
  odoo_data:

networks:
  willmantech_net:
    driver: bridge
```

### 2.5 Secuencia de arranque

Ejecuta los siguientes comandos desde el directorio raíz del proyecto:

```bash
# 1. Descargar las imágenes de los registros oficiales de Docker
docker compose pull

# 2. Iniciar todos los servicios en modo segundo plano
docker compose up -d

# 3. Verificar que ambos contenedores están en estado "running"
docker compose ps

# 4. Seguir los registros de Odoo en tiempo real (útil para detectar errores)
docker compose logs -f odoo
```

Transcurrido aproximadamente un minuto, la interfaz estará disponible en `http://localhost:8069`. En el primer acceso, el asistente de Odoo solicitará crear la base de datos inicial.

### 2.6 Procedimiento de reinstalación

Si necesitas destruir el entorno por completo y volver a empezar:

```bash
# Detener contenedores y eliminar volúmenes de datos
docker compose down -v

# Opcional: eliminar imágenes descargadas para forzar una descarga fresca
docker rmi odoo:16 postgres:15

# Reiniciar desde cero
docker compose up -d
```

> El parámetro `-v` borra todos los datos almacenados. Asegúrate de tener un backup antes de ejecutar este comando.

---

## 3. Seguridad y Control de Acceso

### 3.1 Modelo de roles

El sistema implementa tres perfiles de acceso diferenciados según el principio de **mínimo privilegio**:

**Administrador del sistema**
Acceso completo a toda la configuración técnica, gestión de usuarios, módulos y parámetros del sistema. Solo debe asignarse al personal de IT responsable.

**Contable**
Acceso a todos los módulos financieros: facturas, pagos, extractos bancarios, libros contables y exportaciones fiscales. No puede modificar la configuración del sistema ni crear usuarios.

**Comercial**
Acceso a CRM, presupuestos y pedidos de venta. Ve únicamente sus propios clientes y oportunidades. No accede a datos financieros ni a registros de otros comerciales.

### 3.2 Alta de un nuevo usuario

1. Acceder a **Ajustes → Usuarios y compañías → Usuarios**
2. Pulsar **Nuevo**
3. Completar nombre completo y dirección de correo corporativa
4. Asignar el perfil en la sección **Permisos de acceso**
5. Guardar — el sistema enviará automáticamente un correo de activación

### 3.3 Política de contraseñas obligatoria

| Requisito | Valor |
|---|---|
| Longitud mínima | 8 caracteres |
| Complejidad | Mayúsculas + números obligatorios |
| Caducidad | 90 días |
| Historial | No reutilizar las últimas 5 contraseñas |
| Intentos fallidos | Bloqueo tras 5 intentos consecutivos |

Configuración en Odoo: **Ajustes → Permisos → Política de contraseñas**

### 3.4 Restricciones adicionales recomendadas

- Habilitar **autenticación en dos pasos (2FA)** para todos los usuarios con rol Administrador o Contable.
- Aplicar **reglas de registro** en el módulo CRM para que cada comercial vea únicamente sus propios contactos.
- Restringir el acceso a **Ajustes técnicos** mediante el grupo "Administrador técnico".

---

## 4. Procedimiento de Backup y Restauración

### 4.1 Estrategia de copias de seguridad

| Frecuencia | Retención | Destino |
|---|---|---|
| Diaria | 7 copias | Servidor local |
| Semanal | 4 copias | Almacenamiento externo |
| Mensual | 12 copias | Servicio cloud (ej. S3) |

### 4.2 Realizar un backup

**Desde la interfaz web** *(recomendado para usuarios no técnicos)*

1. Navegar a `http://localhost:8069/web/database/manager`
2. Pulsar **Backup** junto al nombre de la base de datos
3. Seleccionar formato **zip** — incluye base de datos y todos los ficheros adjuntos
4. Guardar el archivo descargado en una ubicación segura

**Desde la terminal** *(recomendado para automatización)*

```bash
# Exportar la base de datos a un fichero SQL comprimido
docker exec willmantech-db-1 \
    pg_dump -U odoo_user willmantech_db \
    | gzip > backups/willmantech_$(date +%Y-%m-%d).sql.gz
```

**Tarea programada — backup automático diario a las 02:00 h**

```bash
# Editar el crontab del usuario del servidor
crontab -e

# Añadir la siguiente entrada
0 2 * * * docker exec willmantech-db-1 pg_dump -U odoo_user willmantech_db | gzip > /srv/backups/willmantech_$(date +\%Y-\%m-\%d).sql.gz
```

### 4.3 Restaurar desde un backup

**Desde la interfaz web**

1. Ir a `http://localhost:8069/web/database/manager`
2. Pulsar **Restore Database**
3. Seleccionar el fichero `.zip` del backup
4. Indicar un nombre para la base de datos restaurada
5. Pulsar **Continuar** y esperar a que finalice el proceso

**Desde la terminal**

```bash
# Copiar el backup comprimido al interior del contenedor
docker cp backups/willmantech_2025-05-15.sql.gz willmantech-db-1:/tmp/

# Descomprimir el fichero dentro del contenedor
docker exec willmantech-db-1 gunzip /tmp/willmantech_2025-05-15.sql.gz

# Restaurar el volcado en la base de datos
docker exec willmantech-db-1 \
    psql -U odoo_user -d willmantech_db \
    -f /tmp/willmantech_2025-05-15.sql
```

---

## 5. Flujo Operativo de Facturación e Informes

### 5.1 Crear y emitir una factura paso a paso

**Paso 1 — Acceder al módulo**
Desde el menú principal, ir a **Facturación → Clientes → Facturas** y pulsar **Nuevo**.

**Paso 2 — Rellenar los datos principales**
Seleccionar el cliente en el campo correspondiente. Odoo cargará automáticamente su dirección fiscal, condiciones de pago y datos bancarios si están configurados.

**Paso 3 — Añadir líneas de detalle**
Pulsar **Añadir una línea** por cada producto o servicio facturado. Indicar descripción, cantidad y precio unitario. El IVA se aplicará automáticamente según el producto.

**Paso 4 — Confirmar la factura**
Al pulsar **Confirmar**, la factura pasa a estado *Publicada*: se le asigna un número de serie correlativo y queda registrada contablemente. Esta acción es irreversible; si hay un error, deberá emitirse una nota de crédito.

**Paso 5 — Enviar al cliente**
Pulsar **Enviar e imprimir** para enviar la factura por correo electrónico o descargarla directamente en PDF.

### 5.2 Generación del PDF

Cuando el sistema genera el PDF de una factura, ejecuta internamente el siguiente proceso de tres etapas:  

**Etapa 1 — Plantilla QWeb:** El fichero `report_invoice_willmantech.xml` define la maquetación de la factura. Contiene directivas como `t-foreach` (para iterar las líneas) y `t-if` (para ocultar columnas vacías).

**Etapa 2 — Generación del HTML:** El motor interno de Odoo procesa la plantilla, consulta la base de datos PostgreSQL y sustituye cada directiva `t-field` por el valor real del campo correspondiente (nombre del cliente, importes, fechas...). El resultado es un documento HTML completo.

**Etapa 3 — Conversión a PDF con wkhtmltopdf:** La herramienta `wkhtmltopdf`, incluida en la imagen Docker de Odoo, recibe el HTML generado y lo renderiza como un PDF de calidad tipográfica listo para imprimir o enviar.

### 5.3 Exportar informes de ventas y análisis

1. Ir a **Facturación → Informes → Análisis de facturas**
2. Aplicar filtros de fecha (mes, trimestre, año fiscal)
3. Agrupar los resultados por cliente, comercial o producto según necesidad
4. Exportar a **Excel** para análisis adicional o a **PDF** para su archivo

---

## 6. Consultas IA

Este apartado recoge las consultas realizadas a herramientas de inteligencia artificial durante el desarrollo del proyecto, incluyendo los prompts utilizados y las respuestas obtenidas que sirvieron de apoyo técnico.

---

### #1

**Agente:** Claude

**Prompt:**
> Tengo este fragmento de código QWeb en mi plantilla de factura de Odoo. ¿Puedes explicarme cómo funciona la condición `t-if` y la iteración `t-foreach` que aparecen en él?
>
> ```xml
> <t t-if="any(line.discount for line in doc.invoice_line_ids)">
>     <th style="padding:10px 12px; text-align:center;">Dto. (%)</th>
> </t>
> ...
> <t t-foreach="doc.invoice_line_ids" t-as="line">
>     <tr style="border-bottom:1px solid #e8e8e8;">
>         <td><t t-field="line.name"/></td>
>         <td><t t-field="line.quantity"/></td>
>         <td><t t-field="line.price_unit"/></td>
>     </tr>
> </t>
> ```

**Respuesta textual:**

El código tiene dos directivas QWeb distintas que trabajan juntas para construir la tabla de la factura de forma dinámica.

La directiva **`t-if`** actúa como un interruptor: evalúa la expresión Python `any(line.discount for line in doc.invoice_line_ids)`, que recorre todas las líneas de la factura y devuelve `True` si al menos una de ellas tiene un descuento mayor que cero. Si el resultado es `True`, Odoo renderiza el elemento `<th>` con la cabecera "Dto. (%)". Si ninguna línea tiene descuento, la expresión devuelve `False` y esa columna entera desaparece del HTML generado, evitando que aparezca una columna vacía en el PDF final.

La directiva **`t-foreach`** funciona como un bucle `for`. La expresión `t-foreach="doc.invoice_line_ids"` le indica a QWeb que itere sobre la lista de líneas de la factura, y `t-as="line"` asigna cada elemento de esa lista a la variable temporal `line`. Por cada línea encontrada se genera un bloque `<tr>` completo con sus celdas. Dentro de ese bloque, `t-field` se encarga de imprimir el valor real del campo del ERP: `line.name` muestra la descripción del producto, `line.quantity` la cantidad y `line.price_unit` el precio unitario. El resultado es que Odoo pinta automáticamente tantas filas como líneas tenga la factura, sin necesidad de escribir el HTML a mano para cada una.

---

### #2

**Agente:** Claude

**Prompt:**
> Explícame con ejemplos cómo funcionan los namespaces de componentes agregados (`cac`) y componentes básicos (`cbc`) en UBL, y qué es el ID de personalización europeo compatible con la red PEPPOL.

**Respuesta textual:**

En el estándar UBL 2.1, los namespaces `cac` y `cbc` son prefijos XML que organizan los elementos del documento en dos categorías según su complejidad.

**`cbc` — CommonBasicComponents** agrupa los campos simples que contienen un único valor de texto, número o fecha. Son los elementos "hoja" del documento, aquellos que no contienen otros elementos dentro. Por ejemplo:

```xml
<cbc:ID>FACT-2025-0042</cbc:ID>
<cbc:IssueDate>2025-05-15</cbc:IssueDate>
<cbc:Name>WillmanTech S.L.</cbc:Name>
<cbc:TaxAmount currencyID="EUR">210.00</cbc:TaxAmount>
```

**`cac` — CommonAggregateComponents** agrupa los bloques complejos que actúan como contenedores de otros elementos. Son estructuras compuestas que agrupan información relacionada. Por ejemplo, la dirección postal de una empresa no es un campo simple, sino un conjunto de campos que se agrupan bajo un elemento `cac`:

```xml
<cac:PostalAddress>
    <cbc:StreetName>Calle Tecnología, 10</cbc:StreetName>
    <cbc:CityName>Sevilla</cbc:CityName>
    <cbc:PostalZone>41092</cbc:PostalZone>
    <cac:Country>
        <cbc:IdentificationCode>ES</cbc:IdentificationCode>
    </cac:Country>
</cac:PostalAddress>
```

Respecto al **ID de personalización europeo PEPPOL**, es una cadena de texto que se coloca en el campo `CustomizationID` al inicio del documento y declara qué versión exacta del estándar se está aplicando. La red PEPPOL (Pan-European Public Procurement OnLine) es la infraestructura europea de intercambio de documentos fiscales entre empresas y administraciones públicas. Para que una factura sea aceptada en esta red debe declarar explícitamente su perfil de conformidad:

```xml
<cbc:CustomizationID>
    urn:cen.eu:en16931:2017#compliant#urn:fdc:peppol.eu:2017:poacc:billing:3.0
</cbc:CustomizationID>
```

Esta cadena indica que el documento cumple la norma europea EN 16931:2017 y que además sigue el perfil de facturación PEPPOL BIS Billing 3.0, que es el requerido para intercambio transfronterizo de facturas en Europa.

---

### #3

**Agente:** Claude

**Prompt:**
> ¿Por qué `CustomizationID` y `ProfileID` son obligatorios en una factura PEPPOL? ¿Qué función cumple cada uno exactamente?

**Respuesta textual:**

Ambos campos son obligatorios porque son el mecanismo que utiliza la red PEPPOL para identificar y validar automáticamente el tipo de documento que recibe antes de procesarlo.

**`CustomizationID`** especifica el conjunto exacto de reglas de negocio y restricciones técnicas con las que se ha construido el documento. Indica qué versión del estándar europeo EN 16931 se aplica y qué extensión PEPPOL se ha utilizado. Los sistemas receptores leen este campo para saber qué validaciones aplicar al documento. Sin él, el sistema no puede determinar cómo interpretar el contenido ni si cumple los requisitos legales europeos.

```xml
<cbc:CustomizationID>
    urn:cen.eu:en16931:2017#compliant#urn:fdc:peppol.eu:2017:poacc:billing:3.0
</cbc:CustomizationID>
```

**`ProfileID`** identifica el proceso de negocio concreto al que pertenece el documento. PEPPOL define distintos perfiles para distintos escenarios (facturación, pedidos, albaranes...). El valor `urn:fdc:peppol.eu:2017:poacc:billing:01:1.0` indica que el documento forma parte del proceso de facturación estándar (Billing Process 01).

```xml
<cbc:ProfileID>urn:fdc:peppol.eu:2017:poacc:billing:01:1.0</cbc:ProfileID>
```

En resumen: `CustomizationID` responde a la pregunta *¿con qué reglas está construido este documento?* y `ProfileID` responde a *¿dentro de qué proceso de negocio se enmarca?*. Juntos permiten que cualquier sistema receptor conectado a la red PEPPOL procese la factura correctamente de forma automática, sin intervención manual.

---

### #4

**Agente:** Claude

**Prompt:**
> Ayúdame a estructurar correctamente el fichero `invoice_ubl.xml` para que cumpla con el estándar UBL 2.1 y sea compatible con PEPPOL. Explícame qué bloques debe tener y en qué orden.
**Respuesta textual:**

Un fichero `invoice_ubl.xml` válido para PEPPOL debe organizarse en siete bloques principales, siempre en este orden:

**Bloque 1 — Identificación del perfil y el documento.** Es la cabecera obligatoria. Debe incluir el `CustomizationID` con el identificador PEPPOL europeo, el `ProfileID` del proceso de facturación, el número único de la factura, la fecha de emisión, la fecha de vencimiento, el código del tipo de documento (380 para factura ordinaria) y la divisa utilizada.

**Bloque 2 — Datos del emisor (proveedor).** Contiene toda la información de quien emite la factura: nombre legal de la empresa, dirección postal completa con país, número de identificación fiscal con prefijo del país, y opcionalmente datos de contacto como correo electrónico y teléfono.

**Bloque 3 — Datos del receptor (cliente).** Misma estructura que el bloque anterior pero con la información de la empresa compradora que recibirá la factura.

**Bloque 4 — Condiciones de pago.** Describe cómo y cuándo se debe realizar el pago: plazo acordado, medio de pago (transferencia, tarjeta...) y, si corresponde, el número de cuenta bancaria del beneficiario.

**Bloque 5 — Desglose de impuestos.** Especifica el importe total de impuestos aplicados, la base imponible sobre la que se calculan y el porcentaje de cada tipo impositivo. En España lo habitual es IVA al 21 %, 10 % o 4 % según el tipo de bien o servicio.

**Bloque 6 — Totales monetarios.** Resume los importes globales de la factura: suma de líneas sin impuestos, base imponible, total con IVA incluido y cantidad final a abonar.

**Bloque 7 — Líneas de detalle.** Una entrada por cada producto o servicio facturado, con su número de línea, cantidad, unidad de medida, importe de línea, descripción completa del artículo y precio unitario.

---

### #5

**Agente:** Claude
**Prompt:**
> ¿Cómo puedo cumplir con los requisitos de calidad y usabilidad del estándar internacional ISO/IEC/IEEE 26514:2022 al redactar un manual técnico de explotación de software?

**Respuesta textual:**

El estándar ISO/IEC/IEEE 26514:2022 establece los requisitos que debe cumplir la documentación técnica de un sistema software para que sea útil, comprensible y mantenible. Aplicarlo correctamente implica trabajar en cuatro dimensiones:

**Estructura y completitud.** El documento debe cubrir todos los aspectos que un usuario necesita para operar el sistema de forma autónoma: instalación, configuración, uso habitual, seguridad, mantenimiento y recuperación ante fallos. Cada sección debe tener un propósito claro y diferenciado, y el conjunto debe seguir una progresión lógica desde lo general a lo específico.

**Precisión técnica.** Toda la información debe ser verificable y reproducible. Los comandos, rutas de ficheros, nombres de campos y valores de configuración tienen que ser exactos. Un manual que contiene un comando incorrecto no solo es inútil, sino peligroso en un entorno productivo.

**Usabilidad para el lector objetivo.** El estándar exige adaptar el lenguaje y el nivel de detalle al perfil del destinatario. Un administrador de sistemas y un usuario de negocio tienen necesidades distintas. El texto debe evitar ambigüedades, usar terminología consistente a lo largo de todo el documento y apoyarse en tablas, diagramas y ejemplos cuando el texto solo no sea suficiente.

**Mantenibilidad del documento.** El manual debe incluir un historial de versiones que permita rastrear los cambios realizados. Cada procedimiento debe estar escrito de forma modular para que pueda actualizarse de manera independiente cuando el sistema evolucione, sin necesidad de reescribir el documento completo.

---

*Este documento ha sido redactado conforme a los requisitos de contenido, estructura y usabilidad establecidos por el estándar **ISO/IEC/IEEE 26514:2022**