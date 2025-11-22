# SmartRAG Local Architecture - Implementierungs-Roadmap

## Übersicht

Diese Roadmap beschreibt die schrittweise Implementierung von SmartRAG als permanente lokale Linux-Anwendung.

---

## Phase 1: Foundation Layer (Woche 1-2)

### 1.1 Projekt-Restrukturierung

**Ziel**: Trennung von UI und Daemon-Backend

```
smartrag/
├── daemon/                    # NEU: Daemon-Modul
│   ├── __init__.py
│   ├── main.py               # Haupteinstiegspunkt
│   ├── config.py             # Daemon-spezifische Config
│   └── lifecycle.py          # Startup/Shutdown Logic
├── api/                       # NEU: API Layer
│   ├── __init__.py
│   ├── app.py                # FastAPI Application
│   ├── routes/
│   │   ├── sessions.py
│   │   ├── documents.py
│   │   ├── queries.py
│   │   └── admin.py
│   ├── websocket/
│   │   └── handlers.py
│   └── models/               # Pydantic API Models
│       ├── requests.py
│       └── responses.py
├── services/                  # NEU: Business Logic
│   ├── session_manager.py
│   ├── conversation_manager.py
│   ├── document_service.py
│   └── rag_service.py
├── persistence/               # NEU: Datenbank Layer
│   ├── __init__.py
│   ├── database.py           # SQLite Connection
│   ├── models.py             # SQLAlchemy Models
│   └── migrations/           # Alembic Migrations
└── multimodal_rag/           # BESTEHEND: RAG Core
    └── ...                   # Bleibt größtenteils unverändert
```

**Tasks**:
- [ ] Neue Verzeichnisstruktur anlegen
- [ ] `daemon/main.py` mit Basic FastAPI App
- [ ] Dependency Injection Setup (FastAPI Depends)
- [ ] Config-Loading aus `config_schema.py` anpassen

**Acceptance Criteria**:
- FastAPI-App startet auf Port 8765
- Health-Endpoint `/health` funktioniert
- Bestehende RAG-Funktionalität unverändert

---

### 1.2 Datenbank-Schema Implementation

**Ziel**: Erweiterte Persistenz für Sessions, Conversations, Messages

**Tasks**:
- [ ] SQLAlchemy Models definieren (siehe LOCAL_ARCHITECTURE_CONCEPT.md Abschnitt 5.1)
- [ ] Alembic Setup für Migrations
- [ ] Initial Migration erstellen
- [ ] DatabaseManager-Klasse mit Connection Pooling

**Dateien**:
```python
# smartrag/persistence/models.py
from sqlalchemy import Column, String, Integer, DateTime, JSON, ForeignKey, Text
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    user_id = Column(String, primary_key=True)
    username = Column(String, unique=True, nullable=False)
    created_at = Column(DateTime)
    last_active = Column(DateTime)
    settings = Column(JSON)

class Session(Base):
    __tablename__ = 'sessions'
    session_id = Column(String, primary_key=True)
    user_id = Column(String, ForeignKey('users.user_id'))
    created_at = Column(DateTime)
    last_active = Column(DateTime)
    context = Column(JSON)

# ... weitere Models
```

**Tests**:
- [ ] Unit Tests für alle Models
- [ ] Migration Up/Down Tests
- [ ] Constraint-Tests (Foreign Keys, Unique)

**Acceptance Criteria**:
- Alle Tabellen werden korrekt erstellt
- Migrations laufen fehlerfrei
- Grundlegende CRUD-Operationen funktionieren

---

### 1.3 Session Management

**Ziel**: Persistente Session-Verwaltung

**Tasks**:
- [ ] `SessionManager` Service implementieren
- [ ] Session Create/Load/Update/Delete
- [ ] Session-Context Management
- [ ] Automatisches Cleanup inaktiver Sessions

**Dateien**:
```python
# smartrag/services/session_manager.py
class SessionManager:
    def __init__(self, db: Database, vector_store: VectorStore):
        self.db = db
        self.vector_store = vector_store

    async def create_session(self, user_id: str = None) -> Session:
        """Erstellt neue Session mit isolierter ChromaDB Collection"""
        pass

    async def load_session(self, session_id: str) -> Session:
        """Lädt Session aus DB"""
        pass

    async def cleanup_inactive_sessions(self, hours: int = 24):
        """Löscht inaktive Sessions"""
        pass
```

**Tests**:
- [ ] Session Lifecycle Tests
- [ ] Concurrent Session Tests
- [ ] Cleanup Tests

**Acceptance Criteria**:
- Sessions werden persistent gespeichert
- Session-Context kann geladen werden
- Inaktive Sessions werden automatisch bereinigt

---

## Phase 2: API Layer (Woche 3-4)

### 2.1 REST API Endpoints

**Ziel**: Vollständige REST API für alle Operationen

**Priorität 1 (MVP)**:
```python
# smartrag/api/routes/sessions.py
@router.post("/sessions")
async def create_session(request: CreateSessionRequest) -> SessionResponse:
    """Erstellt neue Session"""
    pass

@router.get("/sessions/{session_id}")
async def get_session(session_id: str) -> SessionResponse:
    """Lädt Session-Details"""
    pass

# smartrag/api/routes/documents.py
@router.post("/documents/upload")
async def upload_document(file: UploadFile, session_id: str) -> DocumentResponse:
    """Lädt Dokument hoch und verarbeitet es"""
    pass

@router.get("/documents")
async def list_documents(session_id: str) -> List[DocumentResponse]:
    """Listet alle Dokumente einer Session"""
    pass

# smartrag/api/routes/queries.py
@router.post("/query")
async def query(request: QueryRequest) -> QueryResponse:
    """Synchrone RAG Query"""
    pass

@router.post("/query/stream")
async def query_stream(request: QueryRequest) -> StreamingResponse:
    """Streaming RAG Query"""
    pass
```

**Tasks**:
- [ ] Request/Response Models (Pydantic)
- [ ] Error Handling & Validation
- [ ] API Documentation (OpenAPI/Swagger)
- [ ] Rate Limiting (optional)

**Tests**:
- [ ] Integration Tests für alle Endpoints
- [ ] Validation Tests
- [ ] Error Case Tests

---

### 2.2 WebSocket Streaming

**Ziel**: Real-time Streaming für Queries

**Tasks**:
- [ ] WebSocket Handler implementieren
- [ ] Message Protocol definieren (JSON)
- [ ] Streaming-Integration mit Ollama
- [ ] Connection Management

**Dateien**:
```python
# smartrag/api/websocket/handlers.py
@app.websocket("/ws/query/{session_id}")
async def websocket_query(websocket: WebSocket, session_id: str):
    await websocket.accept()

    try:
        while True:
            # Empfange Query
            data = await websocket.receive_json()
            query = data['query']

            # Streaming Response
            async for chunk in rag_service.query_streaming(query, session_id):
                await websocket.send_json({
                    'type': 'chunk',
                    'content': chunk
                })

            await websocket.send_json({'type': 'done'})

    except WebSocketDisconnect:
        logger.info(f"Client disconnected: {session_id}")
```

**Tests**:
- [ ] WebSocket Connection Tests
- [ ] Streaming Tests
- [ ] Disconnect Handling Tests

---

### 2.3 Conversation Persistence

**Ziel**: Dauerhafte Speicherung aller Konversationen

**Tasks**:
- [ ] `ConversationManager` Service
- [ ] Message Storage & Retrieval
- [ ] Conversation Export (JSON/Markdown)
- [ ] Search in Conversation History

**Acceptance Criteria**:
- Alle Nachrichten werden persistent gespeichert
- Konversationen können geladen werden
- Export funktioniert in beiden Formaten

---

## Phase 3: Daemon Integration (Woche 5-6)

### 3.1 Systemd Service

**Ziel**: SmartRAG als Linux-Systemdienst

**Tasks**:
- [ ] Systemd Unit File erstellen
- [ ] Pre-Start Health Checks
- [ ] Graceful Shutdown Handler
- [ ] Log Rotation Integration

**Dateien**:
```ini
# systemd/smartrag.service
[Unit]
Description=SmartRAG - Local AI Knowledge Base
After=network.target

[Service]
Type=simple
User=smartrag
WorkingDirectory=/opt/smartrag
ExecStartPre=/opt/smartrag/scripts/pre-start.sh
ExecStart=/opt/smartrag/venv/bin/python -m smartrag.daemon
ExecStop=/opt/smartrag/scripts/graceful-shutdown.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

**Tasks**:
- [ ] Installation Script (`install.sh`)
- [ ] User/Group Setup
- [ ] Directory Permissions
- [ ] SELinux/AppArmor Policies (optional)

---

### 3.2 Lifecycle Management

**Ziel**: Sauberes Startup/Shutdown

**Tasks**:
- [ ] Startup-Sequenz implementieren:
  - Config validieren
  - Datenbank migrieren
  - Ollama-Server prüfen/starten
  - Models preloaden
  - API-Server starten

- [ ] Shutdown-Sequenz:
  - Neue Requests ablehnen
  - Aktive Requests abschließen
  - Sessions speichern
  - Datenbank schließen
  - Cleanup

**Dateien**:
```python
# smartrag/daemon/lifecycle.py
class DaemonLifecycle:
    async def startup(self):
        """Startup-Sequenz"""
        logger.info("Starting SmartRAG Daemon...")

        # 1. Config
        self.config = load_config()

        # 2. Database
        await self.init_database()

        # 3. Ollama
        await self.check_ollama()

        # 4. Models
        await self.preload_models()

        # 5. API
        await self.start_api_server()

        logger.info("SmartRAG Daemon ready")

    async def shutdown(self):
        """Graceful Shutdown"""
        logger.info("Shutting down SmartRAG Daemon...")

        # 1. Stop accepting requests
        self.api_server.stop_accepting()

        # 2. Wait for active requests (max 30s)
        await self.wait_for_active_requests(timeout=30)

        # 3. Close connections
        await self.close_database()
        await self.close_vector_store()

        logger.info("SmartRAG Daemon stopped")
```

---

### 3.3 Health Monitoring

**Ziel**: Umfassendes Health-Monitoring

**Tasks**:
- [ ] Health-Check Endpoint erweitern
- [ ] Component-Level Checks (DB, Ollama, Disk, Memory)
- [ ] Prometheus Metriken (optional)
- [ ] Alerting-Integration (optional)

**Metriken**:
```python
# smartrag/monitoring/metrics.py
from prometheus_client import Counter, Histogram, Gauge

queries_total = Counter('smartrag_queries_total', 'Total queries')
query_duration = Histogram('smartrag_query_duration_seconds', 'Query latency')
active_sessions = Gauge('smartrag_active_sessions', 'Active sessions')
documents_total = Gauge('smartrag_documents_total', 'Total documents')
```

---

## Phase 4: Enhancement & Tools (Woche 7-8)

### 4.1 CLI Tools

**Ziel**: Command-line Interface für Administration

**Commands**:
```bash
smartrag-cli status              # Service-Status
smartrag-cli upload <file>       # Dokument hochladen
smartrag-cli query "text"        # Query ausführen
smartrag-cli sessions list       # Sessions auflisten
smartrag-cli backup              # Backup erstellen
smartrag-cli restore <backup>    # Backup wiederherstellen
smartrag-cli optimize            # DB optimieren
```

**Implementation**:
```python
# smartrag/cli/main.py
import click
import requests

@click.group()
def cli():
    """SmartRAG CLI Tool"""
    pass

@cli.command()
def status():
    """Zeigt Service-Status"""
    response = requests.get('http://localhost:8765/health')
    click.echo(response.json())

@cli.command()
@click.argument('file', type=click.Path(exists=True))
def upload(file):
    """Lädt Dokument hoch"""
    with open(file, 'rb') as f:
        response = requests.post(
            'http://localhost:8765/api/v1/documents/upload',
            files={'file': f}
        )
    click.echo(f"Uploaded: {response.json()['document_id']}")
```

---

### 4.2 Backup & Restore

**Ziel**: Automatisches Backup-System

**Tasks**:
- [ ] Backup-Script (SQLite + ChromaDB + Files)
- [ ] Restore-Script
- [ ] Cron-Integration
- [ ] Retention Policy

**Dateien**:
```bash
# scripts/backup.sh
#!/bin/bash
BACKUP_DIR="/var/backups/smartrag/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# SQLite Backup (online)
sqlite3 /var/lib/smartrag/database/smartrag.db \
    ".backup '$BACKUP_DIR/smartrag.db'"

# ChromaDB (tar)
tar czf "$BACKUP_DIR/vector_db.tar.gz" \
    -C /var/lib/smartrag vector_db/

# Files (rsync incremental)
rsync -av --link-dest=/var/backups/smartrag/latest \
    /var/lib/smartrag/file_storage/ "$BACKUP_DIR/file_storage/"

# Symlink latest
ln -snf "$BACKUP_DIR" /var/backups/smartrag/latest

echo "Backup completed: $BACKUP_DIR"
```

---

### 4.3 Documentation & Examples

**Tasks**:
- [ ] API-Dokumentation (Swagger UI)
- [ ] Integration-Beispiele
- [ ] Deployment-Guide
- [ ] Troubleshooting-Guide

**Beispiele**:
```python
# examples/basic_integration.py
"""Beispiel: SmartRAG in eigener App nutzen"""
import requests

class MyApp:
    def __init__(self):
        self.smartrag = "http://localhost:8765/api/v1"
        self.session = self._create_session()

    def _create_session(self):
        response = requests.post(f"{self.smartrag}/sessions")
        return response.json()['session_id']

    def ask(self, question: str):
        response = requests.post(
            f"{self.smartrag}/query",
            json={
                'query': question,
                'session_id': self.session
            }
        )
        return response.json()['answer']

# Verwendung
app = MyApp()
answer = app.ask("Was ist RAG?")
print(answer)
```

---

## Phase 5: Testing & Optimization (Woche 9-10)

### 5.1 Testing

**Test-Coverage Ziele**:
- Unit Tests: 80%+
- Integration Tests: Alle API Endpoints
- E2E Tests: Kritische User-Flows

**Test-Suiten**:
```python
# tests/integration/test_full_workflow.py
async def test_complete_rag_workflow():
    """Testet kompletten RAG-Workflow"""
    # 1. Session erstellen
    session = await create_session()

    # 2. Dokument hochladen
    doc = await upload_document(session.id, "test.pdf")

    # 3. Warten auf Processing
    await wait_for_processing(doc.id)

    # 4. Query ausführen
    response = await query(session.id, "Test-Frage")

    # 5. Assertions
    assert response.answer is not None
    assert len(response.sources) > 0
```

**Tasks**:
- [ ] Unit Test Suite
- [ ] Integration Test Suite
- [ ] Performance Tests
- [ ] Load Tests

---

### 5.2 Performance Optimization

**Fokus-Bereiche**:
- [ ] Database Query Optimization (Indexes)
- [ ] Vector Search Caching
- [ ] Connection Pooling (DB + Ollama)
- [ ] Model Preloading
- [ ] Async I/O Optimization

**Benchmarks**:
```python
# benchmarks/query_performance.py
async def benchmark_query_latency():
    """Misst Query-Performance"""
    results = []
    for i in range(100):
        start = time.time()
        await query("Test Query")
        latency = time.time() - start
        results.append(latency)

    print(f"p50: {np.percentile(results, 50)}s")
    print(f"p95: {np.percentile(results, 95)}s")
    print(f"p99: {np.percentile(results, 99)}s")
```

---

### 5.3 Resource Optimization

**Ziel**: Effiziente Ressourcennutzung

**Tasks**:
- [ ] Memory Profiling
- [ ] Garbage Collection Tuning
- [ ] Model Quantization (optional)
- [ ] Disk I/O Optimization

---

## Abhängigkeiten & Voraussetzungen

### System Requirements
- **OS**: Linux (Ubuntu 20.04+, Debian 11+, CentOS 8+)
- **Python**: 3.10+
- **RAM**: Minimum 8GB, Empfohlen 16GB
- **Disk**: 50GB+ frei
- **CPU**: 4+ Cores empfohlen

### Software Dependencies
```txt
# Neue Dependencies für Daemon-Mode
fastapi==0.104.0
uvicorn[standard]==0.24.0
websockets==12.0
python-multipart==0.0.6
sqlalchemy==2.0.23
alembic==1.12.1
prometheus-client==0.19.0
click==8.1.7

# Bestehende Dependencies
streamlit==1.28.1
chromadb==0.4.15
ollama==0.1.0
# ... rest from existing requirements.txt
```

---

## Milestones & Deliverables

### Milestone 1: MVP API (Ende Woche 4)
**Deliverables**:
- ✅ FastAPI-Server läuft
- ✅ Basic REST API (Sessions, Documents, Queries)
- ✅ Session-Persistenz
- ✅ Conversation-History
- ✅ Tests (Unit + Integration)

**Definition of Done**:
- Alle API-Endpoints funktionieren
- Tests laufen durch (>80% Coverage)
- Dokumentation verfügbar

---

### Milestone 2: Daemon Integration (Ende Woche 6)
**Deliverables**:
- ✅ Systemd Service
- ✅ Installation Script
- ✅ Lifecycle Management
- ✅ Health Monitoring
- ✅ Logs & Metrics

**Definition of Done**:
- Service startet automatisch beim Boot
- Graceful Shutdown funktioniert
- Health-Checks laufen
- Logs werden geschrieben

---

### Milestone 3: Production-Ready (Ende Woche 10)
**Deliverables**:
- ✅ CLI Tools
- ✅ Backup/Restore
- ✅ Vollständige Tests
- ✅ Performance-Optimierung
- ✅ Dokumentation
- ✅ Deployment-Guides

**Definition of Done**:
- Alle Features implementiert
- Tests laufen durch (>80% Coverage)
- Performance-Ziele erreicht (<2s Query-Latenz p95)
- Dokumentation vollständig

---

## Risiken & Mitigation

| Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|--------|-------------------|--------|------------|
| Ollama-Kompatibilität | Mittel | Hoch | Frühzeitig testen, Fallback-Optionen |
| ChromaDB Migrations | Mittel | Mittel | Backup-Strategie, Testing |
| Performance-Issues | Hoch | Hoch | Kontinuierliches Profiling, Benchmarks |
| Memory Leaks | Mittel | Hoch | Memory Profiling, Stress-Tests |
| Breaking Changes | Niedrig | Mittel | Versionierung, Migrations-Guide |

---

## Erfolgsmetriken

### Technical Metrics
- **Query Latency (p95)**: <2 Sekunden
- **API Uptime**: >99.9%
- **Memory Usage**: <8GB under normal load
- **Test Coverage**: >80%
- **API Response Time**: <100ms (ohne LLM-Inference)

### User Metrics
- **Session Persistence**: 100% (kein Datenverlust)
- **Document Processing Success**: >95%
- **Concurrent Users**: 10+ ohne Degradation

---

## Nächste Schritte

**Sofort (diese Woche)**:
1. Projekt-Restrukturierung durchführen
2. Datenbank-Schema implementieren
3. FastAPI-Grundgerüst aufsetzen

**Diese Woche starten**:
```bash
# Branch für Development
git checkout -b feature/daemon-architecture

# Neue Verzeichnisse anlegen
mkdir -p smartrag/{daemon,api,services,persistence}

# FastAPI Dependencies installieren
pip install fastapi uvicorn sqlalchemy alembic

# Erste Implementation
touch smartrag/daemon/main.py
touch smartrag/api/app.py
```

**Review-Checkpoints**:
- Woche 2: Code Review Phase 1
- Woche 4: Milestone 1 Review
- Woche 6: Milestone 2 Review
- Woche 10: Final Review

---

**Dokumentversion**: 1.0
**Letztes Update**: 2025-11-22
**Status**: Ready for Implementation
