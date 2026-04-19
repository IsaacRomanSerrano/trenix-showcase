# Trenix — Sistema de preparación estructurada

> Aplicación web mobile-first que genera planes de entrenamiento y nutrición personalizados mediante IA, con ajuste semanal automático basado en el progreso real del usuario.

**Live:** [trenix.net](https://trenix.net)

---

## Producto

Trenix reemplaza al preparador físico para usuarios de gimnasio que buscan estructura sin pagar 100€/mes. El usuario completa un perfil una sola vez; el sistema genera un programa completo de entrenamiento y dieta adaptado a su fase (volumen, definición, recomposición), nivel y disponibilidad. Cada semana, un check-in de 2 minutos actualiza el plan según la progresión real de cargas y métricas corporales.

---

## Stack técnico

### Frontend
| Tecnología | Uso |
|---|---|
| **Next.js 16** (App Router) | Framework principal — SSR, RSC, rutas de API |
| **TypeScript** | Tipado completo end-to-end |
| **Tailwind CSS v4** | Design system con CSS variables, tema oscuro |
| **Vercel** | Hosting + CDN + Edge Functions |

### Backend
| Tecnología | Uso |
|---|---|
| **FastAPI** (Python 3.12) | API REST — async, tipado con Pydantic v2 |
| **SQLAlchemy 2.0** | ORM + query builder |
| **Alembic** | Migraciones de base de datos |
| **Uvicorn** | ASGI server, 2 workers |
| **Docker** | Containerización para deploy en Render |

### Infraestructura y servicios
| Tecnología | Uso |
|---|---|
| **PostgreSQL** (Neon) | Base de datos principal — serverless |
| **Redis** | Cola de workers para notificaciones y recordatorios |
| **Render** | Hosting del backend — Docker container |
| **Stripe** | Suscripciones premium, webhooks |
| **Resend** | Emails transaccionales y recordatorios semanales |
| **Google OAuth 2.0** | Autenticación social |

### IA
| Tecnología | Uso |
|---|---|
| **Anthropic Claude API** | Motor de generación de planes (LVM) |
| **Pydantic v2** | Validación estructural del output del LLM |

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────┐
│                     trenix.net (Vercel)                  │
│                                                         │
│  Next.js App Router                                     │
│  ├── Server Components (landing, SEO, metadata)         │
│  ├── Client Components (auth, training, nutrition)      │
│  └── Edge Functions (OG image generation)               │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS + JWT (Authorization header)
                       │ Refresh token (HttpOnly cookie)
┌──────────────────────▼──────────────────────────────────┐
│              API REST (Render — Docker)                  │
│                                                         │
│  FastAPI                                                │
│  ├── /auth       JWT + Google OAuth + Magic Link        │
│  ├── /users      Perfil y estado del usuario            │
│  ├── /plans      Generación y consulta de planes        │
│  ├── /training   Registro de sesiones y progresión      │
│  ├── /checkin    Check-in semanal y ajuste de plan      │
│  ├── /subscribe  Stripe webhooks y suscripciones        │
│  └── /admin      Panel de administración                │
│                                                         │
│  LVM Engine                                             │
│  ├── Prompt construction (user state → structured JSON) │
│  ├── Claude API call                                    │
│  └── Pydantic validation (output schema enforcement)    │
└──────────┬────────────────────────┬────────────────────┘
           │                        │
┌──────────▼──────────┐  ┌──────────▼──────────┐
│  PostgreSQL (Neon)   │  │  Redis              │
│                     │  │                     │
│  users              │  │  Weekly check-in    │
│  plans              │  │  reminder queue     │
│  workout_sessions   │  │  Push notification  │
│  checkins           │  │  workers            │
│  subscriptions      │  └─────────────────────┘
└─────────────────────┘
```

---

## Funcionalidades implementadas

### Core
- **Onboarding** — Recoge objetivo, nivel, disponibilidad, restricciones alimentarias y métricas físicas
- **LVM (Language-based Validation Model)** — Genera planes completos de entrenamiento + nutrición vía Claude, con validación estructural Pydantic. Si el output del LLM no cumple el schema, la generación falla de forma controlada
- **Plan de entrenamiento** — Estructura semanal completa: ejercicios con series, reps, intensidad (RPE/RIR), alternativas biomecánicas y cardio según fase
- **Plan de nutrición** — Macros diarios + comidas estructuradas con sustituciones por alimento, adaptadas a restricciones y fase
- **Registro de sesiones** — El usuario registra los pesos reales por set; el sistema usa esos datos para calcular progresión de cargas

### Progression engine
- **Check-in semanal** — El usuario reporta peso corporal, adherencia y sensaciones; el backend calcula ajustes proporcionales de volumen, carga y calorías
- **Ajuste de plan** — El LVM recibe el historial de progresión y el contexto semanal para regenerar o ajustar el plan sin perder continuidad

### Auth
- JWT con refresh token rotation (refresh en HttpOnly cookie, access token en memoria)
- Google OAuth 2.0
- Magic link por email
- Verificación de email

### Producto
- **Suscripción premium** vía Stripe — checkout, webhooks, sincronización de tier en cada login
- **Panel de administración** — gestión de usuarios, visualización de planes generados, trazas de generación LVM, toggle manual de premium
- **PWA** — Web App Manifest + Service Worker (network-first, sin cacheo de API)
- **Notificaciones push web** + recordatorio semanal por email (viernes)

---

## SEO y rendimiento

Resultado de auditoría Lighthouse (mobile):

| Métrica | Score |
|---|---|
| Performance | **99 / 100** |
| SEO | **100 / 100** |
| Accessibility | **97 / 100** |
| First Contentful Paint | 1.0 s |
| Largest Contentful Paint | 2.0 s |
| Total Blocking Time | 10 ms |
| Cumulative Layout Shift | 0 |

Implementaciones SEO:
- `sitemap.xml` generado dinámicamente por Next.js App Router
- `robots.txt` con declaración de sitemap
- JSON-LD estructurado: `SoftwareApplication`, `WebSite`, `Organization`, `FAQPage`
- Dynamic OG image generada en Edge (Next.js `ImageResponse`)
- `metadataBase` + OpenGraph + Twitter Cards
- Server Components para contenido crítico de landing (reduce bundle JS del cliente)

---

## Decisiones técnicas destacadas

**Validación de output LLM con Pydantic**
El LVM no confía en el texto libre del modelo. Cada respuesta se parsea contra un schema Pydantic v2 estricto (`extra="forbid"`). Si el JSON del modelo no cumple el contrato (campos faltantes, tipos incorrectos, rangos inválidos), la generación devuelve `generation_status: "blocked"` con razones explícitas. Esto elimina estados corruptos en base de datos.

**Refresh token en HttpOnly cookie**
El access token vive en memoria del cliente (no en `localStorage`), expiración corta. El refresh token se almacena en cookie HttpOnly, inaccessible desde JavaScript. Al montar la app, se hace un background sync silencioso contra `/auth/me` para sincronizar el subscription tier sin bloquear la UI.

**Backward compatibility de planes en DB**
Al añadir nuevos campos al schema del LVM (e.g. `CardioRecommendation`), los planes existentes en base de datos tienen el schema antiguo. `TrainingDayPlan` usa `extra="ignore"` (Pydantic) para silenciar campos obsoletos en reads, mientras que los schemas de output nuevos usan `extra="forbid"` para forzar contratos estrictos en generación.

---

## Repositorio privado

El código fuente es privado. Este repositorio es una documentación técnica del proyecto con fines de portfolio.

---

*Desarrollado por [Isaac Román](https://github.com/IsaacRomanSerrano)*
