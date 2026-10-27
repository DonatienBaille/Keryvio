# Spécification technique — Plateforme IA On-Premise

**Version :** 0.2.0 — MVP  
**Statut :** Spécification fonctionnelle et technique  
**Backend :** Rust  
**Frontend :** Angular  
**Hébergement :** 100 % On-Premise  
**Architecture :** Multi-utilisateur, modulaire, extensible et white-label

---

# 1. Objectif du projet

Développer une plateforme conversationnelle d'intelligence artificielle professionnelle, comparable à ChatGPT, entièrement auto-hébergeable.

La plateforme doit permettre à plusieurs utilisateurs de communiquer avec différents modèles LLM locaux, tout en offrant une expérience conversationnelle moderne.

Elle doit également exposer une API utilisable par des applications tierces, compatible avec les principales conventions de l'API OpenAI.

## 1.1 Principes architecturaux

1. **On-Premise First** : aucune dépendance cloud obligatoire.
2. **Provider Agnostic** : indépendance vis-à-vis des fournisseurs de modèles et d'identité.
3. **Responses API First** : la Responses API constitue le protocole canonique d'inférence.
4. **Multi-Transport** : HTTP, SSE et WebSocket.
5. **Multi-User** : données et permissions isolées par utilisateur.
6. **Security by Design** : authentification, autorisation et audit.
7. **White-Label** : identité visuelle totalement personnalisable.
8. **Extensible** : ajout ultérieur d'agents, MCP, RAG et clients desktop.
9. **Protocol Compatibility** : compatibilité vérifiée avec des SDK OpenAI.
10. **Graceful Degradation** : une fonctionnalité non supportée par un modèle doit être explicitement signalée.

## 1.2 Périmètre fonctionnel MVP

| Fonctionnalité | MVP |
|---|---|
| Interface Web responsive type ChatGPT | Oui |
| Authentification OIDC générique | Oui |
| Multi-utilisateur et RBAC | Oui |
| Sessions serveur révocables | Oui |
| Conversations persistantes | Oui |
| Sélection des modèles | Oui |
| Contrôle du niveau de raisonnement | Oui |
| Responses API HTTP | Oui |
| Responses API SSE | Oui |
| Responses API WebSocket | Oui |
| Multiplexage WebSocket via `stream_id` | Oui |
| Chat Completions API | Oui |
| Continuation via `previous_response_id` | Oui |
| Background Responses | Oui |
| Function Calling côté protocole | Oui |
| Historique et régénération | Oui |
| Annulation des générations | Oui |
| Suivi des tokens | Oui |
| Administration des modèles | Oui |
| Clés API personnelles | Oui |
| OpenTelemetry | Oui |
| Upload documentaire et RAG | Non |
| Exécution autonome des outils | Non |
| Agents et MCP | Non |
| OpenAI Realtime audio | Non |
| Client desktop | Non |

L'expression *compatible OpenAI* désigne le sous-ensemble documenté et testé par ce projet, et non une implémentation intégrale de tous les services et outils propriétaires d'OpenAI.

---

# 2. Technologies

## 2.1 Frontend

- Angular.
- TypeScript.
- Tailwind CSS.
- Angular Signals.
- HTTP REST.
- SSE.
- WebSocket natif.
- Rendu Markdown.
- Shiki pour la coloration syntaxique.
- KaTeX pour les expressions mathématiques.

La même interface devra pouvoir être réutilisée ultérieurement dans une application Tauri.

## 2.2 Backend

| Composant | Technologie |
|---|---|
| Langage | Rust |
| API HTTP | Axum |
| Runtime asynchrone | Tokio |
| WebSocket | Axum WebSocket / Tokio |
| Client HTTP | Reqwest |
| Client WebSocket amont | tokio-tungstenite ou équivalent |
| Middlewares | Tower |
| Sérialisation | Serde |
| Base de données | PostgreSQL |
| Accès SQL | SQLx |
| Authentification | OIDC / OAuth 2.0 |
| Observabilité | OpenTelemetry + tracing |

L'architecture sera un **monolithe modulaire**, sans multiplication prématurée des microservices.

## 2.3 Infrastructure

- NGINX.
- PostgreSQL.
- Rust / Axum.
- Angular.
- SGLang et/ou vLLM.
- Fournisseur OIDC configurable.
- Docker Compose.

---

# 3. Architecture générale

```text
                  CLIENTS
                     |
          +----------+----------+
          |                     |
     Angular Web          SDK / Apps
          |                     |
          | HTTPS / WSS         | HTTP / WSS
          |                     |
          +----------+----------+
                     |
                   NGINX
                     |
                     v
           +--------------------+
           |      RUST API      |
           |       AXUM         |
           +--------------------+
                     |
        +------------+------------+
        |            |            |
        v            v            v
   Identity      Conversation   Model
   & Access      Service        Registry
        |            |            |
        +------------+------------+
                     |
                     v
           +--------------------+
           |   INFERENCE CORE   |
           |                    |
           | Responses Protocol |
           | Streaming Events   |
           | Routing / Context  |
           +--------------------+
                     |
          +----------+----------+
          |                     |
          v                     v
     HTTP/SSE Adapter      WS Adapter
          |                     |
          +----------+----------+
                     |
                     v
           SGLang / vLLM / LLM
                     |
               GPU Servers

    PostgreSQL : users, sessions,
    conversations, responses, usage
```

## 3.1 Principe de normalisation

Les réponses du moteur d'inférence sont converties vers un modèle interne commun.

Ce modèle est ensuite exposé :

- En JSON HTTP.
- En streaming SSE.
- En événements WebSocket.

Le frontend et les clients SDK ne doivent pas connaître les particularités des moteurs d'inférence.

## 3.2 Séparation des interfaces

Deux familles d'API doivent coexister :

**API applicative**

```text
/api/v1/*
```

Utilisée pour les conversations, préférences, utilisateurs et opérations d'administration.

**API compatible OpenAI**

```text
/v1/*
```

Utilisée par le frontend lorsque pertinent et par les SDK ou applications tierces.

Les deux interfaces partagent les mêmes services d'inférence, permissions et quotas.

---

# 4. Authentification et sécurité

## 4.1 Authentification des utilisateurs Web

L'authentification utilise :

- OAuth 2.0.
- OpenID Connect.
- Authorization Code Flow + PKCE.
- OIDC Discovery.
- JWKS.
- Sessions serveur.

Aucun fournisseur d'identité particulier n'est imposé.

Fournisseurs envisageables :

- Keycloak.
- ZITADEL.
- Authentik.
- Microsoft Entra ID.
- Dex.
- Tout IdP respectant le profil OIDC requis.

LDAP/Active Directory reste accessible par fédération auprès de l'IdP.

## 4.2 Architecture BFF

Le backend Rust gère les échanges avec le fournisseur OIDC.

Le navigateur ne stocke pas directement les jetons d'accès OIDC.

Une session applicative est exposée par cookie sécurisé.

Exigences :

- HttpOnly.
- Secure.
- SameSite adapté au déploiement.
- Protection CSRF.
- Sessions révocables.
- Expiration absolue et d'inactivité.
- Déconnexion distante.

## 4.3 Authentification des clients SDK

Les applications tierces doivent pouvoir utiliser des clés API personnelles.

Exemple :

```http
Authorization: Bearer <api-key>
```

Une clé API possède :

- Un identifiant interne.
- Un propriétaire.
- Un nom.
- Une empreinte cryptographique stockée côté serveur.
- Des permissions.
- Une expiration optionnelle.
- Une date de dernière utilisation.
- Un statut de révocation.

La clé n'est affichée en clair qu'à sa création.

Les clés API doivent être révocables indépendamment des sessions Web.

Les SDK utilisent des clés API ; le navigateur Angular utilise sa session applicative.

## 4.4 Authentification WebSocket

Deux mécanismes sont autorisés :

- Session applicative sécurisée pour Angular.
- Bearer Token pendant le handshake pour les clients techniques.

Les connexions WebSocket doivent vérifier l'identité et les permissions avant l'upgrade.

Pour les connexions navigateur fondées sur un cookie, le serveur doit vérifier l'en-tête `Origin`.

Les jetons d'accès ne doivent pas être transmis dans l'URL WebSocket.

Les permissions doivent être réévaluées lors des nouvelles opérations et les connexions doivent pouvoir être fermées après révocation.

## 4.5 Permissions

Rôles initiaux :

- User.
- PowerUser.
- Administrator.

Permissions principales :

- Accès aux modèles.
- Création de conversations.
- Consultation des conversations personnelles.
- Utilisation des API.
- Gestion des clés API.
- Administration des modèles.
- Consultation des statistiques.
- Gestion et révocation des sessions.

Une session ou une clé API ne doit jamais permettre d'accéder aux conversations d'un autre utilisateur sans autorisation explicite.

---

# 5. Responses API — Protocole principal

## 5.1 Objectif

Implémenter un profil de compatibilité avec l'OpenAI Responses API.

Ce profil sera utilisé :

1. Pour les applications clientes utilisant les SDK OpenAI.
2. Comme contrat interne de la couche d'inférence.
3. Pour les conversations du frontend.
4. Pour les futurs agents et outils.

La compatibilité sera basée sur la documentation officielle OpenAI et, lorsque pertinent, la spécification ouverte Open Responses.

Le profil exact pris en charge sera versionné et documenté.

## 5.2 Endpoints

```http
POST   /v1/responses
GET    /v1/responses/{response_id}
DELETE /v1/responses/{response_id}

GET    /v1/responses/{response_id}/input_items

POST   /v1/responses/{response_id}/cancel
```

La création d'une réponse doit prendre en charge les modes synchrones, streaming et background.

L'annulation via `/cancel` respecte le contrat OpenAI pour les réponses `background=true`.

L'annulation des générations interactives ordinaires reste disponible via l'API applicative.

## 5.3 Format d'entrée

Exemple :

```json
{
  "model": "qwen-local",
  "input": [
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Explique le fonctionnement de Rust."
        }
      ]
    }
  ],
  "reasoning": {
    "effort": "medium"
  },
  "stream": true
}
```

### Paramètres MVP

Le profil initial comprend :

- `model`.
- `input` sous forme de chaîne ou de tableau d'items textuels.
- `instructions`.
- `previous_response_id`.
- `stream`.
- `store`.
- `background`.
- `reasoning.effort`.
- `temperature` lorsque supporté.
- `top_p` lorsque supporté.
- `max_output_tokens`.
- `tools` pour les fonctions client.
- `tool_choice`.
- `parallel_tool_calls` lorsque supporté.
- `text.format` pour les formats structurés pris en charge.
- `metadata` selon les contraintes retenues.

Les paramètres non supportés doivent produire une erreur explicite.

Les entrées multimodales ne font pas partie du profil initial.

## 5.4 Objet Response

Le backend doit exposer des objets structurés comprenant notamment :

```text
id
object
created_at
status
model
output
usage
error
incomplete_details
previous_response_id
reasoning
background
```

Les identifiants doivent rester stables et uniques.

Les sorties doivent utiliser les types d'items appropriés, notamment :

- `message`.
- `reasoning` lorsque le moteur expose un élément compatible.
- `function_call`.

Les sorties textuelles doivent être exposées au travers de contenus typés `output_text`.

Ne pas réduire systématiquement toutes les sorties à une chaîne de texte.

## 5.5 Continuation des conversations

La plateforme doit prendre en charge :

```json
{
  "model": "qwen-local",
  "previous_response_id": "resp_previous",
  "input": "Donne un exemple."
}
```

Le backend doit retrouver le contexte lié à la réponse précédente et construire la nouvelle requête.

La continuation doit fonctionner même si le moteur d'inférence ne fournit pas lui-même de mémoire conversationnelle.

Les références à des réponses appartenant à d'autres utilisateurs doivent être rejetées.

## 5.6 Politique de stockage

Le paramètre `store` doit être respecté.

### `store=true`

- Conservation des éléments requis dans PostgreSQL.
- Possibilité de récupérer une réponse.
- Continuation via son identifiant.
- Application d'une politique de rétention.

### `store=false`

- Pas de conservation durable du contenu pour cette requête API.
- Possibilité de conserver temporairement le contexte en mémoire pour la connexion WebSocket active.
- Absence de garantie de récupération après perte du cache.
- Pas de conservation silencieuse des prompts dans les traces.

Les conversations enregistrées volontairement depuis l'interface Web relèvent d'une politique de persistance applicative distincte, clairement signalée à l'utilisateur.

## 5.7 Background Responses

Le MVP doit gérer des générations persistantes en arrière-plan.

États envisagés :

```text
queued
in_progress
completed
failed
incomplete
cancelled
```

Une réponse lancée avec `background=true` doit pouvoir être récupérée et annulée.

L'exécution sera pilotée par des tâches asynchrones Rust avec états persistés et mécanisme de récupération au redémarrage.

---

# 6. Streaming SSE

## 6.1 Protocole

Les réponses HTTP streamées doivent utiliser :

```http
Content-Type: text/event-stream
```

Le flux doit respecter le format événementiel de la Responses API.

## 6.2 Événements

Événements principaux :

```text
response.created
response.in_progress

response.output_item.added
response.content_part.added

response.output_text.delta
response.output_text.done

response.function_call_arguments.delta
response.function_call_arguments.done

response.content_part.done
response.output_item.done

response.completed
response.failed
response.incomplete

error
```

Les événements liés aux fonctions ne sont émis que lorsqu'une fonction est effectivement produite.

Les données de raisonnement ne sont exposées que lorsque le moteur fournit une sortie compatible et autorisée à l'affichage.

Chaque événement doit respecter son schéma et les règles d'ordonnancement du protocole.

## 6.3 Gestion du streaming

Le backend doit gérer :

- L'ordre des événements.
- Les numéros de séquence.
- Les limites de taille des messages.
- Le backpressure.
- Les timeouts.
- La fermeture des connexions.
- Les réponses partielles.
- Les interruptions.
- Les erreurs en cours de génération.

Une erreur de génération ne doit jamais produire artificiellement un événement de succès.

---

# 7. Responses API WebSocket

## 7.1 Objectif

Implémenter le transport WebSocket persistant de la Responses API.

Ce transport permet :

- Plusieurs tours sur une connexion.
- L'envoi d'inputs incrémentaux.
- La continuation d'une réponse.
- Le multiplexage.
- Les conversations parallèles.
- Une latence réseau réduite pour les échanges successifs.

Il est indépendant de l'OpenAI Realtime API, destinée notamment aux interactions audio temps réel.

## 7.2 Endpoint

Le même chemin doit accepter une connexion WebSocket :

```text
wss://ai.example.com/v1/responses
```

Une requête HTTP classique utilise également `/v1/responses`, avec une méthode et un transport différents.

## 7.3 Création d'une réponse

Exemple :

```json
{
  "type": "response.create",
  "stream_id": "conversation_123",
  "model": "qwen-local",
  "store": false,
  "input": [
    {
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Bonjour."
        }
      ]
    }
  ]
}
```

Les champs HTTP spécifiques comme `stream` ou `background` ne doivent pas être envoyés dans le message WebSocket `response.create`.

## 7.4 Continuation

```json
{
  "type": "response.create",
  "stream_id": "conversation_123",
  "model": "qwen-local",
  "previous_response_id": "resp_previous",
  "input": "Continue."
}
```

La plateforme doit réutiliser le contexte disponible associé à `previous_response_id`.

Lorsque le contexte n'est plus récupérable, le backend doit retourner une erreur explicite.

Il ne doit pas fabriquer une continuation à partir d'un historique incomplet.

## 7.5 Multiplexage

Une connexion WebSocket peut transporter plusieurs flux indépendants.

Exemple :

```text
WebSocket Connection
    |
    +-- stream_id: conversation_A
    |       |
    |       +-- response_1
    |       +-- response_2
    |
    +-- stream_id: conversation_B
    |       |
    |       +-- response_3
    |
    +-- stream_id: conversation_C
            |
            +-- response_4
```

Exigences :

- Plusieurs streams par connexion.
- Exécution FIFO des requêtes d'un même stream.
- Parallélisme entre streams différents.
- Identification explicite du stream dans les événements.
- Routage correct des événements entre conversations.
- Limites configurables de streams et générations concurrentes.
- Isolation des droits d'accès entre utilisateurs.
- Possibilité de créer une branche depuis une réponse précédente autorisée.

Le profil de compatibilité ciblera les conventions `stream_id` documentées par OpenAI.

## 7.6 Cycle de vie WebSocket

Le serveur doit prendre en charge :

- Heartbeat / Ping-Pong.
- Détection des déconnexions.
- Backpressure.
- Timeouts d'inactivité.
- Limites de connexions par utilisateur.
- Limites de générations actives.
- Fermeture après révocation d'accès.
- Gestion des erreurs par stream.
- Reconnexion.

Les limites du service doivent être configurables et documentées.

## 7.7 Reconnexion

Après une déconnexion :

1. Le client ouvre une nouvelle connexion.
2. Il restaure ses streams applicatifs.
3. Il tente une continuation avec `previous_response_id`.
4. Si la réponse précédente n'est pas disponible, il reconstruit explicitement le contexte autorisé.

Les éléments conservés uniquement dans le cache de la connexion précédente peuvent être perdus.

Le backend ne doit pas promettre une reprise transparente lorsque le contexte n'est plus disponible.

## 7.8 Événements communs

Les événements transmis par WebSocket doivent reprendre les objets de la Responses API utilisés pour le SSE.

La plateforme ne doit pas entretenir deux formats concurrents pour représenter une génération.

Exemple :

```json
{
  "type": "response.output_text.delta",
  "stream_id": "conversation_123",
  "sequence_number": 12,
  "item_id": "msg_123",
  "output_index": 0,
  "content_index": 0,
  "delta": "Bonjour",
  "logprobs": []
}
```

L'implémentation doit gérer les champs et enveloppes prévus par le profil de protocole.

## 7.9 Fonctionnalités WebSocket futures

L'architecture doit permettre d'ajouter :

- `response.steer` et le Mid-turn Steering.
- Préparation d'une génération avec `generate=false`.
- Gestion avancée des forks et du contexte compacté.
- Interactions multimodales.

Le MVP n'annonce pas de compatibilité avec ces fonctionnalités tant que leur comportement n'a pas été implémenté et testé.

---

# 8. Chat Completions API

Le projet doit conserver une API de compatibilité Chat Completions.

Endpoint :

```http
POST /v1/chat/completions
```

Fonctionnalités :

- Messages multi-tours.
- Streaming ou réponse complète.
- Paramètres d'inférence pris en charge.
- Tool calling lorsque le modèle le permet.
- Usage.
- Gestion des erreurs.

Les réponses doivent respecter les structures Chat Completions, y compris le format des chunks en streaming.

Le backend peut convertir les requêtes Chat Completions vers son protocole d'inférence interne.

Cette conversion ne doit pas altérer les rôles, paramètres ou appels d'outils.

---

# 9. Function Calling

## 9.1 Périmètre MVP

La Responses API doit permettre de déclarer des fonctions.

Exemple :

```json
{
  "model": "qwen-local",
  "input": "Quelle est la météo ?",
  "tools": [
    {
      "type": "function",
      "name": "get_weather",
      "description": "Récupère la météo",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string"
          }
        },
        "required": ["city"],
        "additionalProperties": false
      },
      "strict": true
    }
  ]
}
```

La plateforme doit prendre en charge :

- La déclaration des fonctions.
- Les schémas JSON.
- La génération des arguments.
- Les événements de function calling.
- Les identifiants `call_id`.
- Les réponses de type `function_call`.
- La fourniture de `function_call_output` lors d'une continuation.

## 9.2 Limitation MVP

Le serveur ne doit pas exécuter automatiquement les fonctions fournies par des clients tiers.

Il retourne l'appel à l'application cliente, qui pourra exécuter la fonction et transmettre son résultat.

Le runtime agentique avec exécution d'outils, MCP, sandbox et approbations est prévu pour une phase ultérieure.

---

# 10. Gestion des modèles

## 10.1 Model Registry

Chaque modèle possède :

```text
id
display_name
description
provider
endpoint
remote_model_id
context_window
max_output_tokens
capabilities
reasoning_config
enabled
access_policy
```

Les capacités sont configurables et, lorsque possible, vérifiées par des tests de compatibilité.

Le registre doit distinguer :

- Capacités déclarées.
- Capacités vérifiées.
- Capacités non supportées.

## 10.2 Adaptateurs d'inférence

L'architecture doit prévoir :

```text
InferenceProvider
    |
    +-- ResponsesHttpAdapter
    |
    +-- ChatCompletionsAdapter
    |
    +-- ResponsesWebSocketAdapter
```

Le backend privilégie un endpoint Responses natif lorsque sa compatibilité a été validée.

En l'absence de support natif, il peut convertir les requêtes vers Chat Completions.

Cette conversion doit préserver les informations nécessaires à l'application et signaler les fonctionnalités qu'elle ne peut pas reproduire.

**Le support WebSocket côté client ne doit pas exiger un serveur d'inférence WebSocket.**

Le backend Rust peut recevoir les événements d'un serveur d'inférence en HTTP/SSE et les transmettre sur une connexion WebSocket persistante.

## 10.3 Niveaux de raisonnement

L'interface doit permettre de modifier le raisonnement directement dans le chat.

Niveaux logiques :

- Désactivé.
- Faible.
- Moyen.
- Élevé.
- Très élevé.

Chaque modèle définit ses valeurs réellement acceptées.

Le backend convertit les niveaux logiques vers les paramètres appropriés, par exemple `reasoning.effort`, `reasoning_effort` ou des paramètres de template.

Les niveaux non supportés doivent être masqués ou explicitement refusés.

## 10.4 Formats structurés

Le MVP doit prendre en charge les sorties textuelles et les sorties structurées conformes aux capacités des modèles.

La prise en charge des formats JSON Schema doit être activée uniquement pour les modèles et moteurs validés.

---

# 11. Conversations et interface utilisateur

## 11.1 Fonctionnalités Web

- Création de conversation.
- Liste des conversations.
- Renommage.
- Suppression.
- Historique persistant.
- Affichage Markdown.
- Code avec coloration syntaxique.
- Rendu LaTeX.
- Copie du contenu.
- Sélection du modèle.
- Sélection du raisonnement.
- Réponses streamées.
- Arrêt d'une génération.
- Régénération.
- Affichage des erreurs.
- Thèmes clair et sombre.

## 11.2 Composants de l'interface

```text
AppShell
ConversationSidebar
ConversationView
MessageList
MessageRenderer
ChatComposer
ModelSelector
ReasoningSelector
StreamingStatus
UserSettings
AdminDashboard
```

## 11.3 Abstraction des transports

Le frontend utilise une abstraction unique pour l'inférence.

```typescript
interface InferenceTransport {
  connect(): Promise<void>;

  send(
    request: ResponseRequest
  ): AsyncIterable<ResponseEvent>;

  cancel(responseId: string): Promise<void>;

  disconnect(): Promise<void>;
}
```

Implémentations prévues :

```text
HttpTransport
SseTransport
WebSocketTransport
```

Les interfaces représentées sont conceptuelles : leur contrat définitif sera établi pendant l'implémentation.

L'interface utilisateur ne doit pas dépendre directement de la technologie de transport.

## 11.4 Choix du transport

Par défaut :

- HTTP pour les opérations non streamées.
- SSE pour le streaming standard.
- WebSocket pour les sessions persistantes et interactions avancées.

Le choix peut être configuré selon le navigateur, la disponibilité réseau et les capacités de la plateforme.

Le WebSocket ne doit pas être une condition nécessaire pour utiliser le chat.

---

# 12. API applicative

## 12.1 Utilisateurs

```http
GET    /api/v1/me

GET    /api/v1/me/sessions
DELETE /api/v1/me/sessions/{id}

GET    /api/v1/me/api-keys
POST   /api/v1/me/api-keys
DELETE /api/v1/me/api-keys/{id}
```

## 12.2 Conversations

```http
GET    /api/v1/conversations
POST   /api/v1/conversations

GET    /api/v1/conversations/{id}
PATCH  /api/v1/conversations/{id}
DELETE /api/v1/conversations/{id}

GET    /api/v1/conversations/{id}/messages
POST   /api/v1/conversations/{id}/messages
```

## 12.3 Générations applicatives

```http
GET    /api/v1/runs/{id}
POST   /api/v1/runs/{id}/cancel
```

La politique d'annulation applicative peut différer de celle de l'endpoint OpenAI `/v1/responses/{id}/cancel`, sans modifier le contrat de ce dernier.

## 12.4 Modèles

```http
GET /v1/models

GET /api/v1/models
GET /api/v1/models/{id}/capabilities
```

## 12.5 Administration

```http
GET    /api/v1/admin/users
PATCH  /api/v1/admin/users/{id}

GET    /api/v1/admin/models
POST   /api/v1/admin/models
PATCH  /api/v1/admin/models/{id}
DELETE /api/v1/admin/models/{id}

GET    /api/v1/admin/usage
GET    /api/v1/admin/audit
```

---

# 13. Base de données

## 13.1 Tables principales

```text
users
external_identities
sessions
api_keys
roles
permissions
user_roles
group_role_mappings

model_providers
models
model_capabilities
model_permissions

conversations
messages
responses
response_items
response_inputs
response_events

inference_runs
usage_events

audit_logs
application_settings
```

## 13.2 Modèle de persistance

Les conversations et réponses sont liées à un propriétaire interne.

Les réponses doivent pouvoir représenter :

- Les liens de continuation.
- Les sorties structurées.
- Les appels de fonctions.
- Les statuts des générations.
- La consommation.
- Les erreurs.
- Les paramètres utilisés.

Les événements de streaming peuvent être journalisés de manière temporaire ou durable selon le type de génération et la politique de stockage.

Les requêtes `store=false` ne doivent pas provoquer de stockage durable de contenu.

## 13.3 Isolation

- Vérification systématique de l'utilisateur propriétaire.
- Application des règles RBAC.
- Protection des flux SSE.
- Protection des streams WebSocket.
- Contrôle d'accès aux réponses antérieures.
- Protection des événements et journaux sensibles.
- Possibilité de renforcer avec PostgreSQL RLS.

---

# 14. Fiabilité et performances

## 14.1 Streaming

L'application doit limiter la consommation mémoire par connexion.

Elle doit prévoir :

- Flux asynchrones.
- Buffers bornés.
- Backpressure.
- Annulation.
- Timeouts.
- Gestion des clients lents.
- Contrôle des générations concurrentes.

## 14.2 WebSocket

Une connexion persistante ne doit pas monopoliser un worker dédié bloquant.

La gestion des événements doit reposer sur les primitives asynchrones Tokio.

Les streams doivent rester indépendants au niveau logique.

## 14.3 Background Tasks

Les générations de fond doivent conserver un état cohérent après un redémarrage.

Une tâche interrompue doit être reprise uniquement lorsque le moteur et le protocole le permettent. Sinon, elle doit être marquée comme interrompue ou échouée, sans duplication silencieuse.

## 14.4 Erreurs

Les API compatibles OpenAI doivent retourner des erreurs structurées compréhensibles par les clients OpenAI.

Les erreurs doivent distinguer :

- Requête invalide.
- Authentification requise.
- Permission insuffisante.
- Modèle inconnu.
- Capacité non supportée.
- Réponse introuvable.
- Limite de débit.
- Timeout.
- Erreur du fournisseur d'inférence.
- Interruption d'un flux.

---

# 15. Observabilité

OpenTelemetry est obligatoire.

Métriques minimales :

- Utilisateurs actifs.
- Conversations actives.
- Connexions WebSocket actives.
- Streams simultanés.
- Générations simultanées.
- Latence de première réponse.
- Durée des générations.
- Débit des tokens.
- Tokens entrants et sortants.
- Erreurs par fournisseur.
- Erreurs de streaming.
- Déconnexions WebSocket.
- Utilisation par utilisateur et modèle.

Les traces doivent permettre de suivre :

```text
Client Request
    |
    v
Authentication
    |
    v
Authorization
    |
    v
Response Creation
    |
    v
Context Construction
    |
    v
Inference Adapter
    |
    v
Model Inference
    |
    v
Event Streaming
```

Les prompts et réponses ne doivent pas être enregistrés dans la télémétrie par défaut.

---

# 16. Déploiement

## 16.1 Services Docker Compose

```text
nginx
frontend
backend
postgresql
```

L'IdP et les serveurs d'inférence sont externes au Compose principal et peuvent être fournis par l'infrastructure existante.

## 16.2 Reverse Proxy

NGINX doit gérer :

- HTTPS.
- Upgrade WebSocket.
- SSE sans buffering inapproprié.
- Timeouts de longue durée.
- Limites de taille des requêtes.
- Propagation des identifiants de corrélation.
- Contrôles d'accès réseau.

Les connexions WebSocket ne doivent pas être interrompues par une configuration de timeout HTTP inadaptée.

## 16.3 Configuration

Exemple :

```toml
[server]
host = "0.0.0.0"
port = 8080

[api]
responses_enabled = true
chat_completions_enabled = true

[streaming]
sse_enabled = true
websocket_enabled = true

[websocket]
max_connections_per_user = 5
max_streams_per_connection = 32
max_active_responses_per_connection = 16

[inference]
default_model = "qwen-local"
```

Ces limites correspondent à des valeurs de configuration initiales proposées et doivent pouvoir être ajustées.

Les paramètres de sécurité, URLs autorisées, clés et secrets ne doivent pas être codés en dur.

---

# 17. Structure du projet

```text
ai-platform/
│
├── apps/
│   ├── web/
│   │   ├── src/
│   │   └── package.json
│   │
│   └── server/
│       ├── src/
│       └── Cargo.toml
│
├── crates/
│   ├── authentication/
│   ├── authorization/
│   ├── api-keys/
│   ├── conversations/
│   ├── responses-protocol/
│   ├── streaming/
│   ├── websocket/
│   ├── inference/
│   ├── model-registry/
│   ├── persistence/
│   ├── usage/
│   └── observability/
│
├── database/
│   └── migrations/
│
├── config/
│
├── deploy/
│   ├── docker/
│   ├── nginx/
│   └── compose.yaml
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── protocol/
│   ├── security/
│   └── e2e/
│
├── Cargo.toml
└── README.md
```

---

# 18. Plan d'implémentation

## Étape 1 — Socle

- Workspace Rust.
- Angular.
- Axum.
- PostgreSQL.
- SQLx.
- Docker Compose.
- Configuration.
- Logs et santé.

## Étape 2 — Authentification

- OIDC générique.
- Sessions.
- RBAC.
- Clés API.
- Provisionnement JIT.
- Révocation.

## Étape 3 — Responses Core

- Modèle canonique des réponses.
- Parsing et validation.
- Persistence des réponses.
- Model Registry.
- Inference Gateway.
- Adaptateurs SGLang/vLLM.
- Continuation via identifiants.

## Étape 4 — HTTP et SSE

- Responses API HTTP.
- Événements SSE conformes.
- Chat Completions.
- Gestion des générations.
- Function Calling côté protocole.
- Background Responses.

## Étape 5 — WebSocket

- Upgrade sécurisé.
- Messages `response.create`.
- Streams nommés.
- Multiplexage.
- FIFO par stream.
- Continuation.
- Reconnexion.
- Gestion des erreurs.
- Backpressure.
- Limites de concurrence.

## Étape 6 — Interface

- Conversations.
- Streaming.
- Sélecteur de modèles.
- Sélecteur de raisonnement.
- Historique.
- Rendu Markdown/code.
- Préférences.
- Administration.
- Branding.

## Étape 7 — Validation

- Tests de protocole.
- Tests des SDK OpenAI.
- Tests WebSocket.
- Tests multi-utilisateurs.
- Tests de sécurité.
- Tests de performance.
- Tests de fonctionnement hors Internet.

---

# 19. Tests de compatibilité

La compatibilité sera validée par une suite de tests automatisés.

## 19.1 Responses HTTP

- Création d'une réponse simple.
- Entrée texte sous forme de chaîne.
- Entrée sous forme d'items structurés.
- Réponse JSON structurée.
- Récupération d'une réponse stockée.
- Suppression d'une réponse.
- Continuation avec `previous_response_id`.
- Vérification du comportement `store=false`.
- Background Response.
- Annulation d'une Background Response.
- Erreur sur paramètre non supporté.

## 19.2 SSE

- Ordre des événements.
- Respect des schémas.
- Streaming des textes.
- Événements de fonction.
- Terminaison normale.
- Échec d'inférence.
- Interruption réseau.
- Annulation applicative.
- Cohérence des objets finaux.

## 19.3 WebSocket

- Handshake authentifié.
- Création de réponse.
- Plusieurs tours successifs.
- Plusieurs `stream_id`.
- Exécution concurrente entre streams.
- Respect FIFO dans un même stream.
- Routage des événements.
- Fork depuis une réponse autorisée.
- Reconnexion.
- Expiration des sessions.
- Révocation des permissions.
- Erreur de réponse introuvable.
- Backpressure.

## 19.4 SDK OpenAI

Exécuter des tests d'intégration avec :

- SDK OpenAI Python.
- SDK OpenAI JavaScript/TypeScript.
- SDK OpenAI .NET.

Les tests devront configurer un `base_url` pointant vers la plateforme privée.

Les cas de WebSocket seront testés avec un SDK ou client prenant explicitement en charge le mode Responses WebSocket.

Les écarts entre la plateforme et l'API OpenAI devront être documentés dans une matrice de compatibilité.

## 19.5 Multi-utilisateur

- Impossible de récupérer une réponse d'un autre utilisateur.
- Impossible de continuer une réponse privée d'un autre utilisateur.
- Impossible de s'abonner à ses flux.
- Impossible de réutiliser ses identifiants de génération.
- Une clé API révoquée perd immédiatement ses droits.
- Une connexion WebSocket est invalidée après révocation effective.

---

# 20. Hors MVP

Les fonctionnalités suivantes restent prévues après la première version.

## Productivité

- Upload de fichiers.
- RAG.
- Recherche documentaire.
- Espaces projets.
- Mémoire personnelle.
- Recherche Web.
- Deep Research.

## Agentique

- Moteur agentique autonome.
- Exécution automatique de fonctions.
- MCP.
- Sous-agents.
- Sandbox de code.
- Workflows complexes.
- Gestion avancée des approbations.
- Mid-turn Steering.
- Préparation et compaction avancées du contexte.

## Multimodal

- Génération d'images.
- Analyse d'images.
- Audio.
- OpenAI Realtime API.
- Voix conversationnelle.
- Vidéo.
- Documents Office.

## Clients lourds

- Tauri.
- Windows.
- Linux.
- macOS.
- Intégration système.
- Agents locaux.

---

# 21. Décisions d'architecture finales

| Domaine | Décision |
|---|---|
| Backend | Rust / Axum |
| Frontend | Angular |
| Base de données | PostgreSQL |
| Authentification | OIDC générique |
| OAuth | Authorization Code + PKCE |
| Sessions Web | Côté serveur |
| SDK/API | Clés API personnelles |
| Interface API principale | Responses API |
| API secondaire | Chat Completions |
| Streaming HTTP | SSE |
| Connexion persistante | WebSocket |
| Multiplexage | `stream_id` |
| Conversations | `previous_response_id` + persistance |
| Inférence | SGLang / vLLM |
| Niveaux de raisonnement | Configurables par modèle |
| Function Calling | Protocole uniquement |
| Agents autonomes | Hors MVP |
| Déploiement | Docker Compose |
| White-Label | Oui |
| Dépendance cloud | Aucune obligatoire |

---

# 22. Définition de terminé — MVP

Le MVP est considéré comme terminé lorsque :

1. Plusieurs utilisateurs peuvent se connecter via un IdP OIDC standard.
2. Les conversations sont privées et persistantes.
3. Plusieurs modèles locaux sont disponibles.
4. Le raisonnement est configurable directement dans l'interface.
5. Les réponses sont accessibles via HTTP et SSE.
6. Une connexion WebSocket peut gérer plusieurs conversations simultanément.
7. La continuation `previous_response_id` fonctionne.
8. Les réponses structurées respectent le profil de compatibilité Responses API.
9. Les SDK OpenAI peuvent utiliser les endpoints HTTP pris en charge.
10. Un client WebSocket compatible peut utiliser le protocole prévu.
11. Les appels de fonctions peuvent être retournés aux clients sans exécution automatique.
12. Les générations en arrière-plan sont récupérables et annulables.
13. Les administrateurs peuvent gérer les modèles et autorisations.
14. Les consommations sont suivies par utilisateur.
15. Les sessions et clés API sont révocables.
16. L'ensemble fonctionne avec un fournisseur d'identité et des modèles hébergés localement.

**Résultat attendu : une plateforme IA professionnelle, multi-utilisateur et 100 % auto-hébergeable, dotée d'une API compatible avec le profil OpenAI Responses défini, exposée en HTTP, SSE et WebSocket, et conçue pour évoluer vers une plateforme agentique complète.**