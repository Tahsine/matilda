# Récapitulatif — Matilda (framework Markdown hypermedia zero-JS)

## Concept général
- Compilateur CLI écrit en **Rust**
- Transforme du Markdown à syntaxe de directives (CommonMark Generic Directives : `:::`, `::`, `:`) en **sites web dynamiques**
- **Zéro JavaScript custom** côté client — toute l'interactivité passe par l'hypermedia
- **Rust** pour le compilateur et la bibliothèque de rendu — seul backend visé pour l'instant. Pas de promesse de support multi-langage : ça ne sera envisagé que si une vraie demande existe. Le **protocole hypermedia**, lui, est du HTTP standard (voir ci-dessous) — n'importe quel backend, dans n'importe quel langage, peut y répondre sans toucher au compilateur

## Public cible
Double public (non-développeurs ET développeurs), résolu par un **modèle de complexité progressive à 3 niveaux** :
1. **Niveau rédacteur** — intentions sémantiques pures, aucun détail technique
2. **Niveau développeur** — ajout d'attributs d'interaction hypermedia (cibles, déclencheurs) directement sur les directives
3. **Niveau compilateur** — validation contre des schémas prédéfinis, génération du HTML enrichi

## Protocole hypermedia (contrat backend)
Le protocole est du **HTTP standard** — un backend, quel que soit son langage, y répond en renvoyant du HTML avec quelques headers, sans jamais toucher au compilateur Rust. Il doit définir trois choses :
- **Négociation de contenu** via header `Accept` (HTML pur vs flux d'événements), avec fallback dégradé
- **Commandes DOM** via headers HTTP normalisés custom (pas dans le balisage HTML)
- **Convention d'erreurs** stricte : les réponses 4xx/5xx sont rendues dans un conteneur dédié, pas silencieusement abandonnées

Comparé à : htmx (headers `HX-*`), Datastar (SSE — événements actuels `datastar-patch-elements` / `datastar-patch-signals`), Hotwire Turbo (`text/vnd.turbo-stream.html`).

## Styling
**CSS Custom Properties natives** (variables CSS scoped inline) — pas de Tailwind JIT ni de micro-CSS utility runtime, pour rester sans dépendance de build externe.

## Animations (contrainte technique actée)
- **View Transitions API** native + **scroll-driven animations**
- Fallback obligatoire via `@supports` pour les navigateurs non compatibles (transition CSS classique / opacité) — dégradation propre, pas de blocage

## Limites du hypermedia — remédiations retenues
- **Web Components légers** pour interactions locales à haute fréquence (drag-and-drop, canvas) — zéro latence, émettent un événement DOM intercepté par le runtime
- **Signaux réactifs déclaratifs** (façon Datastar) pour état d'UI volatile (menus, onglets) sans aller-retour serveur
- **SSE/WebSockets** réservés au push serveur / temps réel (coût : architecture stateful assumée)

## Différenciation vs assemblage manuel (htmx+Markdoc, Datastar+Zola)
- Génération automatique de **stubs/contrats API (OpenAPI)** depuis les directives
- **Vérification à la compilation** de la cohérence des cibles DOM (`hx-target`) sur tout le projet
- **Cache par fragment** — décision : manifeste JSON généré au build (hash par fragment) + header HTTP standard `ETag`, pas de service séparé à maintenir

## Sécurité
- **CSRF standard** (cookie + header) par défaut — préserve la compatibilité CDN/cache statique
- **HMAC par attribut** réservé en option, uniquement pour actions admin critiques à paramètres figés — pas un mécanisme universel (casse le cache CDN, alourdit le HTML)

## Outillage / DX
- **MVP** : diagnostics de parsing précis (type miette/ariadne), hot reload (serveur dev + WebSocket), coloration syntaxique via grammaire **Tree-sitter**
- **v2** : Language Server Protocol (LSP) complet

## Routing
Basé sur l'arborescence de fichiers :
- `pages/about.md` → `/about`
- `pages/blog/[slug].md` → route dynamique
- `pages/docs/[...path].md` → catch-all

## Marché validé
Confirmé par données réelles (adoption htmx, traction Astro, fatigue autour du App Router de Next.js) : niche = **sites à forte densité de contenu, documentation, outils CRUD/admin** — pas les applications à état client complexe (éditeurs visuels, dashboards riches).
