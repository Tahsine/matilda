# Roadmap — Matilda (framework Markdown hypermedia zero-JS)

## Phase 1 — MVP noyau
- Compilateur Rust : parsing des directives génériques CommonMark (`:::`, `::`, `:`)
- AST → HTML avec CSS Custom Properties (styling natif)
- Routing basé sur l'arborescence de fichiers (statique)
- Protocole hypermedia v1 : fragments HTML, négociation via header `Accept`, commandes DOM via headers custom, convention de gestion d'erreurs 4xx/5xx
- Sécurité : CSRF standard (cookie/header)
- View Transitions API + fallback `@supports`
- Outillage : diagnostics de parsing (type miette/ariadne) + hot reload

## Phase 2 — Syntaxe développeur & différenciation
- Niveau 2 de la syntaxe : attributs hypermedia sur les directives (cibles, déclencheurs)
- Génération de stubs OpenAPI depuis les directives
- Validation à la compilation des cibles DOM (`hx-target`) cassées, à travers tout le projet
- Cache par fragment (précalcul du hash ETag unifié dans le manifeste JSON — même mécanisme que le hash runtime décidé en Phase 1)
- Coloration syntaxique via grammaire Tree-sitter

## Phase 3 — Temps réel & état local (feature différée)
- SSE/WebSockets pour le push serveur temps réel
- Couche de signaux réactifs déclaratifs pour l'état d'UI volatile côté client (sans aller-retour réseau)
- Web Components légers comme échappatoire pour les interactions locales à haute fréquence (drag-and-drop, canvas)
- C'est cette phase qui ouvre la porte aux cas d'usage type dashboard complexe — volontairement non prioritaire au MVP

## Phase 4 — Maturité & écosystème
- Language Server Protocol (LSP) complet : autocomplétion, saut vers définition, refactoring
- HMAC optionnel pour actions admin critiques à paramètres figés
- Écosystème de schémas/composants partagés entre projets
