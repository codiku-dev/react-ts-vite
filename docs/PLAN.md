# RAM Deal Hunter — Plan technique

Application Next.js qui crawle ~100 places de marché C2C (leboncoin, eBay, Kleinanzeigen, Vinted, forums…),
extrait les annonces de RAM (DDR5/DDR4), détecte les prix **anormalement bas**, notifie en temps réel
et affiche un dashboard du top des deals.

> **Contrainte structurante : coût récurrent = 0 €.**
> Aucune clé d'API payante, aucun proxy résidentiel, aucun SaaS facturé.
> Chaque brique est soit open source self-hosted, soit un free tier permanent.

---

## 1. Architecture générale — hybride VPS + PC maison

Deux machines déjà possédées, chacune sur ce qu'elle fait le mieux :

| Machine | Rôle | Pourquoi elle |
|---|---|---|
| **VPS** (24/7, IP datacenter) | Postgres, Redis, Next.js, scheduler, notifications, crawl des sources faciles (T0/T1) | toujours allumé → le dashboard et les alertes ne tombent jamais |
| **PC Windows** (RTX 4070, 32 Go) | Crawl furtif Camoufox, Firecrawl, Ollama sur GPU | **IP résidentielle FR** + 12 Go de VRAM + beaucoup de RAM |

L'IP résidentielle du PC est l'atout majeur : c'est exactement ce qui manquait pour leboncoin
et Facebook Marketplace. Une box Orange/Free vue par DataDome, c'est un vrai foyer français,
pas un datacenter Hetzner.

```
                    ┌──────────────────────────────────────┐
  navigateur ──────►│  VPS  (toujours allumé)              │
                    │  ┌────────────┐  ┌────────────────┐  │
                    │  │ Next.js 15 │  │ scheduler      │  │
                    │  │ dashboard  │  │ + worker T0/T1 │  │
                    │  └─────┬──────┘  └───┬────────────┘  │
                    │  ┌─────┴─────────────┴─────┐         │
                    │  │ Postgres 16 │ Redis 7   │         │
                    │  └───────────────────┬─────┘         │
                    └──────────────────────┼───────────────┘
                                           │ WireGuard / Tailscale
                                           │ (chiffré, gratuit)
                    ┌──────────────────────┼───────────────┐
                    │  PC Windows (WSL2)   ▼   IP RÉSIDENTIELLE
                    │  ┌─────────────────────────────────┐ │
                    │  │ worker T2/T3 (BullMQ consumer)  │ │
                    │  ├─────────────────────────────────┤ │
                    │  │ Firecrawl self-hosted           │ │
                    │  │   └ playwright-service →        │ │
                    │  │     Camoufox × 6-8              │ │
                    │  ├─────────────────────────────────┤ │
                    │  │ Ollama + Qwen2.5-14B  (CUDA)    │ │
                    │  └─────────────────────────────────┘ │
                    └──────────────────────────────────────┘
                                    │
                            Telegram Bot API
```

**Le PC est un consommateur de queue, pas un serveur.** Il se connecte au Redis du VPS via le tunnel
et pioche les jobs des files `crawl:t2` / `crawl:t3` / `parse:llm`. Conséquence directe :

> **Si le PC est éteint, rien ne casse.** Le dashboard, la base, les alertes et les ~60 sources
> T0/T1 continuent sur le VPS. Les jobs T2/T3 s'accumulent dans Redis et sont traités au prochain
> allumage. C'est le point de design le plus important de cette archi : **dégradation gracieuse**,
> jamais de panne.

**Décision inchangée : le crawling ne tourne pas en serverless.** Camoufox demande ~400 Mo de RAM
et 30-60 s par page. Le front ne déclenche jamais un crawl : il lit la base.

### Stack (tout gratuit)

| Couche | Choix | Coût | Pourquoi |
|---|---|---|---|
| Couche | Choix | Où | Pourquoi |
|---|---|---|---|
| Front | Next.js 15 App Router (`output: standalone`), TS, Tailwind, shadcn/ui | VPS | RSC, une seule codebase, derrière Caddy |
| DB | **Postgres 16 en conteneur** + Drizzle ORM | VPS | SQL relationnel + fenêtres glissantes pour les stats prix |
| Queue | **BullMQ + Redis 7** | VPS | files séparées par tier ; le PC est un consumer distant |
| Lien VPS↔PC | **Tailscale** (gratuit jusqu'à 100 machines) ou WireGuard | — | zéro port à ouvrir sur ta box, chiffré, reconnexion auto |
| Crawl T0/T1 | `undici` + `curl-impersonate` (ou `impit`) | VPS | TLS/JA3 d'un vrai navigateur, ~0 CPU |
| Crawl T2 | **camoufox-js + Playwright** | **PC** | open source (MPL), fingerprint patché en C++, IP résidentielle |
| Crawl T3 | **Firecrawl self-hosted** + `playwright-service` custom pilotant Camoufox | **PC** | orchestration sans clé API, depuis l'IP résidentielle |
| Parsing specs | regex d'abord, **Ollama + Qwen2.5-14B-Instruct Q4_K_M** en fallback | **PC (GPU)** | ~9 Go de VRAM sur les 12 du 4070, ~40 tok/s → qualité proche d'une API payante |
| TLS / reverse proxy | **Caddy** (HTTPS auto Let's Encrypt) | VPS | une ligne de config |
| Notif | **Telegram Bot API** | VPS | instantané, mobile, illimité en pratique |
| Taux de change | API **frankfurter.app** (BCE, sans clé) | VPS | cache 24 h |
| Monitoring | **Uptime Kuma** en conteneur (option) | VPS | détection des sources muettes |

### Le PC sous Windows, concrètement

- **WSL2 + Docker Desktop** avec l'intégration WSL activée. Camoufox tourne en Linux dans WSL2
  (bien plus stable que le portage Windows natif) ; le GPU est exposé à WSL2 via le driver NVIDIA,
  donc Ollama y accède en CUDA sans configuration particulière.
- **Empêcher la mise en veille** : `powercfg /change standby-timeout-ac 0`. L'écran peut s'éteindre,
  pas la machine — sinon le worker se déconnecte de Redis.
- **Démarrage auto** : Docker Desktop au login + `restart: unless-stopped` sur les conteneurs.
- **Le PC reste utilisable normalement** : on plafonne à 4 navigateurs quand une session interactive
  est détectée, 8 sinon. Le crawl est de l'attente réseau, pas du calcul — tu ne le sentiras pas
  en jouant, sauf si Ollama tourne au même moment (d'où la file `parse:llm` limitée à 1 job).

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

### Stratégie IP — l'IP résidentielle du PC change tout

Le problème n°1 du budget zéro était l'absence d'IP résidentielle. Ton PC la fournit gratuitement.
Un fingerprint Camoufox propre **+** une IP de box française **=** exactement le profil qu'un
antibot considère comme légitime. C'est ce que vendent les proxies résidentiels à 8 €/Go.

**Routage des sources par IP :**

| Sources | IP utilisée | Fréquence |
|---|---|---|
| ~60 sources T0/T1 (APIs, forums, OLX-likes) | VPS (datacenter) | 20-30 min |
| ~35 sources T2 (Vinted, Wallapop, Kleinanzeigen…) | **PC résidentielle** | 30-60 min |
| leboncoin, Facebook Marketplace | **PC résidentielle**, exclusivement | 45-60 min |

**Règles de préservation de l'IP résidentielle** — c'est ta seule, et un ban ferait mal
(il toucherait aussi ta navigation perso sur ces sites) :

- **Plafond strict** : 1 req / 20-30 s sur les sites sensibles, jamais plus de 2 pages par run.
- **Jamais de parallélisme** sur un même domaine sensible depuis l'IP maison.
- **Circuit breaker** : au 1er signal de blocage (403, captcha, DataDome), arrêt immédiat de la source
  pour 12 h. On ne s'acharne pas — c'est comme ça qu'on se fait blacklister durablement.
- **Heures creuses évitées** : crawler à 4 h du matin est plus suspect qu'à 20 h. On calque le
  scheduler sur des horaires plausibles (7 h-minuit) avec des trous aléatoires.
- **Cookies persistants par site** dans un volume Docker → tu ressembles à un habitué qui revient,
  pas à un visiteur neuf toutes les heures.

**Filet de sécurité si l'IP maison est bannie quand même :**
rotation IPv6 `/64` du VPS (18 milliards d'adresses gratuites, mais ~40 % des sites seulement sont
joignables en IPv6), Cloudflare WARP gratuit via `wgcf` pour un egress alternatif, et surtout
le **redémarrage de la box** : sur la plupart des FAI français en IP dynamique, ça suffit à changer
d'adresse. À garder comme dernier recours manuel, pas comme routine.

**Attente réaliste : les 100 sources passent, leboncoin et Facebook inclus.** C'était le seul point
d'interrogation du plan précédent ; il est levé.

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

**Étape 2 — LLM local sur le GPU** pour les 10 % non parsés (titres du genre
"vends barrettes gaming rgb pc neuf jamais servi") : **Ollama + Qwen2.5-14B-Instruct Q4_K_M**
sur la RTX 4070, sortie contrainte par JSON Schema, cache par hash de titre normalisé.

Le 4070 (12 Go de VRAM) fait tourner un 14B quantifié en Q4 (~9 Go) à ~40 tok/s. Comme la sortie
fait ~80 tokens, c'est ~2 s par annonce, et on peut batcher. À cette taille, la qualité d'extraction
est très proche d'une API payante — bien meilleure que le 3B CPU du plan précédent.

Cette étape peut aussi servir à **traduire/normaliser** les annonces étrangères (polonais, tchèque,
hongrois) que les regex ne couvrent pas, ce qui débloque une vingtaine de sources CEE.

Si le PC est éteint, les annonces non parsées restent en `needs_review` et sont traitées au prochain
allumage — le pipeline ne bloque jamais.

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

Files séparées par tier, consommées par des machines différentes :

```
crawl:t0, crawl:t1   → consumer VPS (24/7)
crawl:t2, crawl:t3   → consumer PC   (quand allumé)
parse:llm            → consumer PC   (concurrence 1, GPU)
detect, notify, ref  → consumer VPS
```

Avec 8 navigateurs sur le PC, les ~35 sources T2/T3 sont traitées en ~5 min. Le facteur limitant
n'est jamais le CPU mais le **rate limiting volontaire** : on veut rester lent. Si le PC a été
éteint 12 h, on ne rejoue pas les 12 runs manqués — le scheduler **coalesce** les jobs en retard
(un seul run par source au redémarrage), sinon on déclencherait une rafale qui ressemble
exactement à un bot.

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
| **P0** | Monorepo (`apps/web`, `apps/worker`, `services/camoufox`, `packages/db`, `packages/core`), **deux** compose (`compose.vps.yml`, `compose.home.yml`), Tailscale, Drizzle + migrations | `docker compose up` des deux côtés, PC qui consomme la queue du VPS |
| **P1** | Framework de crawl : ladder T0→T3, registre de sources, 3 sources pilotes (eBay API, Kleinanzeigen T2, un OLX T1) | annonces réelles en base |
| **P2** | Parsing specs (regex + LLM fallback) + garde-fous, backfill | `listing_specs` peuplée, précision mesurée sur 200 annonces annotées à la main |
| **P3** | Prix de référence + détection d'anomalie + `deals` | top deals interrogeable en SQL |
| **P4** | Front Next.js : dashboard, filtres, `/sources` | utilisable au quotidien |
| **P5** | Telegram + watches | notifié en < 5 min |
| **P6** | Montée à ~100 sources (adapter générique OLX-like en premier, ~30 d'un coup) | couverture large |
| **P7** | Durcissement : circuit breakers, coalescing, tests de non-régression des sélecteurs, alerte "source muette", Wake-on-LAN optionnel | ça tourne sans surveillance |

Chaque phase est mergeable et utile seule. P1→P3 est le cœur de valeur ; P6 est du volume, pas du risque.

---

## 11. Coûts — et budget RAM du VPS

| Poste | Coût récurrent |
|---|---|
| VPS | **déjà payé** |
| PC Windows | **déjà possédé** |
| Postgres, Redis, Caddy, Firecrawl, Camoufox, Ollama, Uptime Kuma | 0 € (open source) |
| Tailscale (≤ 100 machines) | 0 € |
| Telegram Bot API | 0 € |
| Taux de change (frankfurter.app) | 0 € |
| **Total logiciel + services** | **0 €/mois** |

**Le seul coût réel : l'électricité du PC.** À être transparent — ce n'est pas gratuit :
un PC de ce type consomme ~60-90 W au repos avec du crawl léger (le GPU ne travaille que par
à-coups pour Ollama). Soit ~50-65 kWh/mois, ≈ **10-16 €/mois** au tarif réglementé.

Deux façons de le ramener à ~0 :
- **Allumer le PC seulement quelques heures par jour** (le soir, quand tu l'utilises déjà). Grâce à
  la dégradation gracieuse, le VPS continue les 60 sources faciles 24/7 et le PC rattrape la file
  T2/T3 quand il démarre. Tu perds en réactivité sur leboncoin, pas en couverture.
- **Wake-on-LAN piloté par le VPS** : le VPS réveille le PC toutes les 3 h, le worker traite sa file,
  puis déclenche une mise en veille. ~30 min d'allumage par cycle → 2-3 €/mois. Plus élégant,
  un peu plus de plomberie (P7).

### Budget RAM

**PC (32 Go)** — très confortable : 8 Camoufox (3,2 Go) + Firecrawl (0,5 Go) + Ollama (2 Go RAM,
le modèle vit en VRAM) + WSL2 ≈ **7 Go**. Il te reste 25 Go pour ton usage normal.

**VPS** — allégé par rapport au plan précédent, puisqu'il ne fait plus de navigateur :
Postgres 300 Mo + Redis 100 Mo + Next.js 200 Mo + worker T0/T1 250 Mo ≈ **1 Go**.
→ **Un VPS 2 Go suffit largement.** Même le plus petit que tu aies fera l'affaire.

---

## 12. Risques

| Risque | Mitigation |
|---|---|
| Un site change son HTML → 0 annonce silencieusement | Canary : alerte si une source retourne 0 résultat 2 runs de suite alors qu'elle en retournait > 10 |
| Faux positifs (annonce de recherche, HS, arnaque) | Garde-fous §4 + seuil de confiance + badge scam |
| Référence prix instable au démarrage (peu de données) | Pas d'alerte tant que n < 30 par bucket ; bootstrap possible avec les prix neufs (Amazon/Newegg) comme borne haute |
| **Ban de l'IP résidentielle** (impacterait ta navigation perso) | Le risque le plus sérieux du plan. Circuit breaker au 1er signal, 1 req/20-30 s, jamais de parallélisme par domaine, horaires plausibles. Recours : WARP, IPv6 du VPS, redémarrage box |
| PC éteint / en veille | Dégradation gracieuse par design : le VPS assure 24/7, les jobs T2/T3 s'empilent et sont coalescés au réveil. Option Wake-on-LAN (P7) |
| IP dynamique de la box qui change | Tailscale gère la reconnexion tout seul (c'est le PC qui initie le tunnel, pas l'inverse) |
| Rafale de requêtes au redémarrage du PC | Coalescing des jobs en retard (§7) — un seul run par source, pas 12 |
| Le deal est parti avant que tu voies l'alerte | Fréquence ↑ sur les 10 sources les plus rentables, Telegram en priorité, `recheck:active` pour marquer les vendus |

---

## 13. Décisions à valider

1. **Le PC tourne-t-il 24/7, ou seulement quand tu l'utilises ?** Seul paramètre qui change la
   réactivité sur leboncoin (cf. §11 : ~13 €/mois d'électricité en 24/7, ~0 en usage ponctuel).
   Par défaut je pars sur **usage ponctuel + dégradation gracieuse**, c'est le meilleur ratio.
2. **Périmètre géographique** : FR seul, FR+DE+BENELUX, ou Europe entière ? (acheter en Pologne
   implique du port et du risque ; mais plus de sources = plus de deals)
3. **DDR5 seule ou DDR4 aussi ?** (DDR4 a aussi explosé, marché plus large)
4. **Mono-utilisateur ou multi-utilisateur** (auth) ? Mono simplifie beaucoup la P5.
5. **Dashboard exposé publiquement** (nom de domaine + Caddy) ou accessible **uniquement via
   Tailscale** ? La 2ᵉ option est plus simple et plus sûre : aucun port ouvert, pas d'auth à écrire.

---

## 14. Ce que le budget zéro coûte vraiment

Pour être clair sur les compromis, plutôt que de prétendre que c'est identique au setup payant :

| Aspect | Setup payant | Ce plan (0 €) | Impact réel |
|---|---|---|---|
| Furtivité | Camoufox | Camoufox | **aucun** — gratuit des deux côtés |
| Orchestration | Firecrawl cloud | Firecrawl self-hosted | **aucun** |
| IP résidentielle | proxies à ~8 €/Go | **ta box** | **aucun sur la qualité** ; en échange, volume à ménager (une seule IP à préserver) |
| Résolution DataDome | `proxy: stealth` inclus | Camoufox + IP résidentielle | **aucun** — c'est la même recette |
| Parsing LLM | Haiku 4.5 | Qwen2.5-14B sur RTX 4070 | quasi équivalent sur une tâche d'extraction aussi cadrée |
| Disponibilité | 100 % | VPS 100 %, PC intermittent | latence plus élevée sur ~35 sources quand le PC dort |
| Ops | zéro | 2 × `docker compose` | quelques heures de setup |

**Il ne reste aucun renoncement fonctionnel.** Le seul arbitrage est disponibilité vs électricité,
et il est entre tes mains (§13.1). Ajouter ton PC à l'équation a supprimé le dernier point faible
du plan gratuit.
