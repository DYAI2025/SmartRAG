# SmartRAG Local Architecture Concept
## Permanente Linux-Anwendung mit lokaler LLM-Integration

**Version:** 1.0
**Datum:** 2025-11-22
**Status:** Konzept

---

## 1. Vision & Zielsetzung

SmartRAG soll als **permanenter Hintergrunddienst** auf Linux-Systemen laufen und als **zentrales Gehirn** für lokale LLM-Anwendungen dienen. Die Architektur ist auf vollständige Offline-Fähigkeit, Persistenz und Multi-User-Szenarien ausgelegt.

### Kernziele
- ✅ **Permanente Verfügbarkeit**: Systemd-Service, automatischer Start beim Booten
- ✅ **Vollständig offline**: Keine Cloud-Abhängigkeiten, alle Modelle lokal
- ✅ **Persistente Datenhaltung**: Alle Konversationen, Dokumente und Embeddings persistent
- ✅ **API-First**: REST/WebSocket API für Integration mit beliebigen LLM-Clients
- ✅ **Multi-User**: Unterstützung für mehrere Benutzer/Sessions mit Isolation
- ✅ **Ressourceneffizient**: Optimiert für Consumer-Hardware (8-16GB RAM)

---

## 2. Architektur-Übersicht

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
├─────────────────────────────────────────────────────────────────┤
│  Streamlit UI  │  CLI Tools  │  Custom Apps  │  Other LLMs     │
└────────┬────────────────┬─────────────┬──────────────┬──────────┘
         │                │             │              │
         └────────────────┴─────────────┴──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   API GATEWAY     │
                    │  (FastAPI/ASGI)   │
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐        ┌──────▼──────┐     ┌──────▼──────┐
    │  REST   │        │  WebSocket  │     │   GraphQL   │
    │   API   │        │  Streaming  │     │  (Future)   │
    └────┬────┘        └──────┬──────┘     └──────┬──────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  SERVICE LAYER    │
                    │                   │
                    │ - RAG Orchestrator│
                    │ - Session Manager │
                    │ - Auth Manager    │
                    │ - File Manager    │
                    │ - Query Processor │
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼─────┐      ┌───────▼───────┐    ┌──────▼──────┐
    │Document  │      │  Conversation │    │   Vector    │
    │Processor │      │   Manager     │    │   Search    │
    └────┬─────┘      └───────┬───────┘    └──────┬──────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  PERSISTENCE LAYER│
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐        ┌──────▼──────┐     ┌──────▼──────┐
    │ SQLite  │        │  ChromaDB   │     │  File Store │
    │ (Meta)  │        │ (Vectors)   │     │   (Blobs)   │
    └────┬────┘        └──────┬──────┘     └──────┬──────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   MODEL LAYER     │
                    └─────────┬─────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
    ┌────▼────┐        ┌──────▼──────┐     ┌──────▼──────┐
    │ Ollama  │        │  HuggingFace│     │   Whisper   │
    │(LLM+Emb)│        │  (BLIP/CLIP)│     │   (Audio)   │
    └─────────┘        └─────────────┘     └─────────────┘
```

---

## 3. Service-Architektur (Linux Daemon)

### 3.1 Systemd Service Definition

**Datei:** `/etc/systemd/system/smartrag.service`

```ini
[Unit]
Description=SmartRAG - Local AI Knowledge Base
Documentation=https://github.com/DYAI2025/SmartRAG
After=network.target
Wants=network.target

[Service]
Type=simple
User=smartrag
Group=smartrag
WorkingDirectory=/opt/smartrag

# Environment
Environment="SMARTRAG_MODE=daemon"
Environment="SMARTRAG_API_PORT=8765"
Environment="SMARTRAG_UI_PORT=8501"
EnvironmentFile=-/etc/smartrag/environment

# Execution
ExecStartPre=/opt/smartrag/scripts/pre-start.sh
ExecStart=/opt/smartrag/venv/bin/python -m smartrag.daemon
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/opt/smartrag/scripts/graceful-shutdown.sh

# Restart policy
Restart=always
RestartSec=10
StartLimitIntervalSec=200
StartLimitBurst=5

# Resource Limits
MemoryMax=12G
MemoryHigh=10G
CPUQuota=400%

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/smartrag /var/log/smartrag

[Install]
WantedBy=multi-user.target
```

### 3.2 Verzeichnisstruktur

```
/opt/smartrag/                     # Hauptinstallation
├── smartrag/                      # Python-Paket
│   ├── daemon.py                  # Hauptdienst (FastAPI App)
│   ├── api/                       # REST/WebSocket Endpoints
│   ├── services/                  # Business Logic
│   ├── models/                    # Datenmodelle
│   └── utils/                     # Utilities
├── scripts/
│   ├── pre-start.sh              # Systemchecks
│   ├── graceful-shutdown.sh      # Sauberes Herunterfahren
│   └── backup.sh                 # Backup-Skript
├── config.yaml                    # Konfiguration
└── venv/                          # Virtual Environment

/var/lib/smartrag/                 # Persistente Daten
├── database/
│   ├── smartrag.db               # SQLite Hauptdatenbank
│   └── sessions.db               # Session-Datenbank
├── vector_db/                     # ChromaDB Persistence
├── file_storage/                  # Hochgeladene Dateien
│   ├── documents/
│   ├── images/
│   └── audio/
├── models/                        # Ollama Models
│   ├── llama3.1-8b/
│   └── nomic-embed-text/
└── cache/                         # HuggingFace Cache

/var/log/smartrag/                 # Logs
├── daemon.log                     # Service Log
├── api.log                        # API Requests
├── rag.log                        # RAG Operations
└── error.log                      # Errors

/etc/smartrag/                     # Konfiguration
├── config.yaml                    # Systemkonfiguration
├── environment                    # Environment Variables
└── users.yaml                     # Benutzerverwaltung (optional)
```

### 3.3 Startup-Flow

```
1. Systemd startet smartrag.service
   ↓
2. pre-start.sh führt Checks aus:
   - Prüft Ollama-Server
   - Prüft Datenbank-Integrität
   - Prüft verfügbare Modelle
   - Prüft Festplattenplatz
   ↓
3. daemon.py startet:
   - Lädt Konfiguration
   - Initialisiert Datenbanken
   - Startet Ollama (falls nicht läuft)
   - Lädt Modelle in RAM (optional)
   - Startet FastAPI Server
   - Startet WebSocket Handler
   ↓
4. Health-Check Loop:
   - Überwacht Systemressourcen
   - Prüft Model-Verfügbarkeit
   - Loggt Metriken
   ↓
5. Bei SIGTERM/SIGINT:
   - Graceful Shutdown
   - Schließt offene Sessions
   - Flusht Datenbanken
   - Speichert Cache
```

---

## 4. API Layer (FastAPI)

### 4.1 REST API Endpoints

**Basis-URL:** `http://localhost:8765/api/v1`

#### Session Management
```python
POST   /sessions                    # Neue Session erstellen
GET    /sessions/{session_id}       # Session-Details
DELETE /sessions/{session_id}       # Session beenden
GET    /sessions                     # Alle Sessions auflisten
```

#### Document Management
```python
POST   /documents/upload            # Dokument hochladen
GET    /documents                    # Alle Dokumente
GET    /documents/{doc_id}          # Dokument-Details
DELETE /documents/{doc_id}          # Dokument löschen
POST   /documents/search            # Dokumente suchen
GET    /documents/{doc_id}/chunks   # Chunks eines Dokuments
```

#### RAG Operations
```python
POST   /query                       # RAG Query (synchron)
POST   /query/stream                # RAG Query (streaming)
POST   /embed                       # Text embedden
POST   /enrich                      # Text anreichern (Enrichment)
POST   /search                      # Semantische Suche
GET    /search/similar/{doc_id}     # Ähnliche Dokumente
```

#### Conversation Management
```python
POST   /conversations               # Neue Konversation
GET    /conversations               # Alle Konversationen
GET    /conversations/{conv_id}     # Konversation laden
POST   /conversations/{conv_id}/messages  # Nachricht hinzufügen
DELETE /conversations/{conv_id}     # Konversation löschen
```

#### System Management
```python
GET    /health                      # Health Check
GET    /metrics                     # System-Metriken
GET    /models                      # Verfügbare Modelle
POST   /admin/backup                # Backup erstellen
POST   /admin/optimize              # Datenbank optimieren
```

### 4.2 WebSocket Endpoints

**WebSocket-URL:** `ws://localhost:8765/ws/v1`

```python
/ws/query/{session_id}              # Streaming Queries
/ws/events/{session_id}             # Event Stream (Processing Status)
/ws/notifications                   # System-Benachrichtigungen
```

### 4.3 API Beispiele

#### Query mit Streaming
```python
import requests
import json

# Session erstellen
session = requests.post('http://localhost:8765/api/v1/sessions',
                       json={'user_id': 'user123'}).json()

# Streaming Query
with requests.post('http://localhost:8765/api/v1/query/stream',
                   json={
                       'query': 'Was steht in meinen Dokumenten über ML?',
                       'session_id': session['session_id'],
                       'top_k': 5
                   },
                   stream=True) as response:
    for line in response.iter_lines():
        if line:
            data = json.loads(line)
            print(data['chunk'], end='', flush=True)
```

#### WebSocket Query
```python
import asyncio
import websockets
import json

async def query():
    uri = "ws://localhost:8765/ws/v1/query/session123"
    async with websockets.connect(uri) as websocket:
        await websocket.send(json.dumps({
            'query': 'Erkläre mir RAG',
            'top_k': 3
        }))

        async for message in websocket:
            data = json.loads(message)
            if data['type'] == 'chunk':
                print(data['content'], end='', flush=True)
            elif data['type'] == 'done':
                print(f"\n\nQuellen: {len(data['sources'])}")
                break

asyncio.run(query())
```

---

## 5. Persistenz-Architektur

### 5.1 Erweiterte Datenbankstruktur

#### SQLite Schema (`smartrag.db`)

```sql
-- Benutzer (optional für Multi-User)
CREATE TABLE users (
    user_id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP,
    settings JSON
);

-- Sessions
CREATE TABLE sessions (
    session_id TEXT PRIMARY KEY,
    user_id TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP,
    context JSON,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Konversationen
CREATE TABLE conversations (
    conversation_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    title TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    metadata JSON,
    FOREIGN KEY (session_id) REFERENCES sessions(session_id) ON DELETE CASCADE
);

-- Nachrichten
CREATE TABLE messages (
    message_id TEXT PRIMARY KEY,
    conversation_id TEXT NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSON,
    tokens_used INTEGER,
    FOREIGN KEY (conversation_id) REFERENCES conversations(conversation_id) ON DELETE CASCADE
);

-- Dokumente (erweitert)
CREATE TABLE documents (
    document_id TEXT PRIMARY KEY,
    user_id TEXT,
    filename TEXT NOT NULL,
    file_type TEXT NOT NULL,
    file_size INTEGER NOT NULL,
    file_hash TEXT UNIQUE NOT NULL,
    storage_path TEXT NOT NULL,
    upload_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed BOOLEAN DEFAULT FALSE,
    metadata JSON,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Chunks (für Tracking)
CREATE TABLE document_chunks (
    chunk_id TEXT PRIMARY KEY,
    document_id TEXT NOT NULL,
    chunk_index INTEGER NOT NULL,
    chunk_text TEXT NOT NULL,
    chunk_metadata JSON,
    vector_id TEXT,  -- Referenz zu ChromaDB
    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE
);

-- Query Log (für Analytics)
CREATE TABLE query_log (
    query_id TEXT PRIMARY KEY,
    session_id TEXT,
    query_text TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    top_k INTEGER,
    response_time_ms INTEGER,
    sources_count INTEGER,
    FOREIGN KEY (session_id) REFERENCES sessions(session_id)
);

-- System Metrics
CREATE TABLE system_metrics (
    metric_id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metric_type TEXT NOT NULL,
    metric_value REAL NOT NULL,
    metadata JSON
);

-- Indizes für Performance
CREATE INDEX idx_messages_conversation ON messages(conversation_id, timestamp);
CREATE INDEX idx_documents_user ON documents(user_id, upload_time);
CREATE INDEX idx_chunks_document ON document_chunks(document_id);
CREATE INDEX idx_query_log_session ON query_log(session_id, timestamp);
CREATE INDEX idx_sessions_user ON sessions(user_id, last_active);
```

### 5.2 ChromaDB Struktur

```python
# Collection per User (Isolation)
collections = {
    'user_{user_id}_documents': {
        'embedding_function': 'nomic-embed-text',
        'metadata': {
            'dimension': 768,
            'distance_metric': 'cosine'
        }
    }
}

# Metadata-Schema für Chunks
chunk_metadata = {
    'document_id': str,        # Referenz zu SQLite
    'chunk_index': int,
    'filename': str,
    'file_type': str,
    'page_number': int,        # für PDFs
    'timestamp': str,
    'user_id': str,
    'custom_tags': List[str]
}
```

### 5.3 Backup-Strategie

```bash
# Automatisches Backup (täglich via cron)
0 3 * * * /opt/smartrag/scripts/backup.sh

# backup.sh
#!/bin/bash
BACKUP_DIR="/var/backups/smartrag/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# SQLite Backup
sqlite3 /var/lib/smartrag/database/smartrag.db ".backup '$BACKUP_DIR/smartrag.db'"

# ChromaDB Backup (Copy)
tar czf "$BACKUP_DIR/vector_db.tar.gz" -C /var/lib/smartrag vector_db/

# File Storage (Inkrementell mit rsync)
rsync -av --link-dest=/var/backups/smartrag/latest \
    /var/lib/smartrag/file_storage/ "$BACKUP_DIR/file_storage/"

# Symbolischer Link für neuestes Backup
ln -snf "$BACKUP_DIR" /var/backups/smartrag/latest

# Alte Backups löschen (>30 Tage)
find /var/backups/smartrag/ -mindepth 1 -type d -mtime +30 -exec rm -rf {} +
```

---

## 6. Session & Context Management

### 6.1 Session-Lifecycle

```python
class SessionManager:
    """Verwaltet Benutzer-Sessions mit Persistenz"""

    async def create_session(self, user_id: str = None) -> Session:
        """
        Erstellt neue Session mit:
        - Eindeutiger Session-ID
        - User-Context
        - Isolierter ChromaDB Collection
        - Konversationshistorie
        """
        session_id = generate_uuid()
        session = Session(
            session_id=session_id,
            user_id=user_id or 'anonymous',
            context={},
            created_at=datetime.now()
        )

        # Persistieren
        await self.db.insert_session(session)

        # ChromaDB Collection erstellen
        collection_name = f"user_{session.user_id}_documents"
        await self.vector_store.get_or_create_collection(collection_name)

        return session

    async def load_session(self, session_id: str) -> Session:
        """Lädt bestehende Session aus DB"""
        session = await self.db.get_session(session_id)
        if session:
            # Update last_active
            await self.db.update_session_activity(session_id)
        return session

    async def get_session_context(self, session_id: str) -> Dict:
        """
        Lädt vollständigen Session-Context:
        - Aktuelle Konversation
        - Letzte N Nachrichten
        - Verfügbare Dokumente
        - User-Präferenzen
        """
        conversation = await self.db.get_active_conversation(session_id)
        messages = await self.db.get_recent_messages(
            conversation.conversation_id,
            limit=10
        )
        documents = await self.db.get_session_documents(session_id)

        return {
            'conversation_id': conversation.conversation_id,
            'messages': messages,
            'documents': documents,
            'document_count': len(documents)
        }
```

### 6.2 Konversations-Persistenz

```python
class ConversationManager:
    """Verwaltet persistente Konversationshistorie"""

    async def add_message(
        self,
        conversation_id: str,
        role: str,
        content: str,
        metadata: Dict = None
    ) -> Message:
        """Fügt Nachricht zu Konversation hinzu"""
        message = Message(
            message_id=generate_uuid(),
            conversation_id=conversation_id,
            role=role,
            content=content,
            timestamp=datetime.now(),
            metadata=metadata or {}
        )

        await self.db.insert_message(message)
        return message

    async def get_conversation_history(
        self,
        conversation_id: str,
        limit: int = 50
    ) -> List[Message]:
        """Lädt Konversationshistorie aus DB"""
        return await self.db.get_messages(conversation_id, limit)

    async def export_conversation(
        self,
        conversation_id: str,
        format: str = 'json'
    ) -> str:
        """Exportiert Konversation (JSON/Markdown)"""
        messages = await self.get_conversation_history(conversation_id)

        if format == 'json':
            return json.dumps([m.dict() for m in messages], indent=2)
        elif format == 'markdown':
            lines = [f"# Conversation {conversation_id}\n"]
            for msg in messages:
                lines.append(f"## {msg.role.upper()} ({msg.timestamp})")
                lines.append(msg.content)
                lines.append("")
            return "\n".join(lines)
```

---

## 7. Offline-First Strategie

### 7.1 Modell-Caching

```python
class OfflineModelManager:
    """Stellt sicher, dass alle Modelle offline verfügbar sind"""

    REQUIRED_MODELS = {
        'ollama': [
            'llama3.1:8b',
            'nomic-embed-text'
        ],
        'huggingface': [
            'Salesforce/blip-image-captioning-base',
            'openai/clip-vit-base-patch32'
        ],
        'whisper': [
            'base'
        ]
    }

    async def check_offline_readiness(self) -> Dict[str, bool]:
        """Prüft ob alle Modelle lokal verfügbar sind"""
        status = {}

        # Ollama Models
        for model in self.REQUIRED_MODELS['ollama']:
            status[f'ollama:{model}'] = await self._check_ollama_model(model)

        # HuggingFace Models
        for model in self.REQUIRED_MODELS['huggingface']:
            status[f'hf:{model}'] = self._check_hf_cache(model)

        # Whisper
        status['whisper:base'] = self._check_whisper_model('base')

        return status

    async def download_missing_models(self):
        """Lädt fehlende Modelle herunter"""
        status = await self.check_offline_readiness()

        for model, available in status.items():
            if not available:
                logger.info(f"Downloading {model}...")
                await self._download_model(model)
```

### 7.2 Graceful Degradation

```python
class OfflineCapabilities:
    """Definiert Funktionen ohne Netzwerk"""

    OFFLINE_FEATURES = {
        'document_upload': True,
        'text_processing': True,
        'image_ocr': True,
        'image_captioning': True,  # BLIP lokal
        'audio_transcription': True,  # Whisper lokal
        'rag_query': True,
        'embedding_generation': True,
        'llm_inference': True,

        # Eingeschränkt ohne Netz
        'external_web_search': False,
        'model_download': False,
        'remote_backup': False
    }

    def get_available_features(self, online: bool) -> List[str]:
        """Gibt verfügbare Features zurück"""
        if online:
            return list(self.OFFLINE_FEATURES.keys())
        else:
            return [k for k, v in self.OFFLINE_FEATURES.items() if v]
```

---

## 8. Integration mit lokalen LLMs

### 8.1 SmartRAG als LLM-Backend

```python
class LLMBrainInterface:
    """
    SmartRAG als 'Gehirn' für andere LLM-Anwendungen.
    Stellt Kontext-Anreicherung via API bereit.
    """

    async def enrich_prompt(
        self,
        query: str,
        session_id: str,
        top_k: int = 5
    ) -> Dict[str, Any]:
        """
        Reichert Prompt mit relevantem Kontext an

        Returns:
            {
                'enhanced_prompt': str,  # Anreicherter Prompt
                'sources': List[Source],  # Quelldokumente
                'context': str,  # Extrahierter Context
                'metadata': Dict  # Zusätzliche Metadaten
            }
        """
        # Semantische Suche
        results = await self.rag_system.search(
            query=query,
            session_id=session_id,
            top_k=top_k
        )

        # Context zusammenbauen
        context_parts = []
        for i, result in enumerate(results, 1):
            context_parts.append(
                f"[Quelle {i}] {result.metadata['filename']}:\n"
                f"{result.content}\n"
            )

        context = "\n".join(context_parts)

        # Enhanced Prompt
        enhanced_prompt = f"""Kontext aus Wissensdatenbank:
{context}

Benutzeranfrage: {query}

Beantworte die Anfrage basierend auf dem bereitgestellten Kontext."""

        return {
            'enhanced_prompt': enhanced_prompt,
            'sources': results,
            'context': context,
            'metadata': {
                'sources_count': len(results),
                'total_tokens': len(enhanced_prompt.split())
            }
        }

    async def generate_with_context(
        self,
        query: str,
        session_id: str,
        stream: bool = False
    ) -> Union[str, AsyncIterator[str]]:
        """
        Vollständige RAG-Pipeline:
        1. Context aus Dokumenten holen
        2. Prompt anreichern
        3. LLM-Antwort generieren
        """
        # Kontext anreichern
        enriched = await self.enrich_prompt(query, session_id)

        # LLM generieren
        if stream:
            return self.llm.generate_streaming(enriched['enhanced_prompt'])
        else:
            return await self.llm.generate(enriched['enhanced_prompt'])
```

### 8.2 Verwendung durch externe LLMs

```python
# Beispiel: Integration in eine andere LLM-Anwendung
import requests

class MyCustomLLMApp:
    def __init__(self):
        self.smartrag_api = "http://localhost:8765/api/v1"

    def chat(self, user_input: str, session_id: str):
        """Chat-Funktion mit SmartRAG-Kontext"""

        # 1. Kontext von SmartRAG holen
        response = requests.post(
            f"{self.smartrag_api}/enrich",
            json={
                'query': user_input,
                'session_id': session_id,
                'top_k': 5
            }
        )
        enriched = response.json()

        # 2. Optional: Eigenes LLM verwenden statt Ollama
        # my_llm_response = my_llm.generate(enriched['enhanced_prompt'])

        # 3. Oder SmartRAG's LLM nutzen
        response = requests.post(
            f"{self.smartrag_api}/query",
            json={
                'query': user_input,
                'session_id': session_id
            }
        )

        return response.json()
```

### 8.3 Plugin-System (Zukunft)

```python
# Plugin-Interface für externe LLM-Engines
class LLMPluginInterface:
    """
    Erlaubt Integration beliebiger LLM-Backends
    (Ollama, llama.cpp, Huggingface Transformers, etc.)
    """

    def register_llm_backend(
        self,
        name: str,
        backend: LLMBackend
    ):
        """Registriert neues LLM-Backend"""
        self.backends[name] = backend

    async def query_with_backend(
        self,
        query: str,
        backend_name: str,
        session_id: str
    ):
        """Führt Query mit spezifischem Backend aus"""
        backend = self.backends.get(backend_name)
        if not backend:
            raise ValueError(f"Backend {backend_name} nicht gefunden")

        # Kontext von SmartRAG
        context = await self.enrich_prompt(query, session_id)

        # Generierung mit gewähltem Backend
        return await backend.generate(context['enhanced_prompt'])
```

---

## 9. Deployment & Installation

### 9.1 Installation Script

```bash
#!/bin/bash
# install-smartrag-daemon.sh

set -e

echo "=== SmartRAG Daemon Installation ==="

# Systemvoraussetzungen prüfen
check_requirements() {
    echo "Prüfe Systemvoraussetzungen..."

    # RAM
    total_ram=$(free -g | awk '/^Mem:/{print $2}')
    if [ "$total_ram" -lt 8 ]; then
        echo "WARNUNG: Mindestens 8GB RAM empfohlen (gefunden: ${total_ram}GB)"
    fi

    # Disk Space
    available_space=$(df -BG /var/lib | awk 'NR==2 {print $4}' | sed 's/G//')
    if [ "$available_space" -lt 20 ]; then
        echo "FEHLER: Mindestens 20GB freier Speicher benötigt"
        exit 1
    fi

    # Python
    if ! command -v python3.10 &> /dev/null; then
        echo "FEHLER: Python 3.10+ nicht gefunden"
        exit 1
    fi
}

# Benutzer und Verzeichnisse erstellen
setup_user() {
    echo "Erstelle smartrag Benutzer..."
    sudo useradd -r -s /bin/false -d /opt/smartrag smartrag || true

    echo "Erstelle Verzeichnisse..."
    sudo mkdir -p /opt/smartrag
    sudo mkdir -p /var/lib/smartrag/{database,vector_db,file_storage,models,cache}
    sudo mkdir -p /var/log/smartrag
    sudo mkdir -p /etc/smartrag

    sudo chown -R smartrag:smartrag /opt/smartrag
    sudo chown -R smartrag:smartrag /var/lib/smartrag
    sudo chown -R smartrag:smartrag /var/log/smartrag
}

# Python Environment
setup_python() {
    echo "Erstelle Python Virtual Environment..."
    cd /opt/smartrag
    sudo -u smartrag python3 -m venv venv
    sudo -u smartrag ./venv/bin/pip install --upgrade pip
    sudo -u smartrag ./venv/bin/pip install -r requirements.txt
}

# Ollama installieren
setup_ollama() {
    echo "Installiere Ollama..."
    if ! command -v ollama &> /dev/null; then
        curl -fsSL https://ollama.ai/install.sh | sh
    fi

    echo "Lade Modelle..."
    ollama pull llama3.1:8b
    ollama pull nomic-embed-text
}

# Konfiguration
setup_config() {
    echo "Erstelle Konfiguration..."
    sudo cp config.yaml /etc/smartrag/config.yaml
    sudo cp environment.example /etc/smartrag/environment
    sudo chown smartrag:smartrag /etc/smartrag/*
}

# Systemd Service
setup_service() {
    echo "Installiere Systemd Service..."
    sudo cp systemd/smartrag.service /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable smartrag.service
}

# Ausführen
check_requirements
setup_user
setup_python
setup_ollama
setup_config
setup_service

echo ""
echo "=== Installation abgeschlossen ==="
echo ""
echo "Nächste Schritte:"
echo "  1. Konfiguration anpassen: /etc/smartrag/config.yaml"
echo "  2. Service starten: sudo systemctl start smartrag"
echo "  3. Status prüfen: sudo systemctl status smartrag"
echo "  4. Logs anzeigen: sudo journalctl -u smartrag -f"
echo ""
echo "API verfügbar unter: http://localhost:8765"
echo "UI verfügbar unter: http://localhost:8501"
```

### 9.2 Docker-Alternative (Hybrid)

```yaml
# docker-compose.daemon.yml
version: '3.8'

services:
  smartrag-daemon:
    build:
      context: .
      dockerfile: docker/Dockerfile.daemon
    container_name: smartrag-daemon
    restart: unless-stopped

    ports:
      - "8765:8765"  # API
      - "8501:8501"  # UI (optional)

    volumes:
      # Persistente Daten
      - ./data/database:/var/lib/smartrag/database
      - ./data/vector_db:/var/lib/smartrag/vector_db
      - ./data/file_storage:/var/lib/smartrag/file_storage
      - ./data/models:/var/lib/smartrag/models

      # Konfiguration
      - ./config.yaml:/etc/smartrag/config.yaml:ro

      # Logs
      - ./logs:/var/log/smartrag

    environment:
      - SMARTRAG_MODE=daemon
      - SMARTRAG_API_PORT=8765
      - SMARTRAG_UI_PORT=8501

    deploy:
      resources:
        limits:
          memory: 12G
        reservations:
          memory: 8G

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8765/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

---

## 10. Monitoring & Maintenance

### 10.1 Health Checks

```python
class HealthMonitor:
    """Überwacht System-Health"""

    async def get_health_status(self) -> Dict:
        """Vollständiger Health-Check"""
        return {
            'status': 'healthy' | 'degraded' | 'unhealthy',
            'timestamp': datetime.now().isoformat(),
            'components': {
                'database': await self._check_database(),
                'vector_store': await self._check_vector_store(),
                'ollama': await self._check_ollama(),
                'models': await self._check_models(),
                'disk_space': self._check_disk_space(),
                'memory': self._check_memory()
            },
            'metrics': {
                'active_sessions': await self._count_active_sessions(),
                'total_documents': await self._count_documents(),
                'total_queries': await self._count_queries_today()
            }
        }

    def _check_disk_space(self) -> Dict:
        """Prüft verfügbaren Speicherplatz"""
        statvfs = os.statvfs('/var/lib/smartrag')
        free_gb = (statvfs.f_bavail * statvfs.f_frsize) / (1024**3)

        return {
            'status': 'ok' if free_gb > 5 else 'warning',
            'free_gb': round(free_gb, 2),
            'warning_threshold_gb': 5
        }
```

### 10.2 Metriken & Logging

```python
# Prometheus-kompatible Metriken
from prometheus_client import Counter, Histogram, Gauge

# Metriken definieren
query_counter = Counter('smartrag_queries_total', 'Total number of queries')
query_duration = Histogram('smartrag_query_duration_seconds', 'Query duration')
active_sessions = Gauge('smartrag_active_sessions', 'Number of active sessions')
document_count = Gauge('smartrag_documents_total', 'Total documents in system')

# Logging-Konfiguration
LOGGING_CONFIG = {
    'version': 1,
    'handlers': {
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/smartrag/daemon.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5,
            'formatter': 'detailed'
        },
        'error_file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/smartrag/error.log',
            'maxBytes': 10485760,
            'backupCount': 5,
            'level': 'ERROR',
            'formatter': 'detailed'
        }
    },
    'formatters': {
        'detailed': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        }
    },
    'root': {
        'level': 'INFO',
        'handlers': ['file', 'error_file']
    }
}
```

### 10.3 Wartungs-Skripte

```bash
# /opt/smartrag/scripts/maintenance.sh
#!/bin/bash

echo "=== SmartRAG Wartung ==="

# SQLite Vacuum
echo "Optimiere Datenbank..."
sqlite3 /var/lib/smartrag/database/smartrag.db "VACUUM;"
sqlite3 /var/lib/smartrag/database/smartrag.db "ANALYZE;"

# ChromaDB Cleanup (alte Collections)
echo "Bereinige Vector Store..."
# TODO: Python-Skript für ChromaDB Cleanup

# Log Rotation (falls nicht automatisch)
echo "Rotiere Logs..."
find /var/log/smartrag/ -name "*.log" -size +100M -exec gzip {} \;
find /var/log/smartrag/ -name "*.log.gz" -mtime +30 -delete

# Disk Usage Report
echo "Speichernutzung:"
du -sh /var/lib/smartrag/*

echo "Wartung abgeschlossen."
```

---

## 11. Security-Überlegungen

### 11.1 Zugriffskontrolle

```python
class SecurityManager:
    """Grundlegende Sicherheitsfunktionen"""

    def __init__(self):
        self.api_keys = self._load_api_keys()

    async def authenticate_request(
        self,
        api_key: str = None,
        ip_address: str = None
    ) -> bool:
        """
        Authentifizierung via API-Key oder IP-Whitelist
        """
        if api_key:
            return api_key in self.api_keys

        if ip_address:
            # Nur localhost erlauben (für lokalen Betrieb)
            return ip_address in ['127.0.0.1', '::1', 'localhost']

        return False

    def sanitize_filename(self, filename: str) -> str:
        """Verhindert Path-Traversal"""
        return os.path.basename(filename)

    async def check_file_safety(self, file_path: str) -> bool:
        """
        Prüft hochgeladene Dateien auf:
        - Gültige MIME-Types
        - Maximale Dateigröße
        - Keine ausführbaren Dateien
        """
        # TODO: Implementierung
        pass
```

### 11.2 Netzwerk-Isolation

```ini
# systemd Service mit Netzwerk-Beschränkungen
[Service]
# Nur localhost-Zugriff
IPAddressAllow=localhost
IPAddressDeny=any

# Firewall
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
```

---

## 12. Roadmap & Zukunfts-Features

### Phase 1: Foundation (Wochen 1-4)
- ✅ Systemd Service Implementation
- ✅ FastAPI REST API
- ✅ Erweiterte Persistenz (SQLite-Schema)
- ✅ Session Management
- ⬜ WebSocket Streaming

### Phase 2: Core Features (Wochen 5-8)
- ⬜ Multi-User Support
- ⬜ Konversations-Management
- ⬜ Advanced Query API
- ⬜ Backup/Restore System
- ⬜ Health Monitoring

### Phase 3: Integration (Wochen 9-12)
- ⬜ Plugin-System für LLM-Backends
- ⬜ CLI-Tools
- ⬜ Python SDK
- ⬜ Prometheus Metriken
- ⬜ Grafana Dashboards

### Phase 4: Enhancement (Zukunft)
- ⬜ GraphQL API
- ⬜ Federated Search (mehrere RAG-Instanzen)
- ⬜ Advanced Analytics
- ⬜ Kubernetes-Support
- ⬜ Distributed Vector Store

---

## 13. Beispiel-Nutzungsszenarien

### Szenario 1: Lokaler Dokumenten-Assistent
```bash
# SmartRAG läuft als Service
systemctl status smartrag

# Dokumente hochladen via CLI
smartrag-cli upload ~/Documents/papers/*.pdf

# Query via CLI
smartrag-cli query "Was sind die wichtigsten Erkenntnisse aus den Papers?"

# Oder via API
curl -X POST http://localhost:8765/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Erkenntnisse aus Papers", "session_id": "user123"}'
```

### Szenario 2: Integration in IDE
```python
# VS Code Extension verwendet SmartRAG als Backend
class CodeContextProvider:
    def __init__(self):
        self.smartrag = SmartRAGClient("http://localhost:8765")

    async def get_code_context(self, query: str):
        # SmartRAG durchsucht alle Code-Dokumente
        response = await self.smartrag.query(
            query=query,
            session_id="vscode_session",
            filter={'file_type': ['py', 'js', 'md']}
        )
        return response.answer
```

### Szenario 3: Persönlicher Wissens-Hub
```bash
# Automatisches Einlesen neuer Dateien
inotifywait -m ~/Documents -e create -e moved_to |
while read path action file; do
    smartrag-cli upload "$path$file"
done

# Tägliches Backup
0 2 * * * /opt/smartrag/scripts/backup.sh

# Wöchentliche Reports
0 9 * * 1 smartrag-cli report --last-week | mail -s "SmartRAG Weekly" user@localhost
```

---

## 14. Zusammenfassung

### Kernaspekte der Architektur

1. **Permanenter Dienst**: Systemd-Integration für 24/7 Verfügbarkeit
2. **Vollständig Offline**: Alle Modelle lokal, keine Cloud-Abhängigkeiten
3. **API-First**: REST + WebSocket für vielseitige Integration
4. **Persistente Datenhaltung**: SQLite + ChromaDB mit Backup-Strategie
5. **Multi-User Ready**: Session-Management mit Benutzer-Isolation
6. **Ressourcen-Effizient**: Optimiert für Consumer-Hardware
7. **LLM-Gehirn**: Kontext-Anreicherung für beliebige LLM-Anwendungen

### Deployment-Optionen

| Option | Vorteile | Nachteile |
|--------|----------|-----------|
| **Native Systemd** | Maximale Performance, direkte System-Integration | Komplexere Installation |
| **Docker** | Einfaches Setup, Isolation | Leichter Overhead |
| **Hybrid** | Balance zwischen beidem | Erfordert beide Kenntnisse |

### Nächste Schritte

1. **Prototyp entwickeln**: Minimale FastAPI-Implementierung
2. **Schema migrieren**: Erweiterte SQLite-Struktur
3. **Session-Management**: Implementierung & Testing
4. **Systemd-Integration**: Service-Scripts und Installation
5. **Testing**: End-to-End Tests mit verschiedenen Clients

---

**Dokumentversion:** 1.0
**Letztes Update:** 2025-11-22
**Maintainer:** DYAI2025/SmartRAG Team
