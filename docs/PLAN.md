# WoW Macro Assistant — Plan de projet

> Addon World of Warcraft (Retail + Classic) + application web avec authentification Battle.net.
> L'utilisateur décrit en langage naturel la macro qu'il veut, un modèle (Claude) l'interprète,
> pose des questions si besoin, et renvoie une macro prête à importer en jeu.
>
> État des références API : **21 septembre 2026**. Hypothèse retenue : « retail et forcer » = Retail **et Classic**
> (MoP Classic, Classic Era, Anniversary). À confirmer.

---

## 0. Contrainte fondamentale à comprendre avant tout

**Un addon WoW ne peut pas faire de requête réseau.** Le sandbox Lua de Blizzard n'expose ni HTTP, ni socket,
ni accès disque (seules les SavedVariables sont écrites, et uniquement au logout / `/reload`).

Conséquence : le « dialogue » entre le joueur et le modèle **ne peut pas se faire directement dans le jeu**.
Deux architectures existent dans l'écosystème, et c'est ce que fait AskMrRobot :

| Modèle | Fonctionnement | Exemple |
|---|---|---|
| **Copier / coller** (MVP) | L'addon produit une chaîne d'export → le joueur la colle sur le site → le site renvoie une chaîne d'import → le joueur la colle dans l'addon | AskMrRobot, WeakAuras (wago.io) |
| **Application compagnon** (phase 3) | Un petit programme desktop lit/écrit les SavedVariables et fait les appels réseau ; le joueur fait `/reload` | WeakAuras Companion, Raider.IO client |

Le plan ci-dessous part du modèle copier/coller pour le MVP (zéro installation en plus de l'addon),
avec l'application compagnon en option ultérieure pour fluidifier.

Le flux utilisateur cible :

```
[En jeu]  /macroai  →  fenêtre : "Décris ta macro"  →  l'addon ajoute le contexte (classe, spé, niveau,
          sorts connus, version du jeu)  →  chaîne d'export (Ctrl+C)
[Site]    Connexion Battle.net  →  coller la chaîne  →  chat avec le modèle (questions/réponses)
          →  macro validée  →  chaîne d'import (Ctrl+C)
[En jeu]  /macroai import  →  coller  →  l'addon crée la macro via CreateMacro()
```

---

## 1. Références API Blizzard (vérifiées le 21/09/2026)

### 1.1 Battle.net OAuth 2.0 (identification de l'utilisateur)

Portail développeur : https://develop.battle.net (création du client OAuth, `client_id` / `client_secret`, redirect URIs).

| Élément | Valeur |
|---|---|
| Authorize | `https://oauth.battle.net/authorize` |
| Token | `https://oauth.battle.net/token` |
| UserInfo | `https://oauth.battle.net/userinfo` → `{ sub, id, battletag }` |
| Check token | `https://oauth.battle.net/oauth/check_token` |
| Chine | hôte séparé `oauth.battlenet.com.cn` |
| Scopes | `openid` (identité seule), `wow.profile` (personnages WoW), `sc2.profile`, `d3.profile` |
| Flows | *Authorization code* (utilisateur) et *client credentials* (données de jeu, serveur à serveur) |
| Durée token | 24 h |

Ce que l'on obtient sans scope supplémentaire : **l'identifiant numérique du compte (`id`/`sub`) et le BattleTag**.
Pas d'e-mail, pas de nom. L'`id` est la clé d'identité stable de notre base utilisateurs.

Avec `wow.profile` on peut lister les personnages du compte (voir 1.2), ce qui permet de **vérifier qu'une chaîne
d'export collée appartient bien au compte connecté** (anti-partage de quota).

### 1.2 Game Data API et Profile API (WoW)

Hôte : `https://{region}.api.blizzard.com` (`us`, `eu`, `kr`, `tw` ; `cn` sur `gateway.battlenet.com.cn`).
Namespace en en-tête `Battlenet-Namespace:` ou en query `?namespace=`.

| Famille | Namespaces | Auth |
|---|---|---|
| Retail | `static-{region}`, `dynamic-{region}`, `profile-{region}` | client credentials (profile-user : token utilisateur) |
| Classic (progression, MoP Classic aujourd'hui) | `static-classic-{region}`, `dynamic-classic-{region}`, `profile-classic-{region}` | idem |
| Classic Era | `static-classic1x-{region}`, `dynamic-classic1x-{region}`, `profile-classic1x-{region}` | idem |

Endpoints utiles pour ce projet :

| Endpoint | Usage |
|---|---|
| `GET /profile/user/wow` (token utilisateur + `wow.profile`, ns `profile-{region}`) | Liste des personnages du compte → liaison compte ↔ chaîne d'export |
| `GET /profile/wow/character/{realm}/{name}` + `/specializations`, `/equipment` | Contexte serveur si l'export ne contient pas tout |
| `GET /data/wow/playable-class/index`, `/playable-specialization/{id}` | Référentiel classes / spés |
| `GET /data/wow/talent-tree/index`, `/talent-tree/{id}/playable-specialization/{specId}` | Talents (retail) |
| `GET /data/wow/search/spell?name.en_US=...`, `GET /data/wow/spell/{id}` | **Validation des noms de sorts** générés par le modèle |
| `GET /data/wow/item/{id}`, `/search/item` | Validation des objets (`/use item:xxxx`) |
| `GET /data/wow/realm/index` | Slugs de royaumes |

Limites Blizzard : **36 000 requêtes / heure et 100 requêtes / seconde par client API** (429 au-delà),
50 clients max par compte développeur. ⇒ cache agressif des données statiques (sorts, classes) côté serveur.

Conditions d'utilisation à respecter : *Blizzard Developer API Terms of Use* (attribution, pas de revente de
données Blizzard brutes) et *UI Add-On Development Policy* : **l'addon lui-même doit être gratuit** ; la
monétisation ne peut porter que sur le service web (c'est exactement le modèle AskMrRobot).

### 1.3 API Lua côté addon (ce que l'addon peut faire)

| Besoin | API |
|---|---|
| Détecter la version du jeu | `WOW_PROJECT_ID` vs `WOW_PROJECT_MAINLINE`, `WOW_PROJECT_CLASSIC`, `WOW_PROJECT_MISTS_CLASSIC`… ; `GetBuildInfo()` |
| Identité | `BNGetInfo()` → BattleTag du joueur ; `UnitName("player")`, `GetRealmName()`, `GetNormalizedRealmName()` |
| Classe / spé / niveau | `UnitClass`, `UnitLevel`, `GetSpecialization()` + `GetSpecializationInfo()` (retail), talents via `C_ClassTalents` / `C_Traits` (retail) ou `GetTalentInfo` (Classic) |
| Sorts connus | `C_SpellBook.*` (retail ≥ 11.0), `GetSpellBookItemName` (Classic) → **liste exacte des noms de sorts** envoyée au modèle |
| Objets équipés / sac | `GetInventoryItemID`, `C_Container.GetContainerItemID` |
| Barres d'action | `GetActionInfo(slot)` (pour proposer de remplacer un bouton) |
| **Créer / modifier la macro** | `CreateMacro(name, icon, body, perCharacter)`, `EditMacro`, `GetMacroInfo`, `GetNumMacros()` |
| Presse-papier | pas d'API : on affiche un `EditBox` avec le texte pré-sélectionné (Ctrl+C / Ctrl+V manuel) |

Limites des macros (identiques Retail / Classic modernes) : **corps ≤ 255 caractères**, nom ≤ 16 caractères,
**120 macros de compte + 18 par personnage**. `CreateMacro` / `EditMacro` sont **bloqués en combat**.

### 1.4 Versions d'Interface (fichiers TOC)

Le client supporte les TOC par version (`MonAddon_Mainline.toc`, `_Mists.toc`, `_Vanilla.toc`, `_TBC.toc`,
`_Wrath.toc`) **et** la directive multi-versions `## Interface: 120200, 50504, 11509, 20506`.

| Client (sept. 2026) | Suffixe TOC | Interface (à confirmer en jeu) |
|---|---|---|
| Retail — Midnight 12.2.x | `_Mainline` | `1202xx` |
| Mists of Pandaria Classic 5.5.x | `_Mists` | `5050x` |
| Classic Era / Hardcore / SoD 1.15.x | `_Vanilla` | `1150x` |
| Anniversary (TBC) 2.5.x | `_TBC` | `2050x` |
| Titan Reforged (Wrath) | `_Wrath` | `3800x` |

Règle : ne jamais figer ces numéros à la main. Commande en jeu `/dump select(4, GetBuildInfo())` et, en CI,
l'action GitHub **p3lim/toc-interface-updater** (met à jour tous les TOC automatiquement, PTR/beta inclus).

Différences Retail / Classic à gérer dans le générateur de macros : noms de sorts différents, conditionnels
absents ou différents (ex. `[known:…]`, `@cursor`, `/cast` de rangs en Classic Era `Rank 3`), pas de
spécialisations en Classic Era, `#showtooltip` disponible partout, `/castsequence`, `/stopcasting`, `/cancelaura`.

---

## 2. Architecture cible

```
┌──────────────────────┐      chaîne d'export        ┌──────────────────────────────┐
│  Addon Lua (gratuit) │ ───────(copier/coller)────▶ │  Web app (Next.js/TS)         │
│  - UI /macroai       │                             │  - Auth Battle.net (OAuth)    │
│  - collecte contexte │ ◀──────(copier/coller)───── │  - Chat macro (Claude)        │
│  - import + CreateMacro│     chaîne d'import        │  - Linter/validateur de macro │
└──────────────────────┘                             │  - Quotas & facturation       │
                                                     └───────┬───────────────┬───────┘
                                                             │               │
                                              ┌──────────────▼──┐   ┌────────▼───────────┐
                                              │ Postgres (Supabase)│  │ Claude API (Opus 5) │
                                              │ users, quotas,   │   │ + Blizzard Game Data│
                                              │ historique       │   │   (cache Redis)     │
                                              └──────────────────┘   └────────────────────┘
```

### 2.1 Addon (Lua)

- **Commande** `/macroai` → fenêtre avec : zone de texte (demande), choix « macro de compte / de personnage »,
  bouton *Générer la chaîne*, bouton *Importer*.
- **Contexte collecté automatiquement** (c'est ce qui fera la qualité des macros) : version du jeu, locale,
  classe, spé, niveau, race, talents (chaîne d'export retail), **liste des sorts connus avec leurs noms exacts
  dans la langue du client**, objets équipés/trinkets, macros existantes (noms), BattleTag, personnage-royaume.
- **Encodage** : `LibSerialize` + `LibDeflate` + `EncodeForPrint` (même chaîne d'outils que WeakAuras ; décodable
  en JS). Préfixe versionné `MAI1!…` pour l'évolutivité.
- **Import** : décode, vérifie la longueur (≤ 255), la limite de macros, hors combat ; propose *Créer* /
  *Remplacer une macro existante* / *Placer sur la barre d'action* (`PickupMacro`). Affiche la macro et
  l'explication.
- **Compatibilité** : un dossier, TOC multi-version, code conditionné par `WOW_PROJECT_ID`, tests sur Retail,
  MoP Classic, Classic Era, Anniversary.
- **Libs** : Ace3 (AceAddon, AceGUI, AceDB, AceLocale) — standard, éprouvé sur toutes les versions.
- **Packaging** : BigWigsMods/packager → CurseForge, Wago Addons, WoWInterface. Localisation FR/EN au minimum.

### 2.2 Web app (TypeScript)

- **Stack** : Next.js (App Router) + Auth.js (provider Battle.net intégré) + Postgres via Supabase
  (connecteur déjà disponible) + Redis/Upstash pour rate-limit et cache Blizzard + hébergement Vercel ou OVH.
- **Auth** : bouton « Se connecter avec Battle.net » (scopes `openid wow.profile`). On stocke `bnet_id`,
  `battletag`, région, date, et le refresh des personnages (`/profile/user/wow`) en cache 24 h.
- **Page principale** : coller la chaîne d'export → panneau de contexte lisible (classe/spé/version) →
  chat. Le modèle répond soit par des **questions de clarification** (choix cliquables + texte libre), soit
  par une **macro finale** : corps, nom, icône, explication ligne par ligne, avertissements, chaîne d'import.
- **Historique** : mes macros, re-générer, partager un lien public (optionnel).
- **Sans addon** : formulaire manuel (version, classe, spé) pour tester le service, avec qualité moindre.

### 2.3 Moteur IA

- **Modèle** : Claude Opus 5 (`claude-opus-5`) via `@anthropic-ai/sdk`, thinking adaptatif, streaming vers
  l'UI, *prompt caching* sur le référentiel de macros (gros bloc stable) ; Claude Sonnet 5 en option pour un
  tier gratuit moins coûteux si les evals montrent une qualité suffisante.
- **Sorties structurées** (`output_config.format`, schéma JSON) :
  ```json
  {
    "status": "macro" | "question",
    "questions": [{ "text": "...", "choices": ["..."] }],
    "macro": { "name": "...", "icon": "INV_MISC_QUESTIONMARK", "body": "...", "per_character": false },
    "explanation": "...",
    "warnings": ["..."],
    "confidence": 0.0
  }
  ```
- **System prompt / base de connaissances par version** : syntaxe des commandes slash, conditionnels
  (`[@mouseover,harm,nodead]`, `[mod:shift]`, `[combat]`, `[form:1]`, `[spec:2]`, `[known:…]`…), pièges
  (limite 255, pas de logique temporelle, `/castsequence` reset, impossibilité de choisir automatiquement
  selon les CD), différences Retail / Classic. Chaque version du jeu a son propre fichier de référence.
- **Outils (tool use)** côté serveur : `lookup_spell(name, flavor)` (Blizzard Game Data + liste des sorts
  connus envoyée par l'addon), `lookup_item`, `lint_macro(body)`.
- **Validateur déterministe** (hors modèle, dans `packages/macro-lint`) : longueur, commandes inconnues,
  conditionnels invalides pour la version, sort non connu du personnage → renvoyé au modèle pour correction
  avant d'afficher quoi que ce soit. C'est la garantie « la macro marche ».
- **Sécurité** : la chaîne d'export est **de la donnée, pas des instructions** (injection de prompt) ;
  l'utilisateur ne peut pas modifier le system prompt ; sorties limitées au schéma.
- **Catégories** : champ `category` dès le début (`macro` seule au MVP), pour ajouter plus tard
  `keybinds`, `weakaura`, `addon-config`, `rotation-help`.
- **Évaluation** : jeu de ~100 demandes (FR/EN, Retail/Classic, cas ambigus) avec macros attendues ;
  score = linter OK + spell names valides + jugement modèle. Aucun changement de prompt sans passer l'eval.

### 2.4 Identification, quotas, anti-abus

| Mécanisme | Détail |
|---|---|
| Identité | `bnet_id` (stable) + BattleTag ; jamais l'e-mail (Blizzard ne le fournit pas) |
| Liaison export ↔ compte | Le personnage-royaume de la chaîne doit figurer dans `/profile/user/wow` du compte connecté ; sinon avertissement puis blocage au-delà de N échecs |
| Quota par utilisateur | Compteur glissant 24 h en Redis (`rate:{bnet_id}:{jour}`) : ex. **Gratuit 5 macros/jour, ~15 tours de chat** ; **Premium 100/jour** ; limites hebdo pour lisser |
| Quota technique | Rate-limit par IP (anti-scripts), taille max de la demande (1 000 caractères), max 6 tours de questions par session, `max_tokens` borné |
| Budget | Coût suivi par requête (`usage`), coupure « disjoncteur » global journalier, alerte |
| Blizzard | Cache statique 7 jours, cache profil 24 h, file d'attente 100 rps, respect des 36 000 req/h |
| Paiement | Stripe (abonnement mensuel) ou Patreon/Ko-fi au démarrage ; l'addon reste gratuit (politique Blizzard) |
| RGPD | Données minimales, export/suppression de compte, rétention de l'historique paramétrable, politique de confidentialité |

---

## 3. Structure du dépôt (monorepo)

```
Wow-plugin/
├── addon/                      # MacroAI (Lua)
│   ├── MacroAI.toc             # ## Interface: multi-versions (mis à jour en CI)
│   ├── MacroAI_Mainline.toc / _Mists.toc / _Vanilla.toc / _TBC.toc
│   ├── Libs/                   # Ace3, LibSerialize, LibDeflate (via packager .pkgmeta)
│   ├── Core.lua, Context.lua, Export.lua, Import.lua, UI.lua
│   └── Locales/ (frFR, enUS)
├── web/                        # Next.js + Auth.js + Supabase
│   ├── app/ (login, generate, history, account, api/)
│   ├── lib/ (anthropic, blizzard, quotas, codec)
│   └── prompts/ (macro-retail.md, macro-mists.md, macro-era.md, …)
├── packages/
│   ├── codec/                  # encode/decode chaîne d'export-import (TS, compatible LibDeflate)
│   └── macro-lint/             # validateur déterministe de macros par version
├── evals/                      # jeu de tests du générateur
├── companion/                  # (phase 3) app Tauri : sync SavedVariables
└── docs/
```

---

## 4. Feuille de route

### Phase 0 — Cadrage & preuve de concept (1 semaine)
- [ ] Créer le client OAuth sur develop.battle.net (redirect `http://localhost:3000/api/auth/callback/battlenet`).
- [ ] Choisir le nom (vérifier la disponibilité sur CurseForge / Wago / nom de domaine).
- [ ] Script Node : coller une demande + contexte fictif → Claude → macro + linter. Valider la qualité sur 20 cas.
- [ ] Prototype Lua minimal : fenêtre, `CreateMacro` depuis une chaîne collée, sur Retail et MoP Classic.

### Phase 1 — MVP (4 à 6 semaines)
**Addon**
- [ ] Collecte de contexte (classe/spé/niveau/talents/sorts connus) Retail + Classic.
- [ ] Export (LibSerialize + LibDeflate) et Import (création, remplacement, limites, hors combat).
- [ ] UI AceGUI, `/macroai`, minimap/bouton optionnel, localisation FR/EN.
- [ ] Packaging CurseForge/Wago + CI (toc-interface-updater, luacheck).

**Web**
- [ ] Auth Battle.net, tables `users`, `sessions`, `requests`, `macros`, `quotas`.
- [ ] Décodeur de chaîne, page *Générer* avec chat streaming, questions cliquables, chaîne d'import.
- [ ] Prompts par version + linter + validation des sorts (liste addon, puis Game Data API).
- [ ] Quotas gratuits, rate-limit IP, disjoncteur budget, journal des coûts.
- [ ] Pages légales (CGU, confidentialité, mention Blizzard), page d'aide « comment ça marche ».

### Phase 2 — Qualité & confiance (3 à 4 semaines)
- [ ] Jeu d'évaluation (100+ cas), tableau de bord de qualité, boucle d'amélioration des prompts.
- [ ] Liaison export ↔ compte via `/profile/user/wow`, détection de partage de quota.
- [ ] Historique, favoris, partage public de macros, re-génération après patch.
- [ ] Bêta fermée (Discord), télémétrie opt-in (taux d'import réussi, macros modifiées à la main).

### Phase 3 — Confort & monétisation (ouvert)
- [ ] Abonnement Premium (Stripe), quotas différenciés, file prioritaire.
- [ ] Application compagnon (Tauri, Win/macOS) : lit la demande dans les SavedVariables, appelle le site,
      écrit la réponse → le joueur fait `/reload` sans copier/coller.
- [ ] Nouvelles catégories : keybinds, WeakAuras simples, aide à la config d'addons.
- [ ] Multi-langue étendue (deDE, esES…), support des noms de sorts localisés via Game Data `locale=`.

---

## 5. Risques et décisions ouvertes

| Risque / question | Mitigation / à trancher |
|---|---|
| Pas de réseau en jeu → friction copier/coller | Assumer le modèle AskMrRobot ; compagnon en phase 3 |
| Le modèle invente des noms de sorts | Liste des sorts connus envoyée par l'addon + validation Game Data + linter bloquant |
| Coût IA vs gratuit | Quotas serrés, prompt caching, Sonnet 5 pour le tier gratuit si l'eval le permet, disjoncteur |
| Numéros d'Interface qui changent à chaque patch | CI automatique, TOC multi-versions |
| Politique Blizzard (addon payant interdit) | Addon 100 % gratuit ; premium uniquement côté web |
| Partage de compte / scripts | Liaison personnage ↔ compte Battle.net, rate-limit IP, limites hebdo |
| Confidentialité (BattleTag, personnages) | Données minimales, suppression de compte, hébergement UE (OVH) possible |
| « forcer » dans la demande initiale | Interprété comme *Classic* ; à confirmer (quelles versions Classic prioriser ?) |
| Nom du produit | À choisir avant la phase 1 (dépôts CurseForge, domaine) |

---

## 6. Sources consultées (21/09/2026)

- Battle.net OAuth : https://community.developer.battle.net/documentation/guides/using-oauth
- Portail développeur : https://develop.battle.net · exemples OAuth : https://github.com/Blizzard/oauth-client-sample
- Namespaces WoW : https://community.developer.battle.net/documentation/guides/game-data-apis-wow-namespaces
- Profile APIs : https://community.developer.battle.net/documentation/world-of-warcraft/profile-apis
- Game Data APIs : https://community.developer.battle.net/documentation/world-of-warcraft/game-data-apis
- Classic APIs : https://community.developer.battle.net/documentation/world-of-warcraft-classic/game-data-apis
- Rate limits : https://community.developer.battle.net/documentation/guides/game-data-apis · Terms of Use : https://www.blizzard.com/en-us/legal/a2989b50-5f16-43b1-abec-2ae17cc09dd6/blizzard-developer-api-terms-of-use
- Numéros d'Interface : https://warcraft.wiki.gg/wiki/Getting_the_current_interface_number · https://github.com/p3lim/toc-interface-updater
- Format TOC multi-versions : https://warcraft.wiki.gg/wiki/TOC_format · https://us.forums.blizzard.com/en/wow/t/the-client-now-supports-comma-delimited-interface-versions/1896097
- Macros : https://warcraft.wiki.gg/wiki/API_CreateMacro · https://wowpedia.fandom.com/wiki/API_EditMacro
- AskMrRobot (modèle export/import) : https://www.askmrrobot.com/guides/addon-documentation
- WeakAuras Companion (modèle compagnon) : https://github.com/WeakAuras/WeakAuras-Companion
- Patch actuel : https://warcraft.wiki.gg/wiki/Patch_12.2.5 · https://news.blizzard.com/en-us/article/24296142/hotfixes-september-17-2026
