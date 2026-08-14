# Phase 1 — Spécification détaillée (MVP noyau) — projet Matilda

> Note : ce document a été rédigé avant que le nom du projet soit tranché. Les mentions `fw-core` / `fw-cli` / `fw build` ci-dessous désignent maintenant **`matilda-core`** / **`matilda-cli`** / **`matilda build`** — voir le guide de renommage fourni séparément pour la conversion du code déjà écrit.

## 1. Compilateur Rust — pipeline interne

**Crates de base envisagés :**
- Parsing CommonMark + extension directives : `markdown` (port Rust de micromark, supporte les extensions façon remark-directive) ou `pulldown-cmark` avec un tokenizer de directives en pré-passe
- Grammaire des attributs `{ cols=3 gap=medium }` : `winnow` ou `nom` (parser combinators)
- Sanitization HTML (allowlist de balises/attributs) : `ammonia`
- Diagnostics d'erreurs : `miette` (spans, suggestions contextuelles)
- Parallélisation multi-fichiers : `rayon`

**Pipeline en 4 passes :**
1. **Tokenisation** — détection des marqueurs `:::` (conteneur), `::` (bloc), `:` (inline) avant l'évaluation Markdown standard
2. **Construction AST** — les directives deviennent des nœuds conteneurs opaques ; les attributs `{...}` sont attachés comme metadata brute (pas interprétés comme texte)
3. **Résolution sémantique** — une table de correspondance directive → balise HTML5 + attributs hypermedia + variables CSS (ex: `layout-grid` → `<div class="fw-layout-grid" style="--cols:3">`)
4. **Génération de code** — émission du HTML sanitizé (allowlist stricte de balises), injection des variables CSS et du protocole hypermedia

### Grammaire des attributs `{ }` — option A retenue (CommonMark-directives / Pandoc-style)

```
attribute-block ::= '{' WS? attribute (WS attribute)* WS? '}'
attribute       ::= id-shorthand | class-shorthand | key-value | flag
id-shorthand    ::= '#' identifier
class-shorthand ::= '.' identifier
key-value       ::= identifier '=' (quoted-value | bare-value)
flag            ::= identifier
quoted-value    ::= '"' (escaped-char | [^"\\])* '"'
bare-value      ::= [A-Za-z0-9_.\-]+
escaped-char    ::= '\\' ('"' | '\\')
identifier      ::= [A-Za-z][A-Za-z0-9_\-]*
```

Règles :
- `#id` — id de l'élément **courant** (celui que la directive définit). Un seul autorisé ; le dernier gagne si répété (tolérant, pas une erreur bloquante)
- `.class` — classe CSS ajoutée à l'élément courant, cumulable (`.grid .highlighted` → deux classes)
- `key="valeur avec espaces"` — guillemets obligatoires si la valeur contient un espace, `{`, `}` ou `"` ; échappement par `\"` à l'intérieur
- `key=valeur` (sans guillemets) — uniquement pour tokens simples (alphanumériques, `-`, `_`, `.`) : `cols=3`, `delay=100ms`
- `flag` (clé seule, sans `=`) — attribut booléen, présence = `true` (ex : `disabled`, `strict`)
- Le bloc `{ }` est optionnel si la directive n'a besoin d'aucune configuration : `::divider`

**Typage : différé à la résolution sémantique (passe 3), pas dans la grammaire.** La grammaire produit toujours des paires clé-valeur en chaînes de caractères — c'est le schéma propre à chaque nom de directive (`layout-grid`, `animate`...) qui déclare le type attendu par attribut (entier, durée, énumération...) et effectue la coercion/validation à la passe 3. Une valeur qui ne correspond pas au schéma déclenche un diagnostic `miette` (section 8), pas une erreur de grammaire générique — messages bien plus utiles pour l'auteur.

**La cible hypermedia (`target=`) reste un attribut `key=value` standard**, pas un shorthand dédié — le shorthand `#`/`.` de l'option A s'applique à l'élément courant, jamais à un élément distant (section 5).

## 2. Frontière de confiance — mode Template vs mode Contenu

La bibliothèque de rendu expose deux modes explicites à l'appel — c'est la réponse à la difficulté #1 identifiée plus haut.

**Mode Template (confiance totale)**
- Utilisé pour : fichiers `.md` du dépôt compilés au build (SSG), et tout appel in-process où la source Markdown est écrite par un développeur
- Directives autorisées : l'ensemble complet, y compris les directives d'action (`action=`, `target=`, `trigger=`)

**Mode Contenu (non fiable)**
- Utilisé pour : tout Markdown fourni à l'exécution par une source externe (champ CMS, commentaire, post utilisateur) et interpolé dans un fragment
- Distinction **binaire par nature de directive**, appliquée à la passe de résolution sémantique (passe 3 du pipeline, section 1) — pas un allowlist par nom à maintenir :
  - **Directives structurelles/présentation** (layout, typographie, animation) → toujours autorisées ; le pire cas est un rendu visuel cassé, pas une faille
  - **Directives d'action** (toute directive portant `action=`, `target=`, `trigger=`, ou toute référence à une URL/sélecteur en dehors du fragment lui-même) → toujours rejetées en mode Contenu
- **Dégradation par défaut : douce.** Une directive d'action détectée en mode Contenu est retirée, le reste du fragment est rendu normalement, un avertissement est loggé — pas d'échec dur du rendu (cohérent avec la difficulté #2 : le runtime ne doit pas planter sur du contenu imprévisible). Un mode `strict` est exposé pour les intégrations qui préfèrent un rejet total avec erreur (ex : aperçu CMS avant publication, où l'auteur doit être averti immédiatement)
- Cette frontière se situe **en amont** de la sanitization HTML (`ammonia`) — c'est une couche sémantique supplémentaire, pas un remplacement ; `ammonia` reste le filet de sécurité en sortie

**Esquisse d'API :**
```rust
enum RenderMode {
    Template,
    Content { strict: bool },
}

fn render(source: &str, mode: RenderMode) -> Result<RenderedFragment, RenderError>
```

## 3. Styling — CSS Custom Properties

- Chaque directive résolue injecte des variables CSS scoped inline (`style="--cols: 3; --gap: var(--space-md)"`)
- Une feuille de style statique unique (`fw-base.css`) est émise une fois par build, contenant les définitions `.fw-*` (grid, flex, etc.) et référencée par toutes les pages
- `_theme.yml` (tokens globaux : couleurs, typographie, spacing) compilé en `:root { --color-primary: ... }` injecté dans le `<head>`

## 4. Routing par arborescence de fichiers

| Source | Route | Type |
|---|---|---|
| `pages/index.md` | `/` | Statique |
| `pages/about.md` | `/about` | Statique |
| `pages/blog/[slug].md` | `/blog/:slug` | Dynamique |
| `pages/docs/[...path].md` | `/docs/*` | Catch-all |

**Décision : (b) dès la Phase 1** — le pipeline de rendu (passes 2 à 4) est construit comme une **bibliothèque Rust réutilisable**, pas enfermée dans le CLI. Le CLI (SSG) l'utilise pour le build ; le rendu runtime pour le contenu dynamique (cas CRUD/admin, cœur du marché validé) l'utilise aussi.

**Périmètre backend clarifié — un seul langage engagé, pas une promesse d'agnosticisme :**
- **Backend Rust uniquement pour la Phase 1** — dépendance de crate directe, appel in-process de la bibliothèque de rendu. Pas de réseau, pas d'auth interne, pas de service à déployer. C'est le seul backend pour lequel la **compilation runtime des directives** (mode Contenu) est prévue dans cette phase.
- **Bindings Python (`PyO3`) ou service HTTP réseau** — envisageables plus tard, mais pas planifiés ni promis : uniquement si une demande réelle se manifeste. Pas de roadmap figée dessus.

**Important, à distinguer clairement :** cette section ne concerne que la **compilation runtime des directives** (mode Contenu — du Markdown avec notre syntaxe, interprété à l'exécution). Le **protocole hypermedia** (section 5) est une toute autre chose : c'est du **HTTP standard**, indépendant du langage du backend et indépendant du compilateur Rust — un backend qui se contente de répondre aux requêtes hypermedia (formulaires, boutons) en HTML n'a besoin ni de Rust, ni de bindings, ni d'aucune brique de ce projet. Voir section 5.

La frontière de confiance (mode "template" écrit par le dev vs mode "contenu" dynamique non fiable) doit être résolue au niveau de la bibliothèque de rendu elle-même, puisque même l'intégration Rust in-process y est exposée dès qu'elle traite du contenu utilisateur.

## 5. Protocole hypermedia v1

### Déclaration côté client (compilée depuis la directive, portée par l'élément)

Chaque directive d'action compile vers des attributs `data-*` sur l'élément déclencheur — pas besoin d'un aller-retour serveur pour savoir où agit une interaction :

```html
<button data-hm-action="/api/todo/add" data-hm-method="POST"
        data-hm-target="#todo-list" data-hm-swap="beforeend"
        data-hm-trigger="click">Ajouter</button>
```

- `data-hm-method` accepte les vrais verbes HTTP (`PUT`, `DELETE`, `PATCH` inclus) — le runtime passe par `fetch()`, pas par la soumission native d'un `<form>`, donc pas de hack "method override" nécessaire ici
- `data-hm-trigger` par défaut : `submit` pour les formulaires, `click` pour le reste

### Requête émise par le runtime

- Méthode/URL : depuis `data-hm-method` / `data-hm-action`
- Headers : `X-Hypermedia-Request: true` (signale une requête fragment — plus fiable qu'une négociation via `Accept`, qu'un navigateur envoie de toute façon en navigation normale) ; `X-Hypermedia-Target: #selector` (cible d'origine, pour la logique serveur) ; `X-CSRF-Token` (section 6)
- Corps : `application/x-www-form-urlencoded` ou `multipart/form-data` selon le contenu

### Réponse serveur — headers de dépassement (override)

Les attributs `data-*` définissent le comportement **par défaut**. Les headers de réponse permettent au backend de le **dépasser dynamiquement** quand la logique métier l'exige (rediriger après succès, recibler une erreur ailleurs que la cible de succès) :

- `X-Hypermedia-Target` / `X-Hypermedia-Swap` — écrase la cible/stratégie déclarées côté client pour cette réponse précise
- `X-Hypermedia-Trigger: event-name` — déclenche un événement DOM personnalisé après le swap
- `X-Hypermedia-Redirect: /nouveau-chemin` — redirection douce : mise à jour de l'URL (History API) + navigation hypermedia vers la nouvelle route, enveloppée dans la même View Transition — pas de rechargement complet

### Gestion d'erreurs 4xx/5xx

- Le corps reste du HTML (jamais un abandon silencieux)
- `X-Hypermedia-Error-Target` optionnel : si absent, l'erreur est rendue dans la cible **normale** de la requête — comportement par défaut sûr, l'utilisateur voit toujours quelque chose

### Amélioration progressive (no-JS) — décision de phasage

Le fallback natif (option 2) est plus fidèle à la philosophie du projet, mais double la sortie générée par directive — trop de charge pour un MVP déjà chargé (bibliothèque de rendu + deux modes de confiance + protocole + cache).

**Décision : Phase 1 sans fallback natif** (option 1 — le runtime doit charger pour que les interactions fonctionnent), **mais l'AST des directives réserve dès maintenant les champs sémantiques nécessaires** (`action`, `method`, `target`) de façon suffisamment structurée pour qu'ajouter la génération du `<form>`/`<a>` natif en Phase 2 ne touche que l'étape de codegen (passe 4) — pas le parsing ni la résolution sémantique (passes 1 à 3). Même logique que le champ `etag` réservé pour le cache : pas de rupture d'API plus tard.

## 6. Sécurité — CSRF standard

- Cookie `csrf_token` non-httpOnly (lisible par le runtime hypermedia du framework)
- Le petit runtime JS vendored (le seul JS du système, pas écrit par l'utilisateur) lit ce cookie et l'attache automatiquement en header `X-CSRF-Token` sur chaque requête mutable
- Validation double-submit côté serveur (cookie vs valeur en session)

## 7. View Transitions + fallback

- Le swap hypermedia est enveloppé dans `document.startViewTransition()` si l'API est disponible
- CSS `@supports (view-transition-name: root)` pour les règles avancées, `@supports not (...)` pour un fallback en `transition: opacity`

## 8. Outillage — diagnostics + hot reload

- Sous-commande `fw dev` : watcher de fichiers (`notify`) sur `pages/` et `_theme.yml`
- Recompilation incrémentale du fichier modifié, push du nouveau fragment au navigateur via WebSocket
- Réutilisation du même mécanisme de swap hypermedia pour le hot reload (pas de logique séparée)
- Erreurs de parsing affichées avec `miette` : ligne, colonne, extrait de la directive mal formée, suggestion de correction

## 9. Fondation du cache — ETag unifié build & runtime

Un seul mécanisme sous-jacent, pas deux systèmes séparés pour le statique et le dynamique :

**ETag = hash(version du compilateur + version du schéma de directives + contenu source)**, calculé avec un hash rapide (`BLAKE3` plutôt qu'un hash cryptographique lourd type SHA-256 — le coût CPU par fragment devient négligeable, de l'ordre de la microseconde pour un fragment typique).

- **Au build (SSG, mode Template)** : ce hash est calculé une fois et stocké dans le manifeste JSON (Phase 2) — c'est une précalcul du même mécanisme, pas un système différent
- **En runtime (mode Contenu)** : ce même hash est calculé à la demande, sur la source reçue, **avant** de lancer le rendu complet. S'il correspond au `If-None-Match` envoyé par le client, réponse `304` sans re-rendre — le rendu n'est déclenché que si le contenu a réellement changé
- **Salage par version du compilateur** : indispensable des deux côtés — sans ça, une mise à jour du compilateur (changement de mapping directive → HTML) laisserait servir des fragments obsolètes depuis un cache qui ignore le changement de sémantique

**Sur le CDN** : le contenu dynamique n'est pas "non cacheable", il est cacheable **en validation** plutôt qu'**en durée** — `ETag` + `must-revalidate` fonctionne très bien en périphérie CDN ; la requête remonte juste comparer un hash côté origine (bon marché), pas nécessairement re-rendre. Net gain par rapport à toujours re-rendre.

**Timing** : le champ `etag: String` fait partie de `RenderedFragment` dès la Phase 1 (pour ne pas casser l'API plus tard), mais la couche d'optimisation complète (manifeste au build, intégration `If-None-Match`/`304` côté service) reste programmée pour la Phase 2, comme prévu dans la roadmap.
