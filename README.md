# pms-channel-manager-architecture
High-Level Design (HLD) y arquitectura de un PMS / Channel Manager. Ecosistema Full-Stack tipado con React, Node.js, Prisma y app móvil en Flutter. Showcase de integraciones complejas (iCal, SOAP, Webhooks).
# 🏨 Architecture Showcase: PMS & Channel Manager

> ⚠️ **Nota de Confidencialidad:** El código fuente de este proyecto es propiedad privada y está protegido por un NDA. Este repositorio funciona exclusivamente como un **Documento de Diseño Arquitectónico (HLD)** para exponer el stack tecnológico, las decisiones de infraestructura y la resolución de integraciones complejas.

## 📝 Visión General
Desarrollo Full-Stack de un Property Management System (PMS) dual, compuesto por un **Panel de Administración Web** y una **Aplicación Móvil Nativa**. El sistema centraliza la gestión de reservas, automatiza el cumplimiento normativo gubernamental y mantiene sincronización bidireccional en tiempo real con Agencias de Viajes Online (OTAs) y plataformas externas (WordPress/WooCommerce).

## 🛠️ Stack Tecnológico
Todo el ecosistema está tipado estáticamente de extremo a extremo.

* **Backend / API Core:** Node.js, Express, TypeScript.
* **Capa de Datos:** PostgreSQL gestionado mediante Prisma ORM.
* **Frontend Web (Admin):** React.js, Vite, Tailwind CSS, Playwright (E2E Testing).
* **Frontend Móvil:** Flutter (Dart) con integración de Firebase para Push Notifications.
* **Servicios e Integraciones Externas:** * Cloudinary (Gestión de assets y firmas).
  * WhatsApp API (Notificaciones automáticas).
  * Ministerio del Interior (SES.Hospedaje).

## 🔒 Autenticación y Seguridad (RBAC)
El sistema implementa una arquitectura de seguridad basada en tokens (**JWT**) con un Control de Acceso Basado en Roles (RBAC) granular. 

Los *middlewares* de Express (`verifyToken`, `authorizeRole`) interceptan las peticiones e inyectan el contexto del usuario, permitiendo aislar completamente los recursos. Se han modelado jerarquías estrictas:
* `SUPERADMIN`: Control total de la plataforma y auditoría.
* `ADMIN` / `ARRENDADOR`: Gestión de propiedades, calendarios y tarifas estacionales.
* `RECEPCIONISTA`: Acceso limitado al módulo de check-in móvil y escáner QR.
* `LIMPIEZA`: Vista de solo lectura del estado de las habitaciones.

## 🔄 Sincronización de Datos y Concurrencia
La aplicación móvil y el panel web consumen una **API RESTful**. Para manejar el alto volumen de actualizaciones de disponibilidad y evitar condiciones de carrera (*race conditions*) o *overbooking*, se implementó:

1. **Motor iCal Bidireccional:** Un parser personalizado que lee y emite flujos `.ics`.
2. **Sync Queue:** Un sistema de colas (`syncQueue.ts`) que procesa las colisiones de reservas de forma secuencial.
3. **Webhooks Seguros:** Controladores dedicados (`webhookController.ts`, `wordpressWebhookController.ts`) validados por *payload signatures* para ingestar reservas provenientes de plugins personalizados desarrollados para WordPress (WPRentals/WooCommerce).

## 🚀 Retos Técnicos Resueltos

### 1. Compliance Normativo M2M (SES.Hospedaje)
Automatización del RD 933/2021. Se desarrolló un servicio (`sesHospedajeService.ts`) que transforma los datos relacionales de huéspedes y reservas en estructuras compatibles con la administración pública, empaquetándolos en **XML/SOAP**, encriptando los payloads en **Base64** y gestionando los *retries* ante caídas de la pasarela gubernamental.

### 2. Módulo de Check-in Móvil y Hardware
Para reducir la fricción en recepción, la app en Flutter accede al hardware nativo del dispositivo (cámara y pantalla táctil) para:
* Escanear y validar **Códigos QR** de reserva en tiempo real.
* Capturar firmas biométricas (`signature_page.dart`) y subirlas directamente a buckets seguros en la nube.

## 💾 Modelado Relacional (Prisma Schema Extract)
El diseño de la base de datos está normalizado para manejar relaciones complejas de uno-a-muchos y muchos-a-muchos, optimizando las consultas con índices estratégicos.

```prisma
// Extracto simplificado de la arquitectura de datos
model Property {
  id              String         @id @default(uuid())
  name            String
  icalUrl         String?
  syncStatus      SyncStatus     @default(IDLE)
  ownerId         String
  owner           User           @relation("PropertyOwner", fields: [ownerId], references: [id])
  rooms           Room[]
  externalBookings ExternalBooking[]
}

model Reservation {
  id              String         @id @default(uuid())
  status          BookingStatus  @default(PENDING) // PENDING, CONFIRMED, CANCELLED, BLOCKED
  checkIn         DateTime
  checkOut        DateTime
  totalPrice      Float
  roomId          String
  room            Room           @relation(fields: [roomId], references: [id])
  guests          RentalRecord[] 
  
  @@index([checkIn, checkOut])
}

model RentalRecord {
  id              String      @id @default(uuid())
  reservationId   String
  documentData    String      // Datos cifrados del DNI/Pasaporte
  signatureUrl    String?     // URL segura del asset
  sesReported     Boolean     @default(false)
  reservation     Reservation @relation(fields: [reservationId], references: [id])
}

🏗️ Diagrama de Flujo (Integración Continua)
graph TD
    A[App Flutter] -->|JWT Auth / REST API| B(Node.js / Express Backend)
    C[React Admin Web] -->|JWT Auth / REST API| B
    D[WordPress / OTAs] -->|Webhooks / iCal| B
    
    B -->|Prisma ORM| E[(PostgreSQL)]
    B -->|XML / SOAP| F[API Ministerio del Interior]
    B -->|SDK| G[Cloudinary]
