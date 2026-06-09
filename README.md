# SRCN — Sistema de Registro Criminal Nacional

**República de Guinea Ecuatorial · Ministerio del Interior · BP Technology · 2026**

> ⚠️ USO OFICIAL RESTRINGIDO — Acceso no autorizado prohibido

---

## ¿Qué es el SRCN?

El SRCN es una plataforma de tres niveles para la digitalización del registro criminal nacional de Guinea Ecuatorial, operada exclusivamente sobre intranet nacional segura, sin dependencia de internet público para las operaciones cotidianas.

**Problema que resuelve:** Un delincuente con orden de arresto activa en Bioko Norte puede ser detenido en Litoral (Bata) y liberado en minutos porque el agente no tiene acceso a los warrants de otras provincias. El SRCN elimina esta brecha — una consulta por DNI devuelve el estado de antecedentes y warrants nacionales en menos de 8 segundos.

---

## Repositorios

| Nodo | Nivel | Hardware | Repositorio |
|------|-------|----------|-------------|
| Comisaría / Estación Policial | Nivel 1 | Raspberry Pi 4 | [Angeal13/srcn-comisaria](https://github.com/Angeal13/srcn-comisaria) |
| Nodo Provincial | Nivel 2 | Mini-PC x86 | [Angeal13/srcn-nodo-provincial](https://github.com/Angeal13/srcn-nodo-provincial) |
| Servidor Central — Ministerio | Nivel 3 | Servidor Rack | [Angeal13/srcn-ministerio](https://github.com/Angeal13/srcn-ministerio) |
| Nodo Annobón (caso especial) | Nivel 2 | Mini-PC + Satélite | [Angeal13/srcn-annobon](https://github.com/Angeal13/srcn-annobon) |

---

## Arquitectura

```
Comisarías (SQLite · Pi 4)
    ↓  sync cada 30s · HMAC token
Nodos Provinciales (MySQL · x86) × 7
    ↓  bajo demanda · mTLS
Servidor Central (MySQL · Rack · Malabo)
    ↓  propagación paralela
Todas las provincias ← alertas de prófugos
```

### Flujo de verificación inter-provincial (< 8 segundos)

```
Agente introduce DNI en Bata
  → Comisaría → Nodo Litoral (sin resultado local)
    → Servidor Central (consulta índice nacional)
      → Nodo Bioko Norte (fetch expediente)
        → Warrant activo encontrado
      ← Expediente completo
    ← DETENER — warrant activo, provincia: BN
  ← 🚨 DETENER — OBAMA, P. | EXP-2025-00234
```

---

## Roles de acceso

| Rol | Permisos |
|-----|----------|
| `agente` | Registrar detenidos, consultar antecedentes |
| `inspector` | Agente + emitir warrants |
| `jefe` | Inspector + editar perfiles, autorizar liberaciones y transferencias |
| `superadmin` | Acceso total + gestión de usuarios y red |
| `readonly` | Solo lectura (auditores, Fiscalía) |

---

## Instalación rápida

```bash
# 1. Copiar y configurar variables de entorno
cp .env.template .env
nano .env

# 2. Instalar (requiere Ubuntu 24, ejecutar como root)
bash deploy/instalar.sh
```

### Variables críticas a configurar en `.env`

```env
SECRET_KEY=<64-char-random-string>        # python3 -c "import secrets; print(secrets.token_hex(32))"
STATION_CODE=COM-BN-001                   # Código único de esta estación
STATION_MODE=intranet_station             # comisaría: intranet_station
                                          # provincial: provincial_node
                                          # central: central_server
                                          # annobón: annobon_node
PROVINCIAL_NODE_URL=http://10.20.0.1:5000 # URL del nodo provincial (comisarías)
CENTRAL_SERVER_URL=http://10.0.0.1:5000   # URL del central (nodos provinciales)
SYNC_API_TOKEN=<shared-token>             # Mismo valor en todos los nodos
```

### Orden de despliegue

```
1. Servidor Central (Ministerio, Malabo)
2. Nodos Provinciales (uno por provincia)
3. Comisarías (una por estación)
4. Annobón (último — conectividad satelital)
```

---

## Stack tecnológico

| Componente | Tecnología |
|------------|------------|
| Backend | Python 3.11 · Flask 3 · Gunicorn |
| Base de datos comisaría | SQLite 3 |
| Base de datos provincial / central | MySQL 8.0 / MariaDB 10.11 |
| Proxy / TLS | Nginx · mTLS inter-provincial |
| Autenticación | Flask-Login · bcrypt · JWT · HMAC tokens |
| Sync scheduler | APScheduler |
| Frontend | Jinja2 · CSS propio (sin frameworks) |
| OS | Ubuntu Server 24.04 LTS |
| Hardware comisaría | Raspberry Pi 4 8GB |
| Red intranet | Ubiquiti airMAX AC · VLANs por sistema |

---

## Documentación visual

| Archivo | Descripción |
|---------|-------------|
| `srcn_dfd_mermaid.html` | Diagramas de flujo de datos (Mermaid JS) — 5 diagramas |
| `srcn_animated_flow.html` | DFD animado con partículas — flujo en tiempo real |
| `SRCN_Repositorios.html` | Página de repositorios con DFD animado integrado |

---

## Seguridad

- Toda comunicación entre nodos autenticada con token HMAC (`X-SRCN-Token`)
- Comunicación inter-provincial cifrada con **mTLS** (certificados por provincia)
- Sesiones con `session_protection = 'strong'`, expiración en 8 horas
- **Audit log inmutable** — cada consulta registra agente, terminal, timestamp, UUID
- Comisarías sin acceso a internet público — intranet exclusivamente
- Contraseñas hasheadas con bcrypt

---

## Estructura de directorios (por repo)

```
srcn-<nodo>/
├── app/
│   ├── models/models.py          # Sujeto, Detencion, Warrant, AlertaProfugo…
│   ├── routes/
│   │   ├── sujetos.py            # Perfiles criminales
│   │   ├── detenciones.py        # Booking / registro de detenciones
│   │   ├── warrants.py           # Órdenes de arresto + alertas
│   │   ├── transferencias.py     # Traslados inter-comisaría
│   │   ├── estadisticas.py       # Dashboard estadístico
│   │   ├── api_sync.py           # API de sincronización (HMAC)
│   │   ├── nodo_provincial.py    # Lógica provincial / central (no en comisaría)
│   │   ├── auth.py               # Login / logout / cambiar contraseña
│   │   ├── admin.py              # Gestión de usuarios
│   │   └── red.py                # Panel de red y sync
│   ├── templates/                # Jinja2 — diseño oscuro navy/cyan
│   ├── static/
│   │   ├── css/srcn.css          # Design system completo
│   │   └── js/srcn.js            # DNI lookup, autocomplete, confirmaciones
│   └── utils/
│       ├── intranet_sync.py      # Push en tiempo real al nodo provincial
│       └── sync_scheduler.py     # Sync periódico fallback (APScheduler)
├── config/settings.py            # Configuración por modo de despliegue
├── scripts/seed_db.py            # Seed inicial: provincias, comisaría, usuarios, delitos
├── deploy/
│   ├── instalar.sh               # Script de instalación Ubuntu 24
│   ├── srcn.service              # Systemd service
│   └── nginx                     # Config Nginx
├── .env.template                 # Variables de entorno a configurar
├── requirements.txt
├── run.py
└── gunicorn.conf.py
```

---

*SRCN · BP Technology · Guinea Ecuatorial · 2026*  
*Ministerio de Seguridad Nacional / Ministerio del Interior*
