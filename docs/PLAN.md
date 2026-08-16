# RAM Deal Hunter — Plan technique

Application Next.js qui crawle ~100 places de marché C2C (leboncoin, eBay, Kleinanzeigen, Vinted, forums…),
extrait les annonces de RAM (DDR5/DDR4), détecte les prix **anormalement bas**, notifie en temps réel
et affiche un dashboard du top des deals.

> **Contraintes structurantes :**
> **(1)** coût récurrent = 0 € — aucune clé d'API payante, aucun proxy, aucun SaaS facturé ;
> **(2)** **tout tourne en local sur le PC Windows** (RTX 4070, 32 Go) — pas de VPS, pas de cloud.

---

## 1. Architecture générale — 100 % local

Une seule machine, un seul `docker compose up`. Le PC a tout ce qu'il faut :
**IP résidentielle française** (ce que les proxies vendent 8 €/Go), **12 Go de VRAM** pour le
parsing LLM, **32 Go de RAM** pour faire tourner 8 navigateurs sans le sentir.

```
   ┌────────────────────────────────────────────────────────────┐
   │  PC Windows — WSL2 + Docker Desktop                        │
   │                                                            │
   │   ┌──────────────┐         ┌──────────────────────────┐    │
   │   │ Next.js 15   │◄───────►│ worker (BullMQ)          │    │
   │   │ dashboard    │         │ scheduler + crawl + deals│    │
   │   │ :3000        │         └────┬──────────────┬──────┘    │
   │   └──────┬───────┘              │              │           │
   │          │      ┌───────────────┴──┐           │           │
   │          └──────┤ Postgres │ Redis │           │           │
   │                 └──────────────────┘           ▼           │
   │                              ┌─────────────────────────┐   │
   │                              │ Firecrawl self-hosted   │   │
   │                              │  └ playwright-service → │   │
   │                              │    Camoufox × 8 (Xvfb)  │   │
   │                              └─────────────────────────┘   │
   │                 ┌──────────────────────────────┐           │
   │                 │ Ollama + Qwen2.5-14B (CUDA)  │           │
   │                 └──────────────────────────────┘           │
   └────────────────────────┬─────────────────────┬─────────────┘
                            │                     │
                    IP RÉSIDENTIELLE FR    Telegram Bot API
                     (vers les 100 sites)   (alertes sur ton tel)
```

**Ce que le tout-local simplifie :** plus de tunnel, plus de deuxième compose, plus de reverse proxy,
plus de HTTPS, plus d'authentification (le dashboard écoute sur `localhost:3000`, personne d'autre
n'y accède), plus de secrets à synchroniser. Le projet perd un tiers de sa plomberie.

**Ce que ça coûte, et qu'il faut assumer :** quand le PC est éteint, **tout est arrêté** — pas de
crawl, pas de dashboard, pas d'alertes. C'est le seul vrai compromis, et il se gère (cf. §7).

> **Accès depuis ton téléphone (optionnel, gratuit)** : installe **Tailscale** sur le PC et sur ton
> mobile → tu ouvres le dashboard depuis n'importe où quand le PC est allumé, sans ouvrir un seul
> port sur ta box. Les alertes Telegram, elles, arrivent de toute façon sur ton téléphone : elles
> ne sortent que du PC vers Telegram, aucune connexion entrante n'est nécessaire.

**Décision inchangée : le front ne déclenche jamais un crawl.** Il lit la base. Le worker crawle
selon son scheduler, le dashboard poll `/api/deals` toutes les 60 s.

### Stack (tout gratuit)

| Couche | Choix | Coût | Pourquoi |
|---|---|---|---|
| Couche | Choix | Pourquoi |
|---|---|---|
| Front | Next.js 15 App Router, TS, Tailwind, shadcn/ui | RSC, une seule codebase, sert sur `localhost:3000` |
| DB | **Postgres 16 en conteneur** + Drizzle ORM | SQL relationnel + fenêtres glissantes pour les stats prix |
| Queue | **BullMQ + Redis 7 en conteneur** | jobs répétables, retry/backoff, rate limit par domaine |
| Crawl T0/T1 | `undici` + `curl-impersonate` (ou `impit`) | TLS/JA3 d'un vrai navigateur, ~0 CPU |
| Crawl T2 | **camoufox-js + Playwright** | open source (MPL), fingerprint patché en C++ |
| Crawl T3 | **Firecrawl self-hosted** + `playwright-service` custom pilotant Camoufox | orchestration sans clé API |
| Parsing specs | regex d'abord, **Ollama + Qwen2.5-14B-Instruct Q4_K_M** en fallback | ~9 Go des 12 Go de VRAM du 4070, ~40 tok/s → qualité proche d'une API payante |
| Notif | **Telegram Bot API** | instantané, sur ton téléphone même quand tu n'es pas devant le PC |
| Accès mobile | **Tailscale** (optionnel) | dashboard depuis le tel, zéro port ouvert sur la box |
| Taux de change | API **frankfurter.app** (BCE, sans clé) | cache 24 h |

### Le PC sous Windows, concrètement

- **WSL2 + Docker Desktop** avec l'intégration WSL activée. Camoufox tourne en Linux dans WSL2
  (bien plus stable que le portage Windows natif) ; le GPU est exposé à WSL2 par le driver NVIDIA,
  donc Ollama y accède en CUDA sans configuration particulière.
- **Empêcher la mise en veille** pendant les sessions de crawl :
  `powercfg /change standby-timeout-ac 0`. L'écran peut s'éteindre, pas la machine.
- **Démarrage auto** : Docker Desktop au login + `restart: unless-stopped` sur tous les conteneurs.
  Tu allumes le PC, le crawl reprend tout seul, tu n'as rien à lancer.
- **Limiter la RAM de WSL2** dans `.wslconfig` (ex. `memory=12GB`) pour qu'il ne grignote pas
  tes 32 Go quand tu joues.
- **Le PC reste utilisable normalement** : le worker détecte une session interactive et plafonne
  à 4 navigateurs (8 sinon), et met la file `parse:llm` en pause si le GPU est déjà chargé —
  sinon Ollama et un jeu se disputeraient la VRAM. Le crawl lui-même est de l'attente réseau,
  pas du calcul : tu ne le sentiras pas.

---

## 2. Stratégie anti-bot : le "scraping ladder"

Chaque source déclare le tier minimum requis. On monte de tier **uniquement** en cas d'échec,
et on mémorise le tier qui a marché (`sources.effective_tier`) pour ne pas repayer.

| Tier | Technique | Coût / page | Cible |
|---|---|---|---|
| **T0 — API officielle / flux** | eBay Browse API (free tier 5 000 appels/j), API Reddit, flux RSS, sitemaps XML, endpoints JSON internes de l'app mobile | 0 | eBay, Reddit, sites avec API ouverte |
| **T1 — HTTP impersonate** | `curl-impersonate` / `impit` : TLS + JA3/JA4 + ordre des frames HTTP/2 d'un vrai Chrome/Firefox, headers cohérents, cookies persistés | 0 | ~55 % des sites (annonces régionales, forums, OLX-likes) |
| **T2 — Camoufox** | Firefox patché en C++ : fingerprint (canvas, WebGL, fonts, screen, timezone, locale) injecté **sous** le runtime JS → indétectable par `navigator.webdriver`, `chrome.runtime`, fuites CDP. Piloté par Playwright via `camoufox-js`. Humanisation curseur/scroll, délais gaussiens | 0 (CPU local) | ~35 % (Vinted, Wallapop, Subito, Kleinanzeigen, Marktplaats) |
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
  geoip: true,                  // timezone/locale/lat-lon cohérents avec ton IP réelle
  locale: ['fr-FR'],            // cohérent avec une IP française — surtout ne pas mentir ici
  os: ['windows'],              // ton IP est résidentielle FR : un fingerprint Windows est le
                                // plus plausible. La cohérence prime sur la rotation
  block_images: true,           // -70 % bande passante (on garde l'URL de l'image)
  // pas de proxy : on sort directement par la box, c'est tout l'intérêt
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

### Stratégie IP — une seule IP, résidentielle, à préserver

Ton IP de box est **exactement** ce que les antibots considèrent comme légitime : un vrai foyer
français, pas un datacenter. C'est ce que vendent les proxies résidentiels à 8 €/Go, et tu l'as
gratuitement. Fingerprint Camoufox propre + IP résidentielle = le duo qui fait passer leboncoin
et Facebook Marketplace.

**Mais c'est ta seule IP, et elle est aussi celle de ta navigation perso.** Un ban sur leboncoin
te gênerait toi, pas juste le bot. Toute la discipline du crawl découle de là :

- **Plafond strict** : 1 req / 15 s sur les sites normaux, 1 req / 25-30 s sur les sensibles
  (leboncoin, Vinted, FB), jamais plus de 2-3 pages de résultats par run.
- **Jamais de parallélisme sur un même domaine.** Les 8 navigateurs travaillent sur 8 **sites
  différents**, jamais sur le même. Le rate limit est un token bucket Redis par domaine.
- **Circuit breaker** : au 1er signal de blocage (403, 429, captcha, DataDome), arrêt immédiat de
  la source pour 12 h. On ne réessaie pas, on ne s'acharne pas — c'est l'acharnement qui transforme
  un blocage temporaire en blacklist durable.
- **Horaires plausibles** : le scheduler ne tourne qu'entre 7 h et minuit, avec des trous aléatoires.
  Un foyer qui consulte leboncoin à 4 h du matin toutes les nuits, c'est un signal.
- **Cookies persistants par site** dans un volume Docker → tu es un habitué qui revient, pas un
  visiteur neuf toutes les heures.
- **Budget global** : ~100 sources × 2 pages / heure ≈ 200 requêtes/h réparties sur 100 domaines
  différents. Par domaine, c'est 2 pages/heure — moins qu'un humain qui chine sérieusement.

**Si l'IP est bannie quand même** (recours manuels, pas de routine) :
- **Redémarrage de la box** — chez la plupart des FAI français en IP dynamique, ça suffit à changer
  d'adresse. Le plus simple et le plus efficace.
- **Cloudflare WARP** gratuit via `wgcf` → egress alternatif, à réserver aux sources non sensibles.
- La page `/sources` affiche l'état de blocage de chaque site : tu vois immédiatement ce qui coince,
  plutôt que de croire que le crawl marche alors qu'il ramène des pages d'erreur.

**Attente réaliste : les 100 sources passent, leboncoin et Facebook inclus.**

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

### Liste des sources retenues

Colonne **P** = priorité d'implémentation (1 = P1, à faire en premier ; 3 = volume, phase P6).
Colonne **T** = tier de crawl (cf. §2). ⭐ = source à fort rendement attendu.

#### 🇫🇷 France — priorité absolue (retrait en main propre possible)

| # | Site | T | P | Note |
|---|---|---|---|---|
| 1 | **leboncoin.fr** ⭐ | T3 | 1 | Le gisement n°1. DataDome agressif → IP résidentielle obligatoire. Explorer d'abord l'API de l'app mobile, plus permissive que le web |
| 2 | **eBay.fr** ⭐ | T0 | 1 | API Browse officielle, 5 000 appels/j gratuits. Zéro risque, à faire en premier |
| 3 | **Vinted.fr** | T2 | 2 | Oui, il y a de l'informatique dessus, et les vendeurs y connaissent mal les prix — terrain à erreurs |
| 4 | **Facebook Marketplace** ⭐ | T3 | 3 | Gros volume, mais login requis + détection comportementale. Le plus dur du lot, à garder pour la fin |
| 5 | **Rakuten.fr** | T1 | 2 | Ex-PriceMinister, marketplace C2C |
| 6 | **Geektroc** (`geektroc.fr`) ⭐ | T1 | 1 | Troc hardware entre passionnés, petit mais très dense en RAM |
| 7 | **HardwareFr / forum troc** | T1 | 2 | Historique, encore actif |
| 8 | **Cowcotland — forum petites annonces** | T1 | 2 | Communauté hardware FR |
| 9 | **CanardPC — forum troc** | T1 | 2 | Idem, bonne qualité d'annonces |
| 10 | **Overclocking.com — forum PA** | T1 | 3 | Niche |
| 11 | **JeuxVideo.com — forum troc** | T1 | 3 | Bruyant, mais gratuit à crawler |
| 12 | **Cdiscount Marketplace** (occasion) | T1 | 3 | Vendeurs tiers |
| 13 | **Rueducommerce Marketplace** | T1 | 3 | Idem |
| 14 | **Label Emmaüs / Emmaüs Connect** | T1 | 3 | Rare mais prix cassés par méconnaissance |
| 15 | **Troc.com** | T1 | 3 | Dépôt-vente |

#### 🇩🇪🇦🇹🇨🇭 Germanophone — le meilleur rapport volume/facilité d'Europe

| # | Site | T | P | Note |
|---|---|---|---|---|
| 16 | **Kleinanzeigen.de** ⭐ | T2 | 1 | Ex-eBay Kleinanzeigen. **Le plus gros gisement européen**, structure HTML stable, T2 suffit |
| 17 | **eBay.de** ⭐ | T0 | 1 | Même API que eBay.fr, un simple paramètre de marketplace |
| 18 | **Hardwareluxx — Marktplatz** ⭐ | T1 | 2 | Forum d'enthousiastes allemands, RAM haut de gamme fréquente |
| 19 | **ComputerBase — Marktplatz** | T1 | 2 | Idem, très sérieux |
| 20 | **Willhaben.at** | T2 | 2 | Leader autrichien |
| 21 | **Ricardo.ch** | T2 | 3 | Suisse, prix souvent élevés mais volume |
| 22 | **Tutti.ch** | T2 | 3 | Suisse |
| 23 | **Anibis.ch** | T1 | 3 | Suisse |
| 24 | **eBay.at** | T0 | 3 | Via API |

#### 🇧🇪🇳🇱🇱🇺 Benelux — livraison rapide vers la France

| # | Site | T | P | Note |
|---|---|---|---|---|
| 25 | **Marktplaats.nl** ⭐ | T2 | 2 | Le leboncoin néerlandais, très gros volume |
| 26 | **Tweakers.net — Vraag & Aanbod** ⭐ | T1 | 1 | **Pépite** : communauté hardware NL très active, annonces techniques précises, crawl facile |
| 27 | **2dehands.be** | T2 | 2 | Belgique FR/NL, même moteur que Marktplaats |
| 28 | **2ememain.be** | T2 | 2 | Version francophone du précédent |
| 29 | **eBay.nl** | T0 | 3 | API |

#### 🇬🇧🇮🇪 Îles britanniques

| # | Site | T | P | Note |
|---|---|---|---|---|
| 30 | **eBay.co.uk** ⭐ | T0 | 1 | API. Attention TVA/douane post-Brexit → à pondérer dans le score |
| 31 | **Overclockers UK — Member Market** ⭐ | T1 | 2 | Forum d'enthousiastes, excellentes annonces |
| 32 | **Gumtree.com** | T2 | 2 | Généraliste UK |
| 33 | **Adverts.ie** | T1 | 3 | Irlande (zone euro, pas de douane) |
| 34 | **Preloved.co.uk** | T1 | 3 | Petit |

#### 🇪🇸🇮🇹🇵🇹 Europe du Sud

| # | Site | T | P | Note |
|---|---|---|---|---|
| 35 | **Wallapop.es** ⭐ | T2 | 2 | Très gros volume espagnol, API interne exploitable |
| 36 | **Milanuncios.es** | T2 | 3 | Généraliste |
| 37 | **Subito.it** ⭐ | T2 | 2 | Leader italien |
| 38 | **eBay.it** | T0 | 3 | API |
| 39 | **OLX.pt** | T1 | 3 | Portugal |
| 40 | **CustoJusto.pt** | T1 | 3 | Portugal |
| 41 | **Segundamano / Vibbo** | T1 | 3 | Espagne |
| 42 | **Tom's Hardware IT — forum mercatino** | T1 | 3 | Communauté IT |

#### 🇸🇪🇳🇴🇩🇰🇫🇮 Nordiques — souvent sous-cotés, anglais courant

| # | Site | T | P | Note |
|---|---|---|---|---|
| 43 | **Blocket.se** ⭐ | T2 | 2 | Leader suédois |
| 44 | **Finn.no** | T2 | 2 | Norvège (hors UE → douane, à pondérer) |
| 45 | **DBA.dk** | T2 | 3 | Danemark |
| 46 | **Tori.fi** | T2 | 3 | Finlande |
| 47 | **Sweclockers — Marknad** ⭐ | T1 | 2 | Forum hardware suédois, très actif |
| 48 | **Bytbil / Kvdbil** | T1 | 3 | Marginal |

#### 🇵🇱🇨🇿🇸🇰🇭🇺🇷🇴 Europe centrale — les prix les plus bas d'Europe

| # | Site | T | P | Note |
|---|---|---|---|---|
| 49 | **OLX.pl** ⭐ | T1 | 2 | Énorme volume polonais, prix très bas |
| 50 | **Allegro Lokalnie** ⭐ | T1 | 2 | Pologne, section occasion |
| 51 | **Bazos.cz** | T1 | 2 | Tchéquie, HTML rudimentaire = crawl trivial |
| 52 | **Bazos.sk** | T1 | 2 | Slovaquie, même moteur |
| 53 | **Aukro.cz** | T1 | 3 | Enchères tchèques |
| 54 | **Jófogás.hu** | T1 | 3 | Hongrie |
| 55 | **Hardverapró.hu** ⭐ | T1 | 2 | **Pépite** : forum hardware hongrois, énorme sur les composants |
| 56 | **OLX.ro** | T1 | 2 | Roumanie |
| 57 | **Okazii.ro** | T1 | 3 | Roumanie |
| 58 | **OLX.bg** | T1 | 3 | Bulgarie |
| 59 | **Bolha.com** | T1 | 3 | Slovénie |
| 60 | **Njuškalo.hr** | T1 | 3 | Croatie |
| 61 | **Kupujemprodajem.com** | T1 | 3 | Serbie |
| 62 | **OLX.ua** | T1 | 3 | Ukraine — logistique compliquée, à évaluer |
| 63 | **Skelbiu.lt** | T1 | 3 | Lituanie |
| 64 | **SS.lv** | T1 | 3 | Lettonie |
| 65 | **Kuldkäpp / Osta.ee** | T1 | 3 | Estonie |

#### 🌍 Communautés & forums spécialisés — le meilleur ratio signal/bruit

| # | Site | T | P | Note |
|---|---|---|---|---|
| 66 | **Reddit r/hardwareswap** ⭐ | T0 | 1 | **API officielle gratuite**, JSON propre, zéro anti-bot. À faire en P1 |
| 67 | **Reddit r/buildapcsales** ⭐ | T0 | 1 | Idem |
| 68 | **Reddit r/hardwareswapfr** | T0 | 1 | Idem, francophone |
| 69 | **Reddit r/hardwareswapuk** | T0 | 2 | Idem |
| 70 | **Reddit r/hardwareswapEU** | T0 | 2 | Idem |
| 71 | **Reddit r/pcmasterrace (flair sale)** | T0 | 3 | Bruyant |
| 72 | **LinusTechTips — Buy/Sell** | T1 | 3 | Forum international |
| 73 | **Level1Techs — Marketplace** | T1 | 3 | Communauté technique |
| 74 | **AnandTech Forums — FS/FT** | T1 | 3 | Historique, encore actif |
| 75 | **Overclock.net — Marketplace** | T1 | 3 | Enthousiastes |
| 76 | **eBay Kleinanzeigen groupes Discord/Telegram FR** | T1 | 3 | Via bots publics |
| 77 | **Hardware.info — V&A (NL)** | T1 | 3 | Complément de Tweakers |
| 78 | **Forum-3DCenter.org (DE)** | T1 | 3 | Niche allemande pointue |
| 79 | **PCGamingWiki / Discord deals** | T1 | 3 | Optionnel |
| 80 | **Dealabs.com** (FR) ⭐ | T1 | 2 | Pas du C2C, mais les erreurs de prix marchands y remontent en minutes — très rentable |
| 81 | **Pepper.pl / mydealz.de / hotukdeals** | T1 | 2 | Même réseau que Dealabs, versions PL/DE/UK |

#### 🇺🇸🇨🇦 Amérique du Nord — pour la référence de prix, l'achat est secondaire

| # | Site | T | P | Note |
|---|---|---|---|---|
| 82 | **eBay.com** ⭐ | T0 | 1 | API. Sert surtout à **calibrer le prix de référence mondial** |
| 83 | **r/hardwareswap** (déjà #66) | — | — | — |
| 84 | **Craigslist** (SF, NY, LA, Seattle, Austin…) | T1 | 3 | RSS natif par ville → crawl trivial. ~15 villes = 15 sources |
| 85 | **OfferUp.com** | T2 | 3 | Mobile-first, API interne |
| 86 | **Mercari.com** | T2 | 3 | Volume |
| 87 | **Swappa.com** | T1 | 3 | Propre, modéré, peu d'arnaques |
| 88 | **Kijiji.ca** | T2 | 3 | Canada |
| 89 | **Facebook Marketplace US** | T3 | 3 | Même adapter que #4 |

#### 🔧 Sites de référence prix (pas des sources de deals)

Indispensables pour savoir ce qu'est un prix « normal » — ils fournissent la **borne haute** du modèle,
surtout au démarrage quand le corpus C2C est encore vide.

| # | Site | T | P | Note |
|---|---|---|---|---|
| 90 | **Amazon.fr/de** (neuf) | T3 | 2 | Prix neuf de référence |
| 91 | **LDLC.com** ⭐ | T1 | 1 | Prix neuf FR, HTML stable, sert de plafond |
| 92 | **Materiel.net** | T1 | 1 | Idem |
| 93 | **Topachat.com** | T1 | 2 | Idem |
| 94 | **Newegg.com** | T1 | 2 | Référence US |
| 95 | **Geizhals.de** ⭐ | T1 | 1 | **Comparateur allemand**, le meilleur référentiel prix hardware d'Europe |
| 96 | **Idealo.fr/de** | T1 | 2 | Comparateur |
| 97 | **PCPartPicker** | T1 | 2 | Historique de prix, très utile pour les tendances |
| 98 | **Caseking.de** | T1 | 3 | Revendeur DE |
| 99 | **Alternate.de** | T1 | 3 | Revendeur DE |
| 100 | **Mindfactory.de** | T1 | 3 | Revendeur DE, prix agressifs |

### Ce que cette liste implique

- **~40 sources sont en T0/T1** (API ou HTML simple) → crawl quasi gratuit, aucun risque de blocage.
  Les forums et les sites d'Europe centrale ont un HTML rudimentaire : ce sont les plus faciles.
- **Les OLX-likes partagent le même moteur** (OLX.pl/ro/bg/pt/ua, Bazos.cz/sk, 2dehands/2ememain,
  Marktplaats) → **un adapter générique paramétré couvre ~15 sources d'un coup**. Idem pour les
  forums vBulletin/XenForo (~12 sources) et les eBay via API (~7 sources en changeant un paramètre).
  Concrètement : **~100 sources ≈ 25 adapters à écrire**, pas 100. C'est ce qui rend la cible tenable.
- **Priorité 1 = 12 sources** qui couvrent déjà l'essentiel du volume exploitable :
  eBay (API multi-pays), leboncoin, Kleinanzeigen, Tweakers, Geektroc, les 3 subreddits, LDLC,
  Materiel.net, Geizhals.
- **Les « pépites » sous-estimées** : Hardverapró (HU), Tweakers V&A (NL), Sweclockers (SE),
  Hardwareluxx (DE), Geektroc (FR), Overclockers UK. Peu de concurrence d'acheteurs automatisés,
  vendeurs particuliers, prix parfois très anciens jamais réajustés à la flambée de la DDR5.
- **Dealabs & co (#80-81)** sont hors périmètre C2C mais je les garde : une erreur de prix marchand
  y est signalée en quelques minutes, et c'est typiquement là que se trouvent les vraies affaires
  sur du neuf.

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

Le 4070 (12 Go de VRAM) fait tourner un 14B quantifié en Q4 (~9 Go) à ~40 tok/s. La sortie fait
~80 tokens, donc ~2 s par annonce. À cette taille, la qualité d'extraction est très proche d'une
API payante.

Cette étape sert aussi à **traduire/normaliser** les annonces en polonais, tchèque, hongrois ou
suédois que les regex ne couvrent pas — ce qui débloque à lui seul une vingtaine des sources
d'Europe centrale et nordique listées en §3.

La file `parse:llm` a une concurrence de 1 et se met en pause si le GPU est déjà chargé (jeu en
cours) : les annonces non parsées attendent en `needs_review` et sont traitées plus tard.
Le pipeline ne bloque jamais.

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

**Bootstrap au démarrage** : tant que le corpus C2C est vide, aucun bucket n'atteint 30 observations
et rien n'est alerté. Les sources de référence prix (#90-100 du §3 : LDLC, Geizhals, PCPartPicker)
fournissent immédiatement une **borne haute** — un prix neuf. On peut donc alerter dès le jour 1 sur
la règle simple « occasion < 45 % du neuf équivalent », puis basculer sur le modèle médiane/MAD dès
que les buckets se remplissent (~1 semaine).

**Pondération géographique** : une annonce polonaise à -50 % n'est pas la même affaire qu'une
annonce lyonnaise à -40 % (port, risque, délai, recours). Chaque source porte un
`friction_score` (0-1) qui module le `deal_score` final : 1,0 pour la France en main propre,
~0,75 pour l'UE avec envoi, ~0,5 hors UE (douane). Tu peux le désactiver dans les filtres.

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

Files BullMQ, toutes sur le même Redis local :

```
crawl:http    concurrence 8   → sources T0/T1 (API, HTML simple) — léger
crawl:browser concurrence 4-8 → sources T2/T3 (Camoufox) — 400 Mo chacun
parse:llm     concurrence 1   → Ollama, en pause si le GPU est occupé
detect, notify, refresh:reference → concurrence 1, quasi instantanés
```

Le facteur limitant n'est jamais le CPU mais le **rate limiting volontaire** (§2) : on veut rester
lent. Les 100 sources tiennent largement dans une heure même à 4 navigateurs.

### Le cas « PC éteint » — le point de design central du tout-local

Sans VPS, il n'y a pas de dégradation gracieuse possible : quand le PC dort, tout dort. Deux règles
rendent ça acceptable :

1. **Coalescing au démarrage.** Si le PC a été éteint 12 h, on ne rejoue **pas** les 12 runs manqués.
   Le scheduler ne garde qu'un seul run par source (`removeOnComplete` + jobId déterministe par
   fenêtre horaire). Sans ça, l'allumage déclencherait une rafale de 1 200 requêtes qui ressemble
   très exactement à ce qu'un antibot cherche à détecter — c'est le meilleur moyen de se faire
   bannir en une fois.
2. **Démarrage échelonné.** Au boot, on étale les premiers runs sur 20 min au lieu de tout lancer
   à la seconde 0.

**Conséquence à assumer :** sur les annonces les plus chaudes (une RAM à -50 % sur leboncoin part
en quelques dizaines de minutes), tu ne verras que ce qui est passé pendant que le PC était allumé.
Si le PC tourne le soir et le week-end, tu couvres déjà les heures où les particuliers postent
le plus — c'est loin d'être rédhibitoire.

**Option pour plus tard (P7) :** planificateur de tâches Windows qui réveille le PC toutes les 3 h,
laisse le worker vider sa file, puis remet en veille. ~30 min d'allumage par cycle, et tu récupères
une couverture quasi 24/7 pour quelques euros d'électricité par mois.

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
   **Point clé du tout-local** : Telegram est sortant uniquement (le PC appelle l'API), donc tu reçois
   les alertes sur ton téléphone où que tu sois, sans exposer quoi que ce soit sur ta box.
2. **Web Push** (VAPID) depuis le dashboard — utile seulement si tu as le navigateur ouvert.
3. **Digest** du top 10 sur Telegram à chaque démarrage du PC + une fois par jour :
   « voilà ce qui est passé pendant que tu n'étais pas là ». Particulièrement adapté à un setup local
   intermittent.

Anti-spam : dédup par listing, max 20 alertes/h, regroupement si > 3 deals en 2 min.

---

## 10. Phasage

| Phase | Contenu | Sortie |
|---|---|---|
| **P0** | Monorepo (`apps/web`, `apps/worker`, `services/camoufox`, `packages/db`, `packages/core`), **un seul** `docker-compose.yml`, Drizzle + migrations, doc de setup WSL2 | `docker compose up` sur ton PC, dashboard vide sur `localhost:3000` |
| **P1** | Framework de crawl : ladder T0→T3, registre de sources, **5 sources pilotes** (eBay API, r/hardwareswap, Kleinanzeigen T2, Tweakers T1, leboncoin T3) | annonces réelles en base, dont leboncoin — le risque technique est levé dès la P1 |
| **P2** | Parsing specs (regex + LLM fallback) + garde-fous, backfill | `listing_specs` peuplée, précision mesurée sur 200 annonces annotées à la main |
| **P3** | Prix de référence + détection d'anomalie + `deals` | top deals interrogeable en SQL |
| **P4** | Front Next.js : dashboard, filtres, `/sources` | utilisable au quotidien |
| **P5** | Telegram + watches | notifié en < 5 min |
| **P6** | Montée à ~100 sources (§3) : d'abord l'adapter générique OLX-like (~15 sources d'un coup), puis les forums, puis le reste | couverture large |
| **P7** | Durcissement : circuit breakers, coalescing, tests de non-régression des sélecteurs, alerte "source muette", réveil programmé Windows optionnel | ça tourne sans surveillance |

Chaque phase est mergeable et utile seule. P1→P3 est le cœur de valeur ; P6 est du volume, pas du risque.

---

## 11. Coûts et ressources

| Poste | Coût récurrent |
|---|---|
| PC Windows | **déjà possédé** |
| Postgres, Redis, Firecrawl, Camoufox, Ollama, Next.js | 0 € (open source, en local) |
| Telegram Bot API | 0 € |
| Tailscale (accès mobile, optionnel) | 0 € |
| Taux de change (frankfurter.app) | 0 € |
| eBay Browse API (5 000 appels/j) | 0 € |
| API Reddit | 0 € |
| **Total** | **0 €/mois** |

**Le seul coût réel : l'électricité**, et il est marginal si le PC est allumé de toute façon.
En 24/7 avec du crawl léger (~60-90 W, le GPU ne travaillant que par à-coups) : ~10-16 €/mois.
En usage normal — allumé le soir et le week-end — le surcoût du crawl est de **quelques centimes** :
tu paies déjà l'allumage, et le crawl est de l'attente réseau.

### Budget mémoire sur les 32 Go

| Service | RAM |
|---|---|
| Postgres | ~300 Mo |
| Redis | ~100 Mo |
| Next.js | ~200 Mo |
| Worker Node | ~250 Mo |
| Firecrawl (api + worker) | ~500 Mo |
| Camoufox × 8 | ~3,2 Go |
| Ollama (RAM ; le modèle de 9 Go vit en **VRAM**) | ~2 Go |
| **Total** | **≈ 7 Go** |

Il te reste **~25 Go** pour ton usage normal. Avec `.wslconfig` plafonné à 12 Go, WSL2 ne peut
structurellement pas déborder. Le stockage est négligeable : ~2 Go de base après un an
(on ne stocke pas les images, seulement leurs URLs).

---

## 12. Risques

| Risque | Mitigation |
|---|---|
| Un site change son HTML → 0 annonce silencieusement | Canary : alerte si une source retourne 0 résultat 2 runs de suite alors qu'elle en retournait > 10 |
| Faux positifs (annonce de recherche, HS, arnaque) | Garde-fous §4 + seuil de confiance + badge scam |
| Référence prix instable au démarrage (peu de données) | Pas d'alerte tant que n < 30 par bucket ; bootstrap possible avec les prix neufs (Amazon/Newegg) comme borne haute |
| **Ban de l'IP résidentielle** — impacterait ta navigation perso, tu n'en as qu'une | **Le risque n°1 du plan.** Circuit breaker au 1er signal, 1 req/15-30 s, jamais de parallélisme par domaine, horaires 7h-minuit. Recours : redémarrage box, WARP |
| **PC éteint = tout est arrêté** (pas de VPS de secours) | Assumé. Coalescing + démarrage échelonné (§7), digest Telegram au réveil. Option réveil programmé (P7) |
| Rafale de requêtes au redémarrage | Coalescing des jobs en retard (§7) — un seul run par source, pas 12. Sans ça, c'est le ban assuré |
| Docker Desktop / WSL2 qui plante ou se met à jour | `restart: unless-stopped`, healthchecks, et le worker est idempotent (rien ne se perd, les jobs sont dans Redis persisté sur volume) |
| Windows qui redémarre pour une mise à jour | Docker Desktop au démarrage + heures actives configurées pour éviter les reboots surprise |
| Le deal est parti avant que tu voies l'alerte | Fréquence ↑ sur les 10 sources les plus rentables, Telegram en priorité, `recheck:active` pour marquer les vendus |

---

## 13. Décisions à valider

1. **Périmètre géographique** — combien des 100 sources du §3 on active. Trois options :
   **(a)** FR seul (~15 sources, main propre possible), **(b)** FR + DE + Benelux + UK (~35 sources,
   le meilleur ratio à mon avis), **(c)** Europe entière (~80 sources, prix les plus bas mais port
   et risque). *Ma reco : (b) pour démarrer, (c) en P6.*
2. **DDR5 seule ou DDR4 aussi ?** DDR4 a aussi flambé et le marché de l'occasion y est plus large.
   *Ma reco : les deux, avec un filtre par défaut sur DDR5.*
3. **Le PC tourne-t-il 24/7 ou seulement quand tu l'utilises ?** Ça ne change pas le code, seulement
   la réactivité. *Ma reco : usage normal + digest Telegram au démarrage ; on ajoutera le réveil
   programmé en P7 si tu trouves ça frustrant.*
4. **Dashboard : `localhost` seul, ou aussi accessible depuis ton téléphone via Tailscale ?**
   *Ma reco : ajouter Tailscale, c'est 10 min et gratuit.*

Aucune de ces questions n'est bloquante — je peux démarrer la P0 avec les valeurs recommandées
et tu ajustes ensuite (le registre de sources rend l'activation/désactivation triviale).

---

## 14. Ce que le tout-local implique

| Aspect | Setup payant/cloud | Ce plan (local, 0 €) | Impact réel |
|---|---|---|---|
| Furtivité | Camoufox | Camoufox | **aucun** — c'est le même outil |
| Orchestration | Firecrawl cloud | Firecrawl self-hosted | **aucun** |
| IP résidentielle | proxies à ~8 €/Go | **ta box** | **aucun sur la qualité** ; en échange, une seule IP à ménager |
| Résolution DataDome | `proxy: stealth` inclus | Camoufox + IP résidentielle | **aucun** — même recette |
| Parsing LLM | Haiku 4.5 | Qwen2.5-14B sur RTX 4070 | quasi équivalent sur une extraction aussi cadrée |
| **Disponibilité** | 100 % | **PC allumé uniquement** | ⚠️ **le seul vrai renoncement** |
| Accès distant | URL publique | Tailscale (et Telegram pour les alertes) | négligeable |
| Ops | zéro | 1 × `docker compose` | quelques heures de setup |

**Le seul renoncement du tout-local est la disponibilité.** Tout le reste est équivalent, voire
meilleur (l'IP résidentielle et le GPU sont des atouts que le setup cloud payant n'a pas
gratuitement). Et il se compense largement : les particuliers postent le soir et le week-end,
c'est-à-dire précisément quand ton PC est allumé.
