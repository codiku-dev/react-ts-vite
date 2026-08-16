# RAM Deal Hunter — Plan technique

Application Next.js qui crawle ~100 places de marché C2C (leboncoin, eBay, Kleinanzeigen, Vinted, forums…),
extrait les annonces de RAM (DDR5/DDR4), détecte les prix **anormalement bas**, notifie en temps réel
et affiche un dashboard du top des deals.

> **Contrainte structurante : coût récurrent = 0 €.** Tout tourne sur le VPS déjà payé.
> Aucune clé d'API payante, aucun proxy résidentiel, aucun SaaS facturé.
> Chaque brique ci-dessous est soit open source self-hosted, soit un free tier permanent.

---

## 1. Architecture générale

Tout est en Docker Compose sur **ton VPS**, une seule commande, aucun service externe facturé.

```
                         ┌─────────────────────────────────────────┐
   navigateur ──────────►│  VPS (docker compose)                   │
                         │                                         │
                         │  ┌───────────────┐   ┌───────────────┐  │
                         │  │ Next.js 15    │   │ worker        │  │
                         │  │ (standalone)  │◄─►│ BullMQ        │  │
                         │  │ dashboard+API │   │ scheduler     │  │
                         │  └───────┬───────┘   └───┬───────┬───┘  │
                         │          │               │       │      │
                         │  ┌───────┴───────────────┴──┐    │      │
                         │  │ Postgres 16   │ Redis 7  │    │      │
                         │  └──────────────────────────┘    │      │
                         │                                  ▼      │
                         │            ┌──────────────────────────┐ │
                         │            │ Firecrawl self-hosted    │ │
                         │            │  └ playwright-service →  │ │
                         │            │    Camoufox (Xvfb)       │ │
                         │            └──────────────────────────┘ │
                         │  ┌──────────────────────────┐           │
                         │  │ Ollama (Qwen2.5-3B) opt. │           │
                         │  └──────────────────────────┘           │
                         └────────────────┬────────────────────────┘
                                          ▼
                                  Telegram Bot API (gratuit)
```

**Décision clé : le crawling ne tourne PAS en serverless.** Un navigateur furtif (Camoufox) demande
~400 Mo de RAM et 30-60 s par page ; les fonctions serverless (10 s / 1 Go) sont inadaptées.
→ Le worker tourne sur le VPS. Le front ne déclenche jamais un crawl : il lit la base.
Le worker crawle en continu selon un scheduler, le front poll `/api/deals` toutes les 60 s.

### Stack (tout gratuit)

| Couche | Choix | Coût | Pourquoi |
|---|---|---|---|
| Front | Next.js 15 App Router (`output: standalone`), TS, Tailwind, shadcn/ui | 0 | RSC, une seule codebase ; déployé en Docker sur le VPS derrière Caddy |
| DB | **Postgres 16 en conteneur** + Drizzle ORM | 0 | SQL relationnel + fenêtres glissantes pour les stats prix |
| Queue | **BullMQ + Redis 7 en conteneur** | 0 | jobs répétables, retry/backoff, concurrence par domaine |
| Crawl T1 | `undici` + `curl-impersonate` (ou `impit`) | 0 | TLS/JA3 d'un vrai navigateur, ~0 CPU |
| Crawl T2 | **camoufox-js + Playwright** | 0 | open source (MPL), fingerprint patché en C++ |
| Crawl T3 | **Firecrawl self-hosted** (déjà sur ton VPS) branché sur un `playwright-service` custom qui pilote Camoufox | 0 | orchestration, retry, extraction markdown, sans clé API |
| Proxy/IP | rotation **IPv6 /64** du VPS + Cloudflare **WARP** gratuit + rate limiting poli | 0 | cf. §2 — remplace les proxies résidentiels payants |
| Parsing specs | regex d'abord, **Ollama + Qwen2.5-3B-Instruct** local en fallback | 0 | ~1,9 Go de RAM, tourne sur CPU, largement suffisant pour extraire 6 champs d'un titre |
| TLS / reverse proxy | **Caddy** (HTTPS auto Let's Encrypt) | 0 | une ligne de config |
| Notif | **Telegram Bot API** | 0 | instantané, mobile, illimité en pratique |
| Taux de change | API **frankfurter.app** (BCE, sans clé) | 0 | cache 24 h |
| Monitoring | **Uptime Kuma** en conteneur (option) | 0 | ping des sources muettes |

---

## 2. Stratégie anti-bot : le "scraping ladder"

Chaque source déclare le tier minimum requis. On monte de tier **uniquement** en cas d'échec,
et on mémorise le tier qui a marché (`sources.effective_tier`) pour ne pas repayer.

| Tier | Technique | Coût / page | Cible |
|---|---|---|---|
| **T0 — API officielle / flux** | eBay Browse API (free tier 5 000 appels/j), API Reddit, flux RSS, sitemaps XML, endpoints JSON internes de l'app mobile | 0 | eBay, Reddit, sites avec API ouverte |
| **T1 — HTTP impersonate** | `curl-impersonate` / `impit` : TLS + JA3/JA4 + ordre des frames HTTP/2 d'un vrai Chrome/Firefox, headers cohérents, cookies persistés | 0 | ~55 % des sites (annonces régionales, forums, OLX-likes) |
| **T2 — Camoufox** | Firefox patché en C++ : fingerprint (canvas, WebGL, fonts, screen, timezone, locale) injecté **sous** le runtime JS → indétectable par `navigator.webdriver`, `chrome.runtime`, fuites CDP. Piloté par Playwright via `camoufox-js`. Humanisation curseur/scroll, délais gaussiens | 0 (CPU du VPS) | ~35 % (Vinted, Wallapop, Subito, Kleinanzeigen, Marktplaats) |
| **T3 — Firecrawl self-hosted** | Ton instance Firecrawl, avec son `playwright-service` **remplacé par un microservice Camoufox** maison. On récupère l'orchestration (retry, cache, `/scrape`, `/crawl`, sortie markdown) sans clé ni facturation | 0 | Leboncoin, Facebook Marketplace, Amazon |

> ⚠️ Point d'attention honnête : le mode `proxy: "stealth"` de Firecrawl (celui qui casse DataDome
> tout seul) est une **fonctionnalité du cloud payant**, absente du self-hosted. Le self-hosted te donne
> l'orchestration, pas la furtivité. C'est donc **Camoufox qui fournit la furtivité**, et Firecrawl qui
> l'orchestre. D'où le branchement `firecrawl → playwright-service custom → Camoufox` ci-dessus :
> c'est le seul montage qui donne les deux gratuitement.

```yaml
# docker-compose.yml (extrait)
firecrawl-api:
  image: ghcr.io/firecrawl/firecrawl:latest
  environment:
    PLAYWRIGHT_MICROSERVICE_URL: http://camoufox-service:3003/scrape
    REDIS_URL: redis://redis:6379
    USE_DB_AUTHENTICATION: "false"     # pas de Supabase, pas de clé

camoufox-service:                       # microservice maison, ~150 lignes
  build: ./services/camoufox
  shm_size: 1gb                         # indispensable, sinon le navigateur crashe
```

### Détails d'implémentation T2 (Camoufox)

```ts
// worker/src/browser/camoufox.ts
import { launchServer } from 'camoufox-js'

const server = await launchServer({
  headless: 'virtual',          // Xvfb → pas de headless flag détectable
  humanize: true,               // mouvements souris bezier + délais
  geoip: true,                  // timezone/locale/lat-lon dérivés de l'IP du proxy
  locale: ['fr-FR'],
  os: ['windows', 'macos'],     // rotation de l'OS fingerprint
  block_images: true,           // -70 % bande passante (on garde l'URL de l'image)
  proxy: { server: proxyUrl, username, password },
})
```

Règles opérationnelles :
- **1 profil navigateur persistant par source** (cookies, localStorage) → on ressemble à un habitué,
  pas à un visiteur neuf à chaque requête.
- **Warm-up** : sur un site sensible, charger la home, attendre, puis naviguer vers la recherche —
  jamais d'entrée directe sur une URL de résultats profonde.
- **Rate limit par domaine** : token bucket Redis, 1 req / 8-20 s avec jitter, concurrence 1-2.
  Le but n'est pas la vitesse : 100 sites × 1 page de résultats / heure = trivial.
- **Budget d'échec** : 3 blocages consécutifs → source mise en `cooldown` 6 h + escalade de tier + alerte
  sur la page `/sources`.
- **Détection de blocage** : classifier la réponse (403/429, page < 2 Ko, présence de `captcha-delivery`,
  `cf-mitigated`, titre "Access Denied") → ne jamais parser silencieusement une page de blocage.

### Stratégie IP sans proxies payants

C'est le vrai renoncement du budget zéro : pas de proxies résidentiels. Quatre leviers gratuits,
par ordre d'efficacité :

1. **Le rate limiting est le meilleur proxy gratuit.** L'écrasante majorité des bans vient du volume,
   pas du fingerprint. À 1 requête / 15 s sur un domaine, avec un fingerprint Camoufox propre et des
   cookies persistants, une IP datacenter passe sur la plupart des sites. On vise 1 page de résultats
   par source et par heure — soit ~24 requêtes/jour/site. C'est moins qu'un utilisateur humain motivé.

2. **Rotation IPv6 `/64`.** Hetzner, OVH, Scaleway fournissent gratuitement un bloc `/64` par VPS,
   soit 18 milliards d'adresses. On bind une IP source différente par session :
   ```ts
   // rotation gratuite, une IPv6 fraîche par requête
   const agent = new Agent({ localAddress: randomIpv6FromPrefix(process.env.IPV6_PREFIX) })
   ```
   Limite honnête : ne marche que sur les sites joignables en IPv6 (~40 %), et certains bannissent
   le `/64` entier plutôt que l'adresse. Utile, pas magique.

3. **Cloudflare WARP** (gratuit, illimité) via `wgcf` → un egress IP différent de celui du VPS,
   dans un pool consommateur. On peut lancer 2-3 conteneurs WARP avec des identités différentes
   et router les sources sensibles à travers. Reconnexion = nouvelle IP.

4. **Fallback résidentiel gratuit** : un vieux Raspberry Pi (ou ton PC) chez toi en tunnel WireGuard
   vers le VPS, utilisé **uniquement** pour les 2-3 sources les plus dures (leboncoin, FB).
   C'est une vraie IP résidentielle FR, gratuite, et le volume y sera très faible.
   → Si tu n'as pas de machine dispo à la maison, dis-le : ça change les attentes sur ces 2 sites.

Attente réaliste : avec ça, **~90 des 100 sources passent sans problème**. Leboncoin et Facebook
Marketplace resteront capricieux sans levier 4 — on les crawlera en best-effort, avec cooldown long,
et la page `/sources` dira honnêtement quand elles sont bloquées plutôt que de faire semblant.

### Ce qui reste réellement dur

Facebook Marketplace (login obligatoire + détection comportementale) et Leboncoin (DataDome agressif)
sont les deux seuls qui coûteront cher. Pour eux : T3 systématique, fréquence réduite (1×/h max),
et pour Leboncoin l'app mobile expose une API JSON plus permissive que le web — à explorer en priorité
avant de payer du Firecrawl.

### Cadre

Robots.txt et CGU varient selon les sites ; certains interdisent le scraping. Les annonces contiennent
des données personnelles (pseudo, ville, parfois téléphone) → **on ne stocke que ce qui sert au deal**
(titre, prix, specs, URL, image, ville) et jamais les coordonnées du vendeur. Rétention 90 jours.
Rate limiting poli. C'est ton appel sur les sites que tu inclus ; le registre de sources rend
l'ajout/retrait d'une source trivial.

---

## 3. Registre des sources (~100)

Une source = un fichier de config déclaratif + optionnellement un adapter TS custom.

```ts
// worker/src/sources/kleinanzeigen.ts
export default defineSource({
  id: 'kleinanzeigen-de',
  country: 'DE', currency: 'EUR', tier: 'T2',
  searchUrls: (q) => [`https://www.kleinanzeigen.de/s-${q}/k0c225`],
  queries: ['ddr5', 'ddr5-32gb', 'corsair-vengeance-ddr5', 'g-skill-trident-z5'],
  pagination: { type: 'path', pattern: '/seite:{n}', maxPages: 3 },
  list: {
    item: 'article.aditem',
    url: { sel: 'a.ellipsis', attr: 'href', absolute: true },
    title: 'a.ellipsis',
    price: '.aditem-main--middle--price-shipping--price',
    image: { sel: 'img', attr: 'src' },
    location: '.aditem-main--top--left',
    postedAt: '.aditem-main--top--right',
  },
  detail: { optional: true, description: '#viewad-description-text' },
})
```

**Cible ~100 sources, réparties :**

- **FR** : leboncoin, eBay.fr, Rakuten, Vinted, Facebook Marketplace, Geektroc/HardwareFr,
  forums Cowcotland / CanardPC / JeuxVideo troc, Rueducommerce marketplace, Cdiscount marketplace, Anibis
- **DE/AT/CH** : Kleinanzeigen, eBay.de, Willhaben, Ricardo, Tutti, Anibis
- **UK/IE** : eBay.co.uk, Gumtree, Facebook, Adverts.ie
- **BENELUX** : Marktplaats, 2dehands, 2ememain, Tweakers V&A (excellent pour le hardware)
- **ES/PT/IT** : Wallapop, Milanuncios, Subito, OLX.pt, CustoJusto
- **Nordics** : Blocket, Tori, DBA, Finn.no
- **CEE** : OLX.pl, Allegro Lokalnie, Bazos.cz/.sk, Jofogas, OLX.ro, Bolha, Njuskalo
- **US/CA** : eBay.com, Craigslist (top 20 villes), OfferUp, Mercari, Swappa, r/hardwareswap
- **Communautés** : Reddit (r/hardwareswap, r/buildapcsales, r/hardwareswapfr) via l'API Reddit officielle,
  Discord webhooks de deals, forums spécialisés (Tweakers, Hardwareluxx, Overclockers UK MM)

Les OLX-likes (OLX.*, Bazos.*, Subito, Blocket…) partagent des structures très proches → un **adapter
générique paramétré** couvre une trentaine de sources d'un coup. C'est ce qui rend les 100 sites tenables.

---

## 4. Extraction des specs RAM

Pipeline : `raw_listing` → normalisation → `listing_spec`.

**Étape 1 — regex (gratuit, ~90 % de couverture)** sur titre + description :

```
génération   /\bDDR(3|4|5)\b/i, /\bPC5-\d{4,5}\b/  → DDR5
kit          /(\d)\s*[xX*]\s*(\d{1,3})\s*(?:GB|GO|GIG)/  → 2x16
total        /(\d{1,3})\s*(?:GB|GO)\b/ → 32
fréquence    /(\d{4,5})\s*(?:MHZ|MT\/S)/ → 6000
latence      /\bCL\s?(\d{2})\b/ → 30
form factor  /\bSO-?DIMM\b/ → laptop (à exclure ou catégorie à part)
ECC/RDIMM    /\b(ECC|RDIMM|LRDIMM)\b/ → serveur, marché différent
marque       Corsair|G.Skill|Kingston|Crucial|TeamGroup|Patriot|ADATA|Kingston Fury...
```

**Étape 2 — LLM local en fallback** pour les 10 % non parsés (titres du genre
"vends barrettes gaming rgb pc neuf jamais servi") : **Ollama + Qwen2.5-3B-Instruct** sur le VPS,
sortie contrainte en JSON (`format: json_schema`), cache par hash de titre normalisé.
~1,9 Go de RAM, tourne en CPU, ~2 s par annonce — largement assez vu qu'on ne l'appelle que sur
quelques dizaines d'annonces par heure. Coût : 0 €.
Si la RAM du VPS est trop juste, on dégrade proprement : les annonces non parsées vont dans une file
`needs_review` visible dans l'UI, sans bloquer le reste du pipeline.

**Étape 3 — garde-fous** (indispensables, c'est ce qui tue les faux positifs) :
- exclure les **annonces de recherche** : `/\b(cherche|recherche|achète|wanted|suche|busco|WTB)\b/`
- exclure le **HS / pièces** : `/\b(HS|ne fonctionne pas|pour pièces|défectueu|faulty|broken|as-is)\b/`
- exclure la **SO-DIMM** et l'**ECC/RDIMM** du calcul de référence (marchés distincts)
- détecter le **prix "à partir de" / lot / à débattre** et les annonces sans prix
- détecter la **confusion kit/barrette** : "32GB (2x16)" vs "32GB" seule barrette → capacité totale identique,
  mais si le titre dit "2x16" et la description "une seule barrette", flag `spec_confidence: low`

---

## 5. Détection d'anomalie de prix

Le prix de référence est **calculé depuis le corpus lui-même**, pas codé en dur — la RAM DDR5 a vu son
prix multiplié en quelques mois, tout seuil statique serait faux en deux semaines.

**Bucket** = `(génération, capacité totale, tranche de fréquence, ECC?)`
ex. `DDR5 / 32GB / 5600-6400 / non-ECC`.

Pour chaque bucket, sur une fenêtre glissante de 30 jours (recalcul horaire, matérialisé en table) :

```
median_ppg  = médiane du prix/GB           (robuste, pas la moyenne)
mad         = median(|x - median|) × 1.4826  (écart-type robuste)
p10, p25    = quantiles bas
```

**Score du deal :**

```
discount = (median_ppg - listing_ppg) / median_ppg
z        = (listing_ppg - median_ppg) / mad

deal_score = 100 × clamp(discount, 0, 1) × confidence_factor
```

Un listing est **alerté** si :
- `z < -2.5` **ET** `discount > 30 %`
- **ET** le bucket a ≥ 30 observations sur 30 j (sinon référence non fiable → pas d'alerte)
- **ET** `spec_confidence >= medium`
- **ET** pas déjà alerté (dédup par `fingerprint = hash(source, external_id)`)

**Filtre anti-arnaque** : un prix < p1 du bucket (ex. 64 Go DDR5 à 25 €) est presque toujours une arnaque
ou une erreur d'annonce. On ne le cache pas — on l'affiche avec un badge `⚠️ Trop beau, méfiance`
plutôt que de le mettre en tête de liste.

**Frais de port** : normaliser en `price_total = price + shipping_estimate` quand disponible ;
sinon marquer `shipping_unknown` et retirer 0 (mais afficher le flag dans l'UI).

**Devises** : conversion vers EUR au taux du jour (API ECB, cachée 24 h), stockage du montant original.

---

## 6. Schéma de base

```sql
sources(id, slug, name, country, currency, tier, effective_tier, enabled,
        rate_limit_ms, last_run_at, consecutive_failures, cooldown_until)

crawl_runs(id, source_id, started_at, finished_at, status, pages_fetched,
           listings_found, listings_new, error, blocked_reason, tier_used, cost_cents)

listings(id, source_id, external_id, url, title, description, price_cents, currency,
         price_eur_cents, shipping_cents, image_url, location, country,
         posted_at, first_seen_at, last_seen_at, is_active, raw jsonb,
         UNIQUE(source_id, external_id))

listing_specs(listing_id PK, ddr_gen, total_gb, modules, gb_per_module, speed_mhz, cl,
              form_factor, is_ecc, brand, confidence, parsed_by /* regex|llm */)

price_reference(bucket_key PK, window_start, window_end, n_obs,
                median_ppg_cents, mad_ppg_cents, p10, p25, updated_at)

deals(listing_id PK, bucket_key, ppg_cents, discount_pct, z_score, deal_score,
      flags text[] /* scam_suspect, shipping_unknown, low_confidence */, detected_at)

alerts(id, listing_id, channel, sent_at, status)
watches(id, user_id, name, filters jsonb, min_discount, channels text[], enabled)
```

Index critiques : `deals(deal_score DESC, detected_at DESC)`, `listings(last_seen_at)`,
`listing_specs(ddr_gen, total_gb)`.

---

## 7. Ordonnancement

BullMQ, une file par tier (concurrence différente), jobs répétables :

```
crawl:source:{id}   toutes les 60 min (sites sensibles) / 20 min (sites faciles)
                    + jitter ±25 % pour ne pas taper à la même minute ronde
parse:listing       déclenché à l'insertion
refresh:reference   toutes les heures
detect:deals        après refresh:reference
notify:dispatch     toutes les 2 min (batch les alertes, évite le spam)
recheck:active      6× / jour sur les deals du top → marquer `sold`/`removed`
```

Concurrence globale : bornée par la RAM du VPS (cf. §11), pilotée par une variable d'env
`CRAWL_CONCURRENCY`. Même à 2 navigateurs en parallèle, 100 sources/heure passent : une source =
1 à 3 pages, soit ~30 s de navigateur. 100 × 30 s = 50 min de temps navigateur cumulé, réparti sur
2 workers = 25 min par heure. On a de la marge, et c'est ce qui rend le budget zéro viable.

---

## 8. Frontend

- **`/` Dashboard** — grille de cartes triées par `deal_score` : image, titre, prix, **€/GB**,
  badge `-45 % vs marché`, logo du site, ville, âge de l'annonce, bouton "Voir l'annonce" (target blank).
  Auto-refresh 60 s, toast quand un nouveau deal entre dans le top.
- **Filtres** (URL state, donc partageable) : génération, capacité (16/32/48/64/96/128),
  config kit (2×16, 1×32, 2×32…), fréquence min, prix max, remise min, pays, source, exclure SO-DIMM.
- **`/deal/[id]`** — détail, historique du prix de référence du bucket (sparkline), annonces comparables.
- **`/watches`** — créer une alerte : "DDR5 2×16 ≥ 6000 MHz, remise > 35 %, FR+DE" → Telegram.
- **`/sources`** — santé du crawl : dernier run, taux de succès 24 h, tier effectif, coût, statut bloqué.
  Avec 100 sources, cette page est **le** outil de maintenance : sans elle on ne voit pas qu'un site
  a changé son HTML et retourne 0 annonce depuis 3 jours.
- Images : proxy via `next/image` avec `remotePatterns` (les CDN des sites bloquent souvent le hotlink)
  → route `/api/img?u=...` qui refetch et cache 24 h.

---

## 9. Notifications

1. **Telegram** (principal) — bot + `chat_id`, message avec photo, prix, €/GB, remise, lien direct.
   Latence < 5 s, **gratuit et illimité**, natif mobile. C'est le canal qui compte : sur un vrai bon
   deal, tu as quelques minutes.
2. **Web Push** (VAPID) depuis le dashboard — gratuit, aucune dépendance externe (clés générées en local).
3. **Digest quotidien** du top 10 : envoyé sur Telegram lui aussi (pas d'email → pas de service SMTP
   à payer ni de problèmes de délivrabilité).

Anti-spam : dédup par listing, max 20 alertes/h, regroupement si > 3 deals en 2 min.

---

## 10. Phasage

| Phase | Contenu | Sortie |
|---|---|---|
| **P0** | Monorepo (`apps/web`, `apps/worker`, `services/camoufox`, `packages/db`, `packages/core`), `docker-compose.yml` complet (Postgres, Redis, Firecrawl, Camoufox, Caddy), Drizzle + migrations | `docker compose up` fonctionne sur le VPS |
| **P1** | Framework de crawl : ladder T0→T3, registre de sources, 3 sources pilotes (eBay API, Kleinanzeigen T2, un OLX T1) | annonces réelles en base |
| **P2** | Parsing specs (regex + LLM fallback) + garde-fous, backfill | `listing_specs` peuplée, précision mesurée sur 200 annonces annotées à la main |
| **P3** | Prix de référence + détection d'anomalie + `deals` | top deals interrogeable en SQL |
| **P4** | Front Next.js : dashboard, filtres, `/sources` | utilisable au quotidien |
| **P5** | Telegram + watches | notifié en < 5 min |
| **P6** | Montée à ~100 sources (adapter générique OLX-like en premier, ~30 d'un coup) | couverture large |
| **P7** | Durcissement : rotation proxy, cooldowns, tests de non-régression des sélecteurs, alerte "source muette" | ça tourne sans surveillance |

Chaque phase est mergeable et utile seule. P1→P3 est le cœur de valeur ; P6 est du volume, pas du risque.

---

## 11. Coûts — et budget RAM du VPS

| Poste | Coût récurrent |
|---|---|
| VPS | **déjà payé** (aucun surcoût) |
| Postgres 16 (conteneur) | 0 € |
| Redis 7 (conteneur) | 0 € |
| Firecrawl self-hosted | 0 € |
| Camoufox | 0 € (open source) |
| Ollama + Qwen2.5-3B | 0 € |
| IPv6 /64 + Cloudflare WARP | 0 € |
| Telegram Bot API | 0 € |
| Caddy / Let's Encrypt | 0 € |
| Taux de change (frankfurter.app) | 0 € |
| **Total** | **0 €/mois** |

La vraie ressource contrainte n'est pas l'argent mais la **RAM du VPS** :

| Service | RAM |
|---|---|
| Postgres | ~300 Mo |
| Redis | ~100 Mo |
| Next.js standalone | ~200 Mo |
| Worker Node | ~250 Mo |
| Firecrawl (api + worker) | ~500 Mo |
| Camoufox × N instances | **~400 Mo chacune** |
| Ollama (Qwen2.5-3B, chargé à la demande) | ~2 Go |

- **VPS 4 Go** → 2 instances Camoufox, Ollama désactivé (regex seule + file `needs_review`). Ça tourne.
- **VPS 8 Go** → 4-6 instances Camoufox + Ollama. Confortable pour 100 sources/heure.

→ **Dis-moi la taille de ton VPS**, c'est le paramètre qui fixe la concurrence de crawl et le sort
du parsing LLM. Le reste du plan ne bouge pas.

---

## 12. Risques

| Risque | Mitigation |
|---|---|
| Un site change son HTML → 0 annonce silencieusement | Canary : alerte si une source retourne 0 résultat 2 runs de suite alors qu'elle en retournait > 10 |
| Faux positifs (annonce de recherche, HS, arnaque) | Garde-fous §4 + seuil de confiance + badge scam |
| Référence prix instable au démarrage (peu de données) | Pas d'alerte tant que n < 30 par bucket ; bootstrap possible avec les prix neufs (Amazon/Newegg) comme borne haute |
| Blocage de l'IP du VPS (pas de proxy payant) | Rate limiting agressif (§2), rotation IPv6 /64, WARP, cooldown par source. Sur les 2-3 sites les plus durs : tunnel WireGuard vers une IP résidentielle perso |
| RAM du VPS insuffisante | `CRAWL_CONCURRENCY` configurable, Ollama optionnel, `block_images` activé, profils navigateur recyclés toutes les N pages |
| Le deal est parti avant que tu voies l'alerte | Fréquence ↑ sur les 10 sources les plus rentables, Telegram en priorité, `recheck:active` pour marquer les vendus |

---

## 13. Décisions à valider

1. **Taille du VPS** (RAM / vCPU) — fixe `CRAWL_CONCURRENCY` et si Ollama est activé (cf. §11).
2. **Une IP résidentielle dispo chez toi ?** (Raspberry Pi / PC allumé en WireGuard) — conditionne
   la faisabilité réelle de leboncoin et Facebook Marketplace.
3. **Périmètre géographique** : FR seul, FR+DE+BENELUX, ou Europe entière ? (acheter en Pologne
   implique du port et du risque ; mais plus de sources = plus de deals)
4. **DDR5 seule ou DDR4 aussi ?** (DDR4 a aussi explosé, marché plus large)
5. **Mono-utilisateur ou multi-utilisateur** (auth) ? Mono simplifie beaucoup la P5.

---

## 14. Ce que le budget zéro coûte vraiment

Pour être clair sur les compromis, plutôt que de prétendre que c'est identique au setup payant :

| Aspect | Payant | Gratuit (ce plan) | Impact réel |
|---|---|---|---|
| Furtivité de base | Camoufox | Camoufox | **aucun** — c'est gratuit des deux côtés |
| Orchestration | Firecrawl cloud | Firecrawl self-hosted | **aucun** |
| Résolution auto DataDome | incluse (`proxy: stealth`) | à faire soi-même via Camoufox + IP propre | leboncoin/FB en best-effort |
| Diversité d'IP | résidentiels illimités | IPv6 + WARP + 1 IP perso | fréquence de crawl réduite sur les sites durs |
| Parsing LLM | Haiku (rapide, très fiable) | Qwen2.5-3B local (plus lent, un peu moins fiable) | ~2-3 % de specs mal parsées en plus → mitigé par la file `needs_review` |
| Ops | zéro | `docker compose` à maintenir | quelques heures de setup |

Rien de bloquant. Le seul vrai renoncement est la fréquence de crawl sur leboncoin et Facebook —
et il disparaît si tu as une machine à la maison pour le tunnel WireGuard.
