# 🌀 Wu Xing Agents

> Système multi-agents événementiel inspiré du cycle des **Cinq Éléments** chinois (五行).  
> Chaque agent incarne une phase de transformation, et la collaboration émerge d'un flux d'événements asynchrones.

---

## 🪷 Philosophie

Le **Wu Xing** (五行) décrit cinq phases qui s'engendrent mutuellement :

```
       🔥 Feu (création)
       ↗         ↘
🎯 Intention      🌍 Terre (mémoire)
       ↑              ↓
    🌳 Bois ←  💧 Eau ← ⚔️ Métal
   (livrable)  (raffinement) (critique)
```

Chaque agent est un **maillon spécialisé** d'une chaîne de transformation :

| Phase | Agent | Rôle |
|------|-------|------|
| 🎯 | **Intention** | Reformule la demande utilisateur en objectif clair |
| 🔥 | **Feu** | Génère une première proposition créative |
| 🌍 | **Terre** | Ancre la proposition dans la mémoire / le contexte |
| ⚔️ | **Métal** | Critique, affûte, identifie les faiblesses |
| 💧 | **Eau** | Raffine, lisse, harmonise |
| 🌳 | **Bois** | Produit le livrable final structuré |

---

## 🏛️ Architecture

### Vue d'ensemble

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ POST /run
       ▼
┌─────────────┐         ┌──────────────────┐
│  FastAPI    │────────▶│ Redis Streams    │
└─────────────┘         │ (event bus)      │
                        └────────┬─────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
  ┌───────────┐            ┌───────────┐            ┌───────────┐
  │ Intention │  ──event── │   Feu     │  ──event── │  Terre    │  ...
  │  worker   │            │  worker   │            │  worker   │
  └───────────┘            └───────────┘            └───────────┘
        │                        │                        │
        └────────────────────────┴────────────────────────┘
                                 ▼
                        ┌──────────────────┐
                        │  cycle_store     │
                        │  (Redis KV)      │
                        └──────────────────┘
```

### Principes clés

- **Event-driven** : aucun agent n'appelle directement un autre. Ils communiquent via des **événements** publiés sur Redis Streams.
- **Workers indépendants** : chaque agent tourne dans son propre worker async, avec son consumer group → **résilience** et **scalabilité horizontale**.
- **State persistant** : l'état du cycle est centralisé dans `cycle_store` (Redis), reconstructible à tout moment.
- **Traçabilité** : chaque événement porte un `event_id` + `correlation_id` permettant de rejouer un cycle complet.

### Flux d'événements

```
task.created
    └─▶ intention_worker
        └─▶ intention.clarified
            └─▶ fire_worker
                └─▶ creation.done
                    └─▶ earth_worker
                        └─▶ memory.anchored
                            └─▶ metal_worker
                                └─▶ critique.done
                                    └─▶ water_worker
                                        └─▶ refinement.done
                                            └─▶ wood_worker
                                                └─▶ cycle.completed
```

---

## 📁 Structure du projet

```
wuxing-agents/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── .env                          # OPENAI_API_KEY, etc.
├── src/
│   ├── api/
│   │   └── main.py               # FastAPI app + endpoints
│   ├── orchestrator.py           # démarre/arrête les workers
│   ├── agents/                   # logique métier (LLM)
│   │   ├── base.py
│   │   ├── intention.py
│   │   ├── fire.py
│   │   ├── earth.py
│   │   ├── metal.py
│   │   ├── water.py
│   │   └── wood.py
│   ├── workers/                  # adapters event-driven
│   │   ├── base.py               # BaseWorker (consume + emit)
│   │   ├── intention_worker.py
│   │   ├── fire_worker.py
│   │   ├── earth_worker.py
│   │   ├── metal_worker.py
│   │   ├── water_worker.py
│   │   └── wood_worker.py
│   ├── bus/
│   │   ├── events.py             # EventType + Event (Pydantic)
│   │   ├── redis_bus.py          # publish / subscribe Streams
│   │   └── cycle_store.py        # persistance d'état
│   └── llm/
│       └── router.py             # LiteLLM (multi-providers)
└── README.md
```

---

## 🚀 Démarrage rapide

### Pré-requis

- **Docker** + **Docker Compose**
- Une clé API **OpenAI** (ou autre provider compatible LiteLLM)

### Installation

```bash
git clone <repo-url>
cd wuxing-agents

# Configure l'environnement
cp .env.example .env
# Édite .env et renseigne OPENAI_API_KEY=sk-...

# Lance le stack (app + Redis)
sudo docker compose up -d --build

# Vérifie que tout tourne
sudo docker compose logs app --tail=20
```

Tu dois voir :
```
🎼 Orchestrateur : 6 workers actifs
🚀 Worker 'intention_worker' démarré
🚀 Worker 'fire_worker' démarré
... (×6)
```

---

## 🎮 Utilisation

### 1. Lancer un cycle

```bash
curl -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"user_input": "Rédige un haïku sur la pluie"}'
```

Réponse (immédiate, < 1s) :
```json
{
  "cycle_id": "3d8acd55-11c7-4063-baf4-da878f9ec0a5",
  "status": "started"
}
```

### 2. Suivre l'avancement

```bash
CYCLE=3d8acd55-11c7-4063-baf4-da878f9ec0a5
curl -s http://localhost:8000/cycles/$CYCLE | jq
```

Pendant le cycle (~30-40s), tu verras `status` évoluer :
```
phase_intention_done → phase_fire_done → phase_earth_done
→ phase_metal_done → phase_water_done → completed
```

### 3. Récupérer le livrable final

```bash
curl -s http://localhost:8000/cycles/$CYCLE \
  | jq '{status, deliverable: .final_deliverable}'
```

```json
{
  "status": "completed",
  "deliverable": {
    "type": "text",
    "content": "Pluie d'été murmure,\nLes feuilles dansent, étreintes,\nÉclats de lumière.",
    "metadata": {"agent": "wood"},
    "created_at": "2026-04-29T16:21:31.143945",
    "cycle": 1
  }
}
```

---

## 🔍 Observation & Debug

### Voir les logs en live

```bash
sudo docker compose logs app -f
```

### Inspecter les streams Redis

```bash
# Liste des streams actifs
sudo docker compose exec redis redis-cli KEYS 'wuxing:events:*'

# Nombre d'événements émis sur un type
sudo docker compose exec redis redis-cli XLEN wuxing:events:intention.clarified

# Lire les 5 derniers événements
sudo docker compose exec redis redis-cli XREVRANGE wuxing:events:cycle.completed + - COUNT 5
```

### Inspecter un cycle stocké

```bash
sudo docker compose exec redis redis-cli KEYS 'cycle:*'
sudo docker compose exec redis redis-cli GET cycle:<cycle_id>
```

---

## 🧪 Endpoints API

| Méthode | Route | Description |
|--------|-------|-------------|
| `POST` | `/run` | Démarre un nouveau cycle |
| `GET`  | `/cycles/{cycle_id}` | État complet d'un cycle |
| `GET`  | `/health` | Liveness probe |

### Schéma `POST /run`

```json
{
  "user_input": "string (la demande de l'utilisateur)"
}
```

---

## ⚙️ Configuration

### Variables d'environnement (`.env`)

```bash
OPENAI_API_KEY=sk-...          # requis
REDIS_URL=redis://redis:6379   # par défaut dans docker-compose
LOG_LEVEL=INFO                 # DEBUG pour plus de verbosité
```

### Choisir le modèle par agent

Dans `src/agents/<element>.py`, chaque agent définit son modèle et sa température :

```python
class FireAgent(BaseAgent):
    name = "fire"
    model = "gpt-4o-mini"
    temperature = 0.9   # haute créativité

class MetalAgent(BaseAgent):
    name = "metal"
    model = "gpt-4o-mini"
    temperature = 0.2   # critique précise
```

LiteLLM permet de pointer vers Claude, Gemini, Mistral, Ollama local, etc. en changeant simplement la string `model`.

---

## 🛠️ Développement

### Relancer après modif

```bash
sudo docker compose restart app
```

### Rebuild complet (si requirements.txt change)

```bash
sudo docker compose up -d --build
```

### Reset total (efface les cycles + streams)

```bash
sudo docker compose down -v
sudo docker compose up -d --build
```

---

## 🧩 Étendre le système

### Ajouter un nouvel agent

1. Crée `src/agents/<nom>.py` héritant de `BaseAgent`
2. Crée `src/workers/<nom>_worker.py` héritant de `BaseWorker`
3. Définis `listens_to = [EventType.XXX]` et le `EventType` émis
4. Ajoute le worker dans `src/orchestrator.py`
5. (Optionnel) Ajoute le nouveau `EventType` dans `src/bus/events.py`

### Scaler horizontalement

Pour lancer plusieurs instances du même worker (réparties par Redis) :

```yaml
# docker-compose.yml
app:
  deploy:
    replicas: 3
```

Les consumer groups Redis garantissent que chaque message est traité **une seule fois**.

---

## 📊 Caractéristiques techniques

- **Langage** : Python 3.11
- **API** : FastAPI (async)
- **Bus d'événements** : Redis Streams + consumer groups
- **LLM** : LiteLLM (multi-providers)
- **Validation** : Pydantic v2
- **Conteneurisation** : Docker Compose

---

## 🗺️ Roadmap

- [ ] Endpoint SSE `/cycles/{id}/stream` pour suivi temps réel
- [ ] Boucle de raffinement (Eau → Feu si qualité insuffisante)
- [ ] Retry / Dead-Letter Queue
- [ ] Tests d'intégration end-to-end
- [ ] Mémoire Terre persistante (vector DB)
- [ ] Front web (visualisation du cycle des 5 phases)

---

## 📜 Licence

MIT — fais-en bon usage 🌿

---

> *"Quand le Bois nourrit le Feu, que le Feu enrichit la Terre,  
> que la Terre forme le Métal, que le Métal porte l'Eau,  
> et que l'Eau nourrit le Bois — alors le cycle est complet."*
```

---

## 📝 Pour le mettre en place

```bash
cd ~/wuxing-agents/wuxing-agents

# Crée le fichier
nano README.md
# (colle le contenu, Ctrl+O, Enter, Ctrl+X)

# Crée aussi un .env.example pour faciliter l'onboarding
cat > .env.example <<'EOF'
OPENAI_API_KEY=sk-your-key-here
REDIS_URL=redis://redis:6379
LOG_LEVEL=INFO
EOF

# Commit
git add README.md .env.example
git commit -m "docs: add architecture README and env example"
```
