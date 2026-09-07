# Planes de implementación

Los planes paso a paso con los que se construyó el RAG de FAQs. Vivían en
`docs/superpowers/plans/` dentro de los repos de código y se movieron acá antes
de mergear a `main`: son documentación del workspace, que es para lo que existe
este repo, y en el PR del backend representaban el **45% del diff** — un
revisor pasaba por casi tanto scaffolding como código.

## Qué son, y qué NO son

Son la **coreografía de ejecución** derivada de los PRDs de [`../`](../): qué
archivo tocar, en qué orden, con qué test. Están escritos para un agente que
ejecuta tarea por tarea, por eso abren con `For agentic workers: REQUIRED
SUB-SKILL`.

**Los checkboxes no significan nada.** Están todos sin tildar aunque el trabajo
esté hecho y desplegado: el ejecutor nunca los volvía a escribir. No los leas
como estado.

**Para saber qué hace el sistema hoy, no mires acá.** Mirá, en este orden:

1. Los **PRDs** en [`../`](../) — la especificación y el porqué de cada
   decisión de producto.
2. Los **comentarios del código** — el porqué de cada decisión técnica, al lado
   de la línea que la toma. Es donde vive lo que más rápido se desactualiza en
   un documento aparte.
3. Estos planes — solo si necesitás reconstruir *cómo se llegó* a algo.

## Lo que sí guardan y no está en otro lado

Razonamiento intermedio que no llegó a los PRDs. Por ejemplo, en
`2026-09-05-faq-import-phases-deh.md`, por qué las fases F y G del PRD 4 no
necesitaban backend: las rutas de contexto de tenant ya le daban el rol `admin`
al cliente, así que lo único que faltaba era la pantalla.

## De dónde vino cada uno

| Archivo | Repo de origen |
|---|---|
| `2026-09-01-rag-knowledge-layer.md` | backend |
| `2026-09-02-faq-content-ingestion-phase1..4.md` | backend |
| `2026-09-03-faq-admin-api.md` | backend |
| `2026-09-05-faq-bulk-import-api.md` | backend |
| `2026-09-05-faq-import-phases-deh.md` | backend |
| `2026-09-03-faq-admin-ui.md` | frontend |
| `2026-09-05-faq-bulk-import-ui.md` | frontend |

Siguen en el historial de git de esos repos, en los commits anteriores al que
los borró.
