# SmartRAG Local Architecture - Schnellübersicht

## Vision in einem Satz
**SmartRAG als permanenter Linux-Systemdienst, der lokalen LLMs als persistentes Gedächtnis und Kontext-Engine dient.**

---

## Kern-Architektur

```
┌──────────────────────────────────────────────────┐
│              CLIENT APPLICATIONS                  │
│  (Streamlit, CLI, Custom Apps, Other LLMs)       │
└───────────────────┬──────────────────────────────┘
                    │
        ┌───────────▼───────────┐
        │   FastAPI REST API    │  ← Port 8765
        │   WebSocket Streaming │
        └───────────┬───────────┘
                    │
        ┌───────────▼───────────┐
        │   SERVICE LAYER       │
        │  - Session Manager    │
        │  - RAG Orchestrator   │
        │  - Conversation Mgr   │
        └───────────┬───────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───▼────┐    ┌────▼─────┐   ┌────▼────┐
│ SQLite │    │ ChromaDB │   │ Ollama  │
│ (Meta) │    │ (Vector) │   │ (LLM)   │
└────────┘    └──────────┘   └─────────┘

         Persistent in /var/lib/smartrag/
```

---

## Hauptkomponenten

### 1. **Systemd Service**
- Automatischer Start beim Booten
- Graceful Shutdown bei SIGTERM
- Resource Limits (Memory, CPU)
- Automatisches Restart bei Crashes

### 2. **FastAPI Server** (Port 8765)
- REST API für CRUD-Operationen
- WebSocket für Streaming
- OpenAPI/Swagger Dokumentation
- Session-basierte Authentifizierung

### 3. **Persistenz-Layer**
```
/var/lib/smartrag/
├── database/
│   └── smartrag.db         # SQLite: Sessions, Messages, Docs
├── vector_db/              # ChromaDB: Embeddings
├── file_storage/           # Original-Dateien
└── models/                 # Ollama Model Cache
```

### 4. **Multi-User Support**
- User-isolierte Sessions
- Separate ChromaDB Collections pro User
- Konversations-Historie persistent
- Query-Tracking & Analytics

---

## API-Übersicht

### Basis-URL
```
http://localhost:8765/api/v1
```

### Wichtigste Endpoints

#### Sessions
```bash
POST   /sessions                 # Neue Session
GET    /sessions/{id}            # Session-Details
DELETE /sessions/{id}            # Session beenden
```

#### Dokumente
```bash
POST   /documents/upload         # Upload + Processing
GET    /documents                # Liste aller Docs
GET    /documents/{id}           # Doc-Details
DELETE /documents/{id}           # Doc löschen
```

#### RAG Queries
```bash
POST   /query                    # Synchron
POST   /query/stream             # Streaming
```

#### Konversationen
```bash
POST   /conversations            # Neue Konversation
GET    /conversations/{id}       # Lade History
POST   /conversations/{id}/messages  # Nachricht hinzufügen
```

#### System
```bash
GET    /health                   # Health-Check
GET    /metrics                  # Prometheus Metrics
POST   /admin/backup             # Backup erstellen
```

---

## Verwendungsbeispiele

### CLI
```bash
# Service starten
sudo systemctl start smartrag

# Dokument hochladen
smartrag-cli upload ~/document.pdf

# Query ausführen
smartrag-cli query "Was steht im Dokument?"

# Status prüfen
smartrag-cli status
```

### Python SDK
```python
import requests

# Session erstellen
session = requests.post('http://localhost:8765/api/v1/sessions').json()

# Dokument hochladen
with open('doc.pdf', 'rb') as f:
    requests.post(
        'http://localhost:8765/api/v1/documents/upload',
        files={'file': f},
        data={'session_id': session['session_id']}
    )

# RAG Query
response = requests.post(
    'http://localhost:8765/api/v1/query',
    json={
        'query': 'Fasse das Dokument zusammen',
        'session_id': session['session_id']
    }
)
print(response.json()['answer'])
```

### Integration in eigene LLM-App
```python
class MyLLMApp:
    def __init__(self):
        self.smartrag = "http://localhost:8765/api/v1"
        self.session_id = self._create_session()

    def chat(self, message: str):
        # SmartRAG als Kontext-Engine nutzen
        response = requests.post(
            f"{self.smartrag}/query",
            json={
                'query': message,
                'session_id': self.session_id,
                'top_k': 5
            }
        )
        return response.json()['answer']
```

---

## Deployment-Optionen

### Option 1: Native Systemd (Empfohlen für Produktion)
```bash
# Installation
sudo ./scripts/install-smartrag-daemon.sh

# Start
sudo systemctl start smartrag
sudo systemctl enable smartrag  # Auto-start

# Logs
sudo journalctl -u smartrag -f
```

**Vorteile**: Maximale Performance, direkte System-Integration
**Nachteile**: Komplexere Installation

---

### Option 2: Docker
```bash
# Start
docker-compose -f docker/docker-compose.daemon.yml up -d

# Logs
docker logs smartrag-daemon -f
```

**Vorteile**: Einfaches Setup, Isolation
**Nachteile**: Leichter Overhead

---

## Verzeichnisstruktur

```
/opt/smartrag/                   # Installation
├── smartrag/                    # Python Package
│   ├── daemon/                  # Daemon Core
│   ├── api/                     # FastAPI App
│   ├── services/                # Business Logic
│   ├── persistence/             # Database Layer
│   └── multimodal_rag/          # RAG Core (existing)
├── scripts/                     # Maintenance Scripts
└── venv/                        # Virtual Environment

/var/lib/smartrag/               # Persistent Data
├── database/                    # SQLite DBs
├── vector_db/                   # ChromaDB
├── file_storage/                # Uploaded Files
└── models/                      # Model Cache

/var/log/smartrag/               # Logs
├── daemon.log
├── api.log
└── error.log

/etc/smartrag/                   # Configuration
├── config.yaml
└── environment
```

---

## Offline-Fähigkeit

### Vollständig Offline Möglich
✅ Dokument-Upload & Processing
✅ Text-Extraktion (PDF, DOCX, etc.)
✅ OCR (Tesseract)
✅ Image Captioning (BLIP - lokal)
✅ Audio Transcription (Whisper - lokal)
✅ Embedding-Generation (Nomic via Ollama)
✅ LLM-Inference (Llama 3.1 via Ollama)
✅ RAG Queries
✅ Konversations-Historie

### Benötigt Internet
❌ Model-Downloads (einmalig)
❌ System-Updates
❌ Remote-Backup (optional)

**Strategie**: Alle Modelle beim Setup herunterladen, dann komplett offline nutzbar.

---

## Resource Requirements

### Minimum
- **RAM**: 8GB
- **Disk**: 50GB frei
- **CPU**: 4 Cores
- **OS**: Linux (Ubuntu 20.04+)

### Empfohlen
- **RAM**: 16GB
- **Disk**: 100GB+ SSD
- **CPU**: 8 Cores
- **OS**: Ubuntu 22.04 LTS

### Speichernutzung
```
/var/lib/smartrag/
├── models/          ~10GB   (Ollama Models)
├── vector_db/       ~1GB    (pro 10k Dokumente)
├── database/        ~100MB  (pro 1k Konversationen)
└── file_storage/    variabel (Original-Dateien)
```

---

## Performance-Ziele

| Metrik | Ziel | Gemessen bei |
|--------|------|--------------|
| Query Latency (p95) | <2s | 10k Dokumente |
| API Response Time | <100ms | Ohne LLM-Inference |
| Document Processing | <30s | Durchschnittliches PDF (20 Seiten) |
| Concurrent Users | 10+ | Ohne Performance-Degradation |
| Memory Usage | <8GB | Normal Load |
| Uptime | >99.9% | Systemd mit Auto-Restart |

---

## Security Features

### Zugriffskontrolle
- Standardmäßig nur localhost (127.0.0.1)
- Optional: API-Key Authentication
- Optional: IP-Whitelist

### System-Isolation
- Dedizierter User (`smartrag`)
- Restricted File Permissions
- Systemd Security Features:
  - `NoNewPrivileges=true`
  - `ProtectSystem=strict`
  - `PrivateTmp=true`

### Daten-Sicherheit
- Tägliche automatische Backups
- SQLite WAL-Mode (Crash-Safety)
- Checksummen für Dateien

---

## Backup & Restore

### Automatisches Backup (täglich)
```bash
# Cron-Job (3 Uhr nachts)
0 3 * * * /opt/smartrag/scripts/backup.sh
```

### Backup-Inhalt
- SQLite Datenbank (Online Backup)
- ChromaDB (tar.gz)
- File Storage (rsync inkrementell)

### Restore
```bash
# Vollständiger Restore
sudo /opt/smartrag/scripts/restore.sh /var/backups/smartrag/20251122
```

### Retention
- Tägliche Backups: 7 Tage
- Wöchentliche Backups: 4 Wochen
- Monatliche Backups: 12 Monate

---

## Monitoring

### Health-Check
```bash
curl http://localhost:8765/health

{
  "status": "healthy",
  "components": {
    "database": "ok",
    "vector_store": "ok",
    "ollama": "ok",
    "disk_space": "ok"
  },
  "metrics": {
    "active_sessions": 5,
    "total_documents": 234,
    "queries_today": 156
  }
}
```

### Prometheus Metrics (Optional)
```bash
# Metrics Endpoint
curl http://localhost:8765/metrics

# Beispiel-Metriken
smartrag_queries_total 1234
smartrag_query_duration_seconds_bucket{le="1.0"} 890
smartrag_active_sessions 5
smartrag_documents_total 234
```

### Logs
```bash
# Echtzeit-Logs
sudo journalctl -u smartrag -f

# Letzte 100 Zeilen
sudo journalctl -u smartrag -n 100

# Fehler-Logs
sudo journalctl -u smartrag -p err
```

---

## Integration-Szenarien

### 1. Als Backend für Streamlit UI
```python
# Streamlit nutzt API statt direktem RAG
import streamlit as st
import requests

api = "http://localhost:8765/api/v1"
session_id = st.session_state.get('session_id')

if not session_id:
    response = requests.post(f"{api}/sessions")
    session_id = response.json()['session_id']
    st.session_state['session_id'] = session_id

# Upload
uploaded = st.file_uploader("Upload")
if uploaded:
    requests.post(
        f"{api}/documents/upload",
        files={'file': uploaded},
        data={'session_id': session_id}
    )

# Query
query = st.chat_input("Ask...")
if query:
    response = requests.post(
        f"{api}/query",
        json={'query': query, 'session_id': session_id}
    )
    st.write(response.json()['answer'])
```

### 2. Als Kontext-Engine für lokale LLMs
```python
# llama.cpp App nutzt SmartRAG für Kontext
from llama_cpp import Llama

llm = Llama(model_path="./models/llama-3.1-8b.gguf")

def chat_with_context(user_input: str):
    # Hole Kontext von SmartRAG
    context_response = requests.post(
        'http://localhost:8765/api/v1/enrich',
        json={'query': user_input, 'session_id': session_id}
    )
    enhanced_prompt = context_response.json()['enhanced_prompt']

    # Eigenes LLM für Generierung
    return llm(enhanced_prompt)
```

### 3. Als Knowledge Base für Agents
```python
# LangChain Agent nutzt SmartRAG als Tool
from langchain.agents import Tool

def smartrag_search(query: str) -> str:
    response = requests.post(
        'http://localhost:8765/api/v1/query',
        json={'query': query, 'session_id': session_id}
    )
    return response.json()['answer']

tools = [
    Tool(
        name="SmartRAG Knowledge Base",
        func=smartrag_search,
        description="Suche in der lokalen Wissensdatenbank"
    )
]
```

---

## Troubleshooting

### Service startet nicht
```bash
# Logs prüfen
sudo journalctl -u smartrag -n 50

# Häufige Probleme:
# 1. Port bereits belegt
sudo lsof -i :8765

# 2. Ollama läuft nicht
sudo systemctl status ollama

# 3. Berechtigungen
ls -la /var/lib/smartrag/
```

### Query langsam
```bash
# Datenbank optimieren
smartrag-cli optimize

# Cache-Status prüfen
curl http://localhost:8765/health | jq '.components'

# Memory-Nutzung prüfen
sudo systemctl status smartrag
```

### Disk voll
```bash
# Alte Backups löschen
find /var/backups/smartrag/ -mindepth 1 -type f -mtime +30 -delete

# Logs rotieren
sudo journalctl --vacuum-time=7d

# Ungenutzte Dokumente archivieren
smartrag-cli cleanup --older-than 90d
```

---

## Weiterführende Dokumentation

- **[LOCAL_ARCHITECTURE_CONCEPT.md](./LOCAL_ARCHITECTURE_CONCEPT.md)**: Vollständiges Architektur-Konzept
- **[IMPLEMENTATION_ROADMAP.md](./IMPLEMENTATION_ROADMAP.md)**: Detaillierte Implementierungs-Roadmap
- **[README.md](../README.md)**: Projekt-Hauptdokumentation

---

## Quick Start

```bash
# 1. Repository klonen
git clone https://github.com/DYAI2025/SmartRAG.git
cd SmartRAG

# 2. Installation
sudo ./scripts/install-smartrag-daemon.sh

# 3. Service starten
sudo systemctl start smartrag

# 4. Status prüfen
curl http://localhost:8765/health

# 5. Erstes Dokument hochladen
smartrag-cli upload ~/document.pdf

# 6. Erste Query
smartrag-cli query "Was ist der Inhalt?"
```

---

**Version**: 1.0
**Letztes Update**: 2025-11-22
**Status**: Konzept-Phase
