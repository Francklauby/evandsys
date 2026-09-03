# État des lieux — projet Evandsys (SaaS Dolibarr)

Rédigé le **2026-08-12** depuis le projet Windows `C:\Users\f.lauby\PhpstormProjects\evandsys`,
à destination de ce projet local Docker (`/home/flauby/evandsys`).
Les deux copies sont sur le **même commit** au moment de la rédaction (`cfdbfaac935`).

---

## Mise à jour de session — 2026-09-03

- **Chantier 3 SUPER PDP validé de bout en bout sur le VPS** (entité 7 `infansgroup`, sandbox) :
  bandeau → activation → OAuth 5 étapes (société fictive Tricatel/000000001) → callback →
  **retour sur le bon sous-domaine `infansgroup`** → statut « Connecté ». En base
  (`llx_facturex_superpdp_tokens`, `fk_entity=7`) : refresh durable présent, `token_expires_at`
  du jour, `superpdp_siren` **vide** (normal, `number_scheme=sandbox`), **`superpdp_account_id=9497`**
  → preuve que `fetchCompanyIdentity()` → `GET companies/me` marche et que le garde-fou SIREN
  n'est plus inerte.
- **Cause racine du blocage du 14/08 = fausse piste OPcache.** En réalité
  `FACTUREX_SUPERPDP_SANDBOX=1` était posé sur l'entité **1** (privée au maître, non héritée) au
  lieu de l'entité **0** → l'entité 7 ne voyait pas le flag → SIREN émis en sandbox →
  `invalid_request`. Même bug que le correctif local du 13/08, jamais reporté en prod. Corrigé :
  `UPDATE llx_const SET entity=0 WHERE name='FACTUREX_SUPERPDP_SANDBOX' AND entity=1;`. Le VPS a
  `opcache.validate_timestamps=On` + `revalidate_freq=2` → OPcache ne pouvait pas être en cause.
  Rien à changer sur le webhook.
- **Chantier 4 V2 (réception) validé de bout en bout sur le VPS** (entité 7, sandbox). Schéma posé
  (`llx_facturex_received` + colonne curseur `last_sync_received_invoice_id`), cron
  `SyncSuperPdpReceived::syncAll` enregistré en entité 1. Facture entrante injectée via l'autre
  société sandbox **000000002** (= entité 1, « Burger Queen », compte 9498) :
  `generate_test_invoice` (vendeur = société du token, acheteur = l'autre société auto) puis
  `POST /v1.beta/invoices` → livrée à 000000001. Le sync a archivé la ligne (vendeur Burger Queen,
  TTC 1863,79 €), le PDF (`superpdp_420664.pdf`) sous `DOL_DATA_ROOT/facturex/received/7/`, la page
  `admin/received.php` l'affiche + téléchargement OK, 2ᵉ passage sans doublon (dédup
  `uk_received_entity_invoice`), curseur avancé. **Deux bugs corrigés au passage** (1er runtime) :
  `8145097` (charger `files.lib.php`/`date.lib.php`, sinon fatale `dol_is_dir()` en cron) et
  `bb85cb8` (`expand[]=en_invoice.totals` invalide → HTTP 400 ; `totals` est une propriété directe
  de `en_invoice`).
- **Chantier 4 V3 cadré** (« que fait-on d'une facture IN reçue ») : statuts du cycle de vie
  renvoyés à SUPER PDP via `POST /v1.beta/invoice_events` (pas `PATCH /status`, fantôme). Périmètre
  initial « parcours standard » `fr:204/205/210/212` (prise en charge / approbation / **refus** /
  **encaissement**), action **manuelle** du client sur `received.php`, journal append-only
  `llx_facturex_received_event` + retry. Cohérent avec « on n'accuse pas réception » (rien
  d'automatique). Détail + points à trancher : brief facturex, « Chantier 4 V3 ». **À développer.**

## ⚑ Checklist go-live prod SUPER PDP (VPS)

État au **2026-09-03** : le VPS est **déjà basculé en prod au niveau global** —
`FACTUREX_SUPERPDP_SANDBOX = 0` sur l'entité **0**, avec un override `= 1` sur l'entité **7**
(`infansgroup`) qui reste la **voie de test sandbox** (l'entité surcharge l'entité 0, cf. §7).
Les endpoints (`api.superpdp.tech`) et le `client_id` sont partagés sandbox/prod ; le mode est
choisi au consentement SUPER PDP, et notre flag ne gouverne que la transmission du vrai SIREN.

Avant d'onboarder un **vrai client** en prod, rester vigilant sur :

- [ ] **Agrément prod côté SUPER PDP** — l'app/dashboard doit être activée en **production** (pas
      seulement sandbox). Même endpoint, mais sans ça la 1ʳᵉ activation client échouera. Préalable
      externe, hors code. **À confirmer.**
- [x] ~~**Chantier 4 V2 (réception)**~~ — **validé en runtime le 2026-09-03** (VPS, entité 7,
      sandbox). Recette d'injection d'une entrante : token de la 2ᵉ société sandbox 000000002
      (entité 1) → `generate_test_invoice` → `POST /v1.beta/invoices` → livrée à 000000001. Voir §8.
- [ ] **Entités de test 8-18 désormais en « prod »** — ne pas cliquer « Activer » dessus (vrai KYC),
      ou leur poser le même override `FACTUREX_SUPERPDP_SANDBOX=1, entity=N` pour tester.
- [ ] **`FACTUREX_OD_ALERT_EMAIL` vide** — l'alerte SIREN part sinon sur `MAIN_MAIL_EMAIL_FROM` (§6).
- [ ] **Rappel** : le contrôle de cohérence SIREN redevient actif en prod (désactivé en sandbox) —
      écart = LOG_WARNING + email OD + `oauth_warning=SIREN_MISMATCH`, **non bloquant**.

Rien ne part par erreur en prod aujourd'hui : `FACTUREX_AUTO_TRANSMIT` n'agit que sur une entité
avec token valide, et la seule qui en a un (7) est en sandbox.

---

## Mise à jour de session — 2026-08-14

- **facturex, 5 commits poussés** (repo `main`) : rework SIREN via `companies/me` (`b5ac1b9`),
  spec OpenAPI versionnée (`b2c603e`), chantier 4 V1 SIRET de réception (`6d2b6a4`), chantier 4 V2
  archivage des factures reçues (`d053265`), protocole de test à jour (`184b310`). Core : untrack
  de `settings.local.json` + `.gitignore` (`3c3ed6ec`, étanchéité LDLC).
- **Test SUPER PDP VPS entamé puis interrompu** (console VPS inaccessible, pare-feu). Entité de test
  unique = **7 `infansgroup`**, en **sandbox**. **Blocage** : l'authorize envoie `superpdp_company_number`
  en sandbox → `invalid_request`, alors que le code à jour (déployé par webhook, vert 14:38) ne le devrait
  pas → **bytecode périmé suspecté (OPcache) ou déploiement pas sur le bon commit**. Détail + plan de reprise :
  mémoire projet `test-superpdp-vps-blocage-opcache`. À reprendre depuis le domicile.
- **Double affichage corrigé** : l'encart V1 « SIRET non renseigné » dupliquait le bouton de la carte
  « Informations vendeur ». État vide de la carte réception allégé (message concis, sans bouton).

## Mise à jour de session — 2026-08-13 (local Docker)

- **Mail routé vers mailpit** : `MAIN_MAIL_SENDMODE=smtps` + serveur `mailpit:1025`, TLS off, sur
  entités 0 et 1 (le dump était en mode `mail`/`127.0.0.1:25` → mails perdus). UI **http://localhost:8026**
  (interne 8025 → 8026). Testé OK. Aucune entité cliente n'avait de surcharge mail.
- **`FACTUREX_SUPERPDP_SANDBOX` déplacé sur l'entité 0** (était à tort sur l'entité 1) → toutes les
  entités sont en sandbox.
- **Mot de passe `test1234`** posé sur tous les users `entity > 7` (bcrypt).
- **Entités 8 et 9 supprimées** (signups de test avortés, sans tiers). Purge des 10 tables portant
  leur empreinte + 1832 `user_rights` éparpillés à tort (artefact de tests, pas un bug d'isolation).
- **Test SUPER PDP parties 1-2 validées en local** (voir §8). Objectif SIREN **résolu par la doc** (§4).
- Règle de perm ajoutée : lecture/écriture MariaDB docker sur base `evandsys`.
- Détails opérationnels pour reprendre : mémoire projet `test-superpdp-local-setup`.

---

## 1. Ce qu'est le projet

Fournir Dolibarr **23.0** en SaaS : une instance, une entité Dolibarr par client,
un sous-domaine par client. Développement solo.

Deux volets métier :
- **Isolation multi-locataires** → module `entitydomain` (sous-domaine → entité, inscription client).
- **Conformité facturation électronique française** → module `facturex` (Factur-X + raccordement PDP).

Evandsys agit comme **OD** (opérateur de dématérialisation) adossé à **SUPER PDP**, PDP agréée DGFiP.

### Dépôts Git (distincts !)

| Dépôt | Chemin | Branche |
|---|---|---|
| Core (fork Dolibarr) | racine du projet | `Evandsys` |
| entitydomain | `htdocs/custom/entitydomain/` | `main` |
| facturex | `htdocs/custom/facturex/` | `main` |
| dolassist | `htdocs/custom/dolassist/` | `main` |
| smsgate | `htdocs/custom/smsgate/` | `main` |
| Site vitrine (evandsys.fr) | **hors projet** : `C:\Users\f.lauby\PhpstormProject\vitrine` (`PhpstormProject` au **singulier**) | — |

Toujours vérifier dans quel dépôt on se trouve avant de commiter.
**L'utilisateur pousse lui-même** : commiter oui, `git push` non.

Têtes de branches au 2026-08-12 :
- core `cfdbfaac935` — *fix: l'email de reinitialisation ne transporte plus de mot de passe*
- entitydomain `4c500ee` — *refactor: URL de confirmation par defaut definie une seule fois*
- facturex `ae148f7` — *feat: protocole de test SUPER PDP + cle SIREN la plus probable en tete*

---

## 2. ⚠ Priorité absolue : échéance légale du 1er septembre 2026

Au 1er septembre 2026, **toute entreprise assujettie à la TVA doit être capable de RECEVOIR
des factures électroniques** (`htdocs/custom/facturex/.claude/brief_reforme_2026.md`, ligne 6).

Point décisif du brief : pour un client, **la capacité à recevoir est acquise dès que son
compte SUPER PDP est actif** — le fournisseur adresse au SIRET, l'annuaire PPF route vers
SUPER PDP qui stocke. Notre code de récupération conditionne le confort, pas la conformité.
**Le chemin critique n'est donc pas du code : c'est que les clients soient raccordés.**

Ordre retenu le 2026-08-10 (importance vs difficulté) :

1. **Tester le volet SUPER PDP de bout en bout** — bandeau → activation → callback OAuth →
   retour sur le bon sous-domaine. Jamais déroulé en entier. Importance maximale, difficulté
   faible : c'est un test. Protocole écrit : `htdocs/custom/facturex/dev/test_superpdp.README.md`.
2. **Faire activer les clients existants, en août** — pas du code, et le plus sous-estimé :
   le **KYC SUPER PDP est obligatoire** (QR code, pièce d'identité, mandataire social), à la
   charge du client, avec son propre délai. Un client qui s'y met le 30 août n'y sera pas.
3. **Chantier 4 en V1** — afficher au client son adresse de réception (son SIRET) pour qu'il
   la communique à ses fournisseurs. Petit, directement lié à l'obligation.
4. **Chantier 4 en V2** — `GET /v1/invoices/received` + stockage ECM. Le gros du travail,
   utile, **mais pas exigé au 1er septembre**.
5. Le reste (§6). Aucune date.

**Réserve à lever** : « être capable de recevoir = avoir un compte PDP actif » vient du brief,
pas d'une lecture du texte de loi. À confirmer auprès de SUPER PDP, qui porte l'agrément —
la réponse peut réordonner ce classement.

---

## 3. Module `entitydomain` — état

Isolation multi-entités par sous-domaine HTTP. **Incompatible avec modMultiCompany**, pas de
table `llx_entity`.

- `llx_entitydomain` : mapping `subdomain` ↔ entity ID ↔ `fk_soc` (tiers porté par l'entité 1).
- `core/entitydomain_boot.php`, appelé depuis `master.inc.php` (bloc `mod_evandsys`, ~ligne 296,
  avant `$conf->setValues`). Force `$conf->entity` et `$_SESSION["dol_entity"]` d'après le host.
- Host à **moins de 3 segments** → sortie immédiate, entité inchangée (important en local, cf. §5).
- Sous-domaine `evandsys` = maître ; `?dol_entity=N` permet au super-admin un switch temporaire
  (non collé en session).
- Sous-domaine inconnu → redirection vers `ENTITYDOMAIN_REDIRECT_URL`.
- Users `entity=0` = super-admin, connectables depuis n'importe quel sous-domaine.

### Parcours d'inscription — double opt-in (livré le 2026-08-08)

Ordre inversé : **vitrine → email de confirmation → création Dolibarr**. Rien n'est créé tant
que le demandeur n'a pas prouvé qu'il relève l'adresse saisie.

1. `api/request_client.php` — dépose la demande dans `llx_entitydomain_request`, réserve le
   sous-domaine, envoie l'email de confirmation. Ne crée rien. Plafond par IP
   (`ENTITYDOMAIN_MAX_REQUESTS_PER_IP`, 5/h ; la vitrine transmet `X-Forwarded-For`).
   Ne renvoie jamais le jeton : il n'existe que dans l'email.
2. `confirmation.php` (vitrine) — soumet en POST automatiquement par JS. Jamais en GET :
   les analyseurs de liens des messageries consommeraient le jeton.
3. `api/confirm_client.php` — `claim()` (verrou atomique) puis provisioning, `release()` si échec.
4. `api/create_client.php` — **plus sur le chemin public** : provisioning interne uniquement
   (admin, reprise manuelle, scripts), il crée sans vérifier l'email.

Jeton stocké en SHA-256 seulement. TTL des liens magiques : **48 h** (ramené de 168 h).

### Sécurité des mots de passe (livré le 2026-08-10)

`pass_temp` **ne contient plus jamais un mot de passe**, dans aucun parcours : le provisioning
y écrit un jeton aléatoire de 32 caractères, et `pass_crypted` reçoit l'empreinte d'une valeur
aléatoire que personne ne conserve — le compte n'a donc aucun mot de passe utilisable avant que
le client n'en choisisse un.

L'email de réinitialisation ne transporte plus de mot de passe. Trois modifications
**solidaires, à ne pas dissocier** :
- corps et sujet par `EntityDomainMail::passwordResetEmail()` (repli natif si le module manque) ;
- lien vers le formulaire (`setnewpassword=1`) au lieu de `action=validatenewpassword`, qui
  promouvait `pass_temp` et renvoyait à la connexion ;
- `newpass1` rendu **obligatoire** dans `passwordforgotten.php` — sans quoi valider le formulaire
  à vide posait un mot de passe que le client ignore.

### Classes

- `EntityDomainProvisioning` — toute la création (tiers, contact, mapping, user, mot de passe,
  identité vendeur, email) + relecture SIREN/SIRET/fk_pays après création. Erreurs via
  `->error` / `->errorCode` / `->httpCode`.
- `EntityDomainRequest` — file d'attente : `claim`/`release`/`attachEntity`,
  `findExistingSpaceByEmail`, `isSubdomainAvailable`, `countRecentByIp`, `purgeExpired`.
- `EntityDomainMagicLink` — échéance des liens (`ENTITYDOMAIN_MAGICLINK_TTL_HOURS`).
- `EntityDomain` — `buildClientBaseUrl()` / `clientBaseUrlForEntity()`, partagés avec
  `facturex/oauth/callback.php`.

### Scripts CLI

- `scripts/magic_link.php <login|email>` — recalcule un lien magique et affiche le contexte
  (salage, algo, échéance). Outil de support et de diagnostic.
- `scripts/purge.php [--dry-run]` — demandes non confirmées + liens échus.
- **Lancer sous `sudo -u www-data`**, sinon l'écriture du journal échoue (avertissements PHP
  à chaque `dol_syslog`).

### Cron d'entretien (installé et validé le 2026-08-10, sur le VPS)

Travail planifié Dolibarr **job 82, entité 1**, type « appel d'une méthode d'une classe »,
appelant `EntityDomainCron::purge()`. Lanceur `cron_run_jobs.php` dans `/etc/cron.d/dolibarr-cron`,
toutes les 15 min, sous `www-data`. Chaîne complète vérifiée : cron → lanceur → job → purge.
Détails et pièges : `entitydomain/scripts/cron.README.md`.

---

## 4. Module `facturex` — état

Lire `htdocs/custom/facturex/.claude/brief_reforme_2026.md` avant tout développement.
Aucune dépendance Composer, pure PHP.

| # | Chantier | État |
|---|---|---|
| 2 | Champs conformité tiers + alertes | ✅ livré |
| 3 | Connecteur SUPER PDP (OAuth + transmission + polling) | ✅ livré, ✅ testé en sandbox (2026-06-18), ✅ OAuth bout-en-bout validé VPS (2026-09-03) |
| 4 | Réception factures fournisseurs | V1 (affichage SIRET) ✅ + V2 (archive des PDF reçus + page « Factures reçues ») ✅ livrés 2026-08-14, ✅ **testé VPS bout-en-bout 2026-09-03** (sandbox) ; génération de factures fournisseur Dolibarr ⬜ (chantier ultérieur) |
| 5 | Embed XML dans PDF Factur-X | ✅ livré |
| 6 | E-reporting B2C | ⬜ 2027 |

### Architecture OAuth multi-entités

- `redirect_uri` **fixe** sur le domaine maître (entity 1) :
  `{dol_main_url_root}/custom/facturex/oauth/callback.php`.
- L'`entity_id` est encodé dans le paramètre `state` (HMAC-SHA256).
- Après échange du code, redirection vers le sous-domaine client (via `llx_entitydomain`).
- Credentials OD (`FACTUREX_SUPERPDP_CLIENT_ID` / `_CLIENT_SECRET` / `_WEBHOOK_SECRET`) stockés
  en **entity=0** → lisibles depuis toutes les entités. `FACTUREX_AUTO_TRANSMIT` est par entité.
- Flow **OAuth 2.0 confidential client** : **PKCE NON requis** (un prototype PKCE a été ajouté
  puis retiré — ne pas le réintroduire sans test sandbox le prouvant).
- Durées : access_token 30 min, refresh_token 1 an glissant. Révocation RFC 7009 non implémentée.
- La voie **webhook a été abandonnée** (redondante avec le cron de polling).

### UI

- `admin/setup.php` : statut « Connecté » basé sur `isConnected()` (refresh_token durable), pas
  sur l'access_token 30 min. Bouton « Tester la connexion » → `testConnection()` (force un refresh) ;
  lien « Reconnecter » seulement après échec. Écran d'activation co-brandé Evandsys × SUPER PDP
  (`tpl/superpdp_activate.tpl.php`).
- **Bandeau** via hook `printCommonFooter` (contexte `toprightmenu`), limité aux **pages clés**
  (accueil `index.php` + `/compta/facture/list.php|card.php`) pour ne pas être omniprésent.
  Admins only, hors entité maître (`<=1`), hors setup, masquable par session.
- Outil dev : `dev/zap_superpdp.php` (admin only) casse/restaure (`?restore=1`) la connexion
  d'une entité pour visualiser le rendu « non connecté ».

### Contrôle de cohérence SIREN (2026-08-10, `4445ba0`) — livré mais **inerte**

Garde-fou contre un raccordement au nom d'une société tierce, au callback OAuth.
Découvert en chemin : `superpdp_siren` n'était **jamais écrit** (colonne présente, affichée par
`setup.php`, mais `saveToken()` n'enregistrait que les jetons).

- `exchangeCode()` capte SIREN et account id via `pickValue()`, tolérant sur le nom du champ ;
- `saveToken(..., $siren, $accountId)` persiste sans écraser avec du vide (un refresh ne doit pas
  effacer l'identité initiale) ;
- `checkSirenConsistency()` compare, même dérivation que `buildAuthorizeUrl()` ;
- écart **non bloquant** : LOG_WARNING + email `FACTUREX_OD_ALERT_EMAIL` + avertissement client
  (`oauth_warning=SIREN_MISMATCH`). Écart ≠ fraude (SIREN mal saisi, groupe multi-SIREN).

**✅ REWORK FAIT le 2026-08-14** (suite à la résolution du 2026-08-13 par la doc OpenAPI).
Le SIREN **n'est pas dans la réponse du token** (token OAuth standard) : il se lit par un appel
distinct `GET /v1.beta/companies/me` → champ **`number`** qualifié par **`number_scheme`**
(`fr_siren` | `be_numero_entreprise` | `sandbox`). Appliqué dans `SuperPdpConnector` :
- `pickValue()` **supprimée**, remplacée par `fetchCompanyIdentity($accessToken)` (GET
  `companies/me`, Bearer, parsing défensif, `null`+WARNING si injoignable/illisible).
- `exchangeCode()` appelle `companies/me` avec l'access_token frais et ne retient `number`
  comme SIREN **que si `number_scheme === 'fr_siren'`** ; `id` → `superpdp_account_id`.
- `buildAuthorizeUrl()` : **déjà conforme** (émet `superpdp_company_number` +
  `superpdp_company_number_scheme=fr_siren`, solidaires, gardés hors sandbox) — non touché.
- `checkSirenConsistency()` / `saveToken()` / `setup.php` : inchangés, la ligne token est
  désormais réellement peuplée → le garde-fou n'est **plus inerte**.

**✅ TESTÉ LIVE sur le VPS le 2026-09-03** (entité 7, sandbox) : OAuth réel → `companies/me` →
`superpdp_account_id=9497` persisté, `superpdp_siren` vide (sandbox). Le garde-fou n'est plus
inerte et le rework est prouvé en runtime. (Le blocage du 14/08 venait de la constante sandbox
posée sur l'entité 1 au lieu de 0, pas d'OPcache — voir §8 et la MÀJ du 03/09.)

Spec : `htdocs/custom/facturex/dev/superpdp_openapi.json` (copie locale) /
`https://api.superpdp.tech/openapi/superpdp.json`. Détail : voir mémoires projet
`superpdp-siren-via-companies-me` et `reference-superpdp-api`.

### GOTCHA déploiement SaaS

Ajouter un contexte de hook au descripteur n'a d'effet **qu'après réactivation du module** sur
chaque instance (`MAIN_MODULE_FACTUREX_HOOKS` n'est réécrite qu'à l'activation, et
`insert_module_parts()` fait un INSERT « ignore si existe » **sans delete** → une valeur globale
stale ne serait pas réécrite). Vérifié OK le 2026-07-17 : la valeur est écrite en **entity=0** et
contient déjà `toprightmenu`, donc le sujet est clos y compris pour les clients existants.

---

## 5. Ce projet local Docker — état de la mise en route

Constaté le 2026-08-12 à 23 h.

Pile `docker/docker-compose.yml` (projet compose `evandsys`), tous les conteneurs **up** :

| Service | Image | Exposition |
|---|---|---|
| `evandsys-nginx-1` | nginx:1.27-alpine | **http://localhost:8081** → `/var/www/htdocs` |
| `evandsys-php-1` | build `docker/php` (UID/GID 1000) | fpm 9000, `host.docker.internal` mappé |
| `evandsys-mariadb-1` | mariadb:11.4 | **3307** → 3306, base/user/pass `evandsys`, root `root` |
| `evandsys-mailpit-1` | axllent/mailpit | UI **http://localhost:8026** ; SMTP interne `mailpit:1025` |

- Sources montées : `../sources/evandsys` → `/var/www` (donc webroot = `/var/www/htdocs`).
- Dump : `docker/dumps/evandsys_2026-08-12.sql.gz`, monté sur `/dumps`.
  **Déjà chargé** : la base `evandsys` contient 283 tables.

### Reste à faire pour que ça tourne

1. **`htdocs/conf/conf.php` n'existe pas** (seul `conf.php.example` est présent) → Dolibarr n'est
   pas encore configuré. À créer avec : `dolibarr_main_db_host='mariadb'`, port 3306,
   base/user/pass `evandsys`, `dolibarr_main_url_root='http://localhost:8081'`,
   `dolibarr_main_document_root='/var/www/htdocs'`.
2. **Répertoire documents** : `dolibarr_main_data_root` doit pointer **hors du webroot**
   (ex. `/var/www/documents`, à créer et donner à l'UID 1000). nginx interdit déjà
   `^/(documents|conf)/`.
3. **Emails vers mailpit** : poser `MAIN_MAIL_SMTP_SERVER='mailpit'` / `MAIN_MAIL_SMTP_PORT=1025`
   **sur l'entité 0** (une entité cliente n'hérite de rien — voir §7), sinon les emails du dump
   partiraient vers le vrai SMTP.
4. **`ENTITYDOMAIN_REDIRECT_URL`** vient du dump et pointe sur evandsys.fr : sans effet sur
   `localhost` (moins de 3 segments → sortie immédiate du boot), mais il enverra dehors dès qu'on
   utilisera un host à 3 segments inconnu.

### Conséquence importante : les sous-domaines en local

`entitydomain_boot.php` sort immédiatement si le host a moins de 3 segments. Sur
`http://localhost:8081`, **on est donc toujours sur l'entité maître** — utile pour l'admin, mais
le parcours client (entités 2+) n'est pas atteignable tel quel.

Pour l'exercer localement : servir des hosts à 3 segments du type
`monclient.evandsys.localhost:8081` (le port n'est pas dans le premier segment, donc il ne gêne
pas) et déclarer le mapping dans `llx_entitydomain`. Le bloc nginx étant le seul du conteneur,
il répond à n'importe quel `Host` — aucune conf nginx à ajouter, juste la résolution DNS/hosts
côté Windows (`*.localhost` résout déjà sur certains systèmes, sinon ajouter les entrées).
Un host commençant par `evandsys.` est traité comme le **maître**.

### Ce que le local ne pourra pas rejouer tel quel

- **Callback OAuth SUPER PDP** : la `redirect_uri` doit être joignable depuis Internet et déclarée
  dans le dashboard SUPER PDP. `http://localhost:8081/...` ne conviendra que si SUPER PDP accepte
  une redirect_uri locale ; sinon le test du volet OAuth reste à dérouler **sur le VPS**
  (protocole : `facturex/dev/test_superpdp.README.md`).
- **dolassist** a besoin d'un **Ollama** (`llama3.2:3b`) ; sur le VPS il écoute en 127.0.0.1:11434.
  En local, `host.docker.internal` est déjà mappé dans le conteneur php → viser
  `http://host.docker.internal:11434` si un Ollama tourne sur l'hôte.
- Le **cron** Dolibarr n'est pas planifié dans cette pile : lancer `cron_run_jobs.php` à la main
  dans le conteneur php si besoin.

---

## 6. Reste à faire (hors priorité §2)

### Déploiement du Chantier 4 V2 (réception) — à faire à la mise en prod

Le code est livré mais **jamais exécuté en runtime** (aucun token valide en local). Étapes :
1. **Jouer les nouveaux SQL** — ils ne rejouent pas seuls (module déjà installé, cf. §7) :
   `sql/llx_facturex_received.sql`, `.key.sql`, `sql/upgrade/add_last_sync_received_invoice_id.sql`
   (idempotents), ou réactiver le module facturex sur l'instance.
2. **Enregistrer le cron** en **entité 1** : classe `SyncSuperPdpReceived`, fichier
   `custom/facturex/cron/SyncSuperPdpReceived.class.php`, méthode `syncAll`, ~15-30 min
   (même modèle que `SyncSuperPdpEvents`).
3. **Tester sur le VPS** avec un compte SUPER PDP réel ayant reçu une facture (`direction=in`) :
   PDF archivé sous `DOL_DATA_ROOT/facturex/received/<entity>/`, ligne dans `llx_facturex_received`,
   onglet « Factures reçues » + téléchargement. Non rejouable en local (mêmes raisons que le callback OAuth).

### Tests jamais déroulés

1. **Volet facturex / SUPER PDP** : bandeau sur accueil et factures, activation → callback OAuth →
   retour sur le bon sous-domaine. Vérifier au passage que la page de démarrage client est bien
   `index.php`, sinon ajouter son chemin aux pages clés dans `facturex/actions_facturex.class.php`.
2. **`purge.php --dry-run`** sur un lien réellement échu (la purge réelle est passée à la place le
   2026-08-10). Contrôle de confort, à couvrir sur une prochaine inscription de test : rejouer une
   ligne de suivi après coup ne teste plus le même chemin, `pass_temp` étant vidé.

### Chantiers et corrections

- **`FACTUREX_OD_ALERT_EMAIL`** est exposé dans l'écran OD (`926fd03`) mais **reste à renseigner** :
  vide, l'alerte part sur `MAIN_MAIL_EMAIL_FROM`.
- **Branche `!$changelater` de `send_password()`** : envoie toujours un mot de passe en clair.
  C'est l'action volontaire « envoyer un nouveau mot de passe » de la fiche utilisateur, qui
  n'offre aucun lien de choix. Priorité basse : ce n'est pas un parcours client.
- **Table de jetons propre + page de choix du mot de passe dans le module**, pour fermer
  « lecture base = prise de contrôle d'un locataire ». Aujourd'hui le secret est en base et son
  dérivé dans l'URL, l'inverse du bon sens. Jugé mauvais rapport travail/gain maintenant que
  `pass_temp` ne porte plus de mot de passe : résidu = quelques comptes en attente sur 48 h.
- **Réparer les tiers 11, 12, 13** : SIREN/SIRET/fk_pays vides, créés avant le correctif
  `idprof1`/`idprof2`/`country_id`. Reprise possible depuis les constantes `MAIN_INFO_*` de
  chaque entité.
- **Entités antérieures au 2026-08-10 : entités HTML en base.** `vitrine/siren.php` échappait les
  champs à la saisie, si bien que `&` et les apostrophes sont stockés `&amp;` / `&#039;` dans
  `llx_societe.nom`, `llx_entitydomain.label`, les `MAIN_INFO_*`, `llx_socpeople`, `llx_user` et
  les `payload` des demandes. **Corrigé à la source, existant non réparé** — et les sous-domaines
  déjà dérivés d'une valeur échappée non plus (entité 17 = `aampcevents` au lieu de `acevents`),
  ce qui ne se rattrape pas sans changer l'URL du client. Script `html_entity_decode` avec
  `--dry-run` à écrire **si un vrai client est concerné** ; sur les entités de test, sans objet.
- **Ménage cron** (sans urgence) : les entités clientes 7 à 15 portent des travaux planifiés créés
  par les descripteurs de `modFacture` / `modFournisseur` (`RecurringInvoicesJob`,
  `RecurringSupplierInvoicesJob`, `SendSmsReminders`…), tous en `ALWAYS_ON`, parcourus à chaque
  cycle, deux de plus par client créé. Le contrôle « module cron non activé dans l'entité » ne joue
  qu'au changement d'entité : le premier job d'une entité est annulé, les suivants s'exécutent.
  `MAIN_MODULE_CRON` est désormais en liste noire, mais cela filtre l'offre sans désactiver
  l'existant :
  `SELECT entity, value FROM llx_const WHERE name = 'MAIN_MODULE_CRON' ORDER BY entity;`

### Décisions déjà prises (ne pas rouvrir)

- **`ENTITYDOMAIN_CONFIRM_URL` en admin : écarté** — `https://evandsys.fr/confirmation.php` ne
  changera pas. Le défaut est porté par `entitydomainConfirmUrlBase()` dans
  `api/request_client.php` ; la constante reste un point de surcharge possible.
- **BCC en dur `xanork@hotmail.fr`** dans le provisioning : classé sans suite, c'est l'adresse
  personnelle de l'utilisateur.
- **Webhook SUPER PDP** : abandonné au profit du polling.
- **PKCE** : non requis, retiré.

### Suspendu

- **`get_profiles.php` de la vitrine renvoie HTTP 500** (récupération des profils depuis le site
  vitrine). Cause probable `API_KEY_NOT_CONFIGURED` ; investigation suspendue faute d'accès au
  `config.php` de la vitrine (le local pointe sur une URL de gabarit, la vraie conf est sur le VPS).

---

## 7. Pièges à connaître (coûteux, déjà payés)

- **Tout lien magique doit porter `&entity=`** : sans ça l'affichage marche mais l'enregistrement
  échoue silencieusement.
- **Les constantes Dolibarr sont par entité, et une entité cliente n'hérite de rien** : poser
  `MAIN_MAIL_*`, `MAIN_APPLICATION_TITLE` etc. sur l'**entité 0** pour qu'elles soient globales.
- **Hooks Dolibarr** : le fichier hook doit être dans `class/` (pas `core/hooks/`), et la sortie
  HTML passe par `$this->resprints` (pas `resPrint`).
- **`$langs->trans()` renvoie des entités HTML** : ne pas ré-échapper ; décoder avant injection
  dans du JS ou de l'`innerHTML`.
- **`Societe::create()`** n'écrit `siren`/`siret`/`fk_pays` que depuis `idprof1`/`idprof2`/
  `country_id`, jamais depuis les propriétés dépréciées.
- **Le dépôt local n'est pas l'instance testée** (règle née sur le VPS, à re-transposer ici) :
  devant un comportement runtime qui contredit le code, vérifier d'abord que l'instance a le
  commit, et penser à l'**OPcache**. Penser aussi aux `sql/*.sql` nouveaux : le module étant déjà
  installé, ils **ne rejouent pas seuls**.

### Couplage au core (à ne pas casser lors d'un merge upstream)

Blocs encadrés par `/* mod_evandsys */` … `/* fin_mod_evandsys */` dans :
`master.inc.php`, `main.inc.php`, `passwordreset.tpl.php`, `passwordforgotten.php`,
`user.class.php` (`send_password`).

Règles : rester dans `htdocs/custom/` autant que possible, étendre par hooks/triggers, encadrer
toute modification du core par ces commentaires + une ligne expliquant la raison, et garder les
scripts SQL **idempotents** (`IF NOT EXISTS` / `IF EXISTS`).

---

## 8. Ce qui est validé en runtime (ne pas re-tester à l'aveugle)

- **2026-08-08**, instance déployée : inscription en double opt-in de bout en bout — formulaire
  vitrine → email de confirmation → création après confirmation → email de bienvenue → choix du
  mot de passe → enregistrement OK. Donc : création tiers/contact/mapping/user, persistance
  SIREN/SIRET/pays, identité vendeur pré-remplie (`MAIN_INFO_*`), lien magique.
- **2026-08-10** (client `swan`, entité 15, user rowid 14) : inscription → confirmation →
  bienvenue → lien magique dont l'empreinte concorde avec `magic_link.php` → antidatage de
  `date_expire` → `purge.php` vide `pass_temp` et supprime la ligne de suivi → le lien affiche
  l'encart « Ce lien n'est plus valable », sans champs ni bouton → « Recevoir un nouveau lien » →
  réinitialisation native pointant sur le bon sous-domaine et connectant le client → `purge.php`
  retourne 0, donc n'a pas touché à la demande native.
- **2026-08-10**, entité 18 (`afileta`) : `pass_temp` = jeton de 32 caractères, échéance 48 h,
  lien magique fonctionnel jusqu'à la connexion.
- **2026-08-10** : chaîne cron complète (cron système → lanceur → job 82 → purge).
- **2026-06-18** : chantier 3 facturex testé en **sandbox** SUPER PDP.
- **2026-09-03** (VPS, entité 7 `infansgroup`, sandbox) : chantier 3 **de bout en bout** — activation →
  OAuth 5 étapes → callback → **retour sur le bon sous-domaine** → « Connecté ». Token persisté
  (`superpdp_account_id=9497`, `superpdp_siren` vide en sandbox) → rework `companies/me` prouvé en
  runtime, garde-fou SIREN plus inerte. Blocage du 14/08 = `FACTUREX_SUPERPDP_SANDBOX` sur entité 1
  au lieu de 0 (pas OPcache), corrigé.
- **2026-09-03** (VPS, entité 7, sandbox) : chantier 4 V2 **réception** de bout en bout — entrante
  injectée depuis 000000002 (entité 1) via `generate_test_invoice`+`POST /invoices`, archivée par
  `SyncSuperPdpReceived` (ligne `llx_facturex_received` vendeur Burger Queen TTC 1863,79 €, PDF sous
  `DOL_DATA_ROOT/facturex/received/7/`, page `admin/received.php` + téléchargement OK, dédup OK,
  curseur avancé). Bugs corrigés au 1er runtime : `8145097` (libs cron `dol_is_dir`/`dol_stringtotime`)
  et `bb85cb8` (expand `en_invoice.totals` invalide → HTTP 400).
- **2026-07-17** : visibilité du bandeau SUPER PDP chez les clients (profils + hooks en base).
- **2026-08-13** (local Docker, entité 18 `afileta`) : test SUPER PDP **parties 1-2** — bandeau rouge
  visible sur `index.php` et `compta/facture/list.php`, absent sur `societe/list.php`, sur le setup et
  sur l'entité maître ; écran d'activation co-brandé traduit (3 étapes + KYC) ; URL `authorize`
  conforme (`state=18:<hmac>`, `login_hint`, `superpdp_send_and_receive=receive`, pas de
  `superpdp_company_number` en sandbox, `redirect_uri`=domaine maître). Parties 4-5 (callback réel) →
  VPS.

---

## 9. Autres modules

- **`dolassist`** — agent conversationnel 100 % local (Ollama + RAG TF-IDF markdown, sans
  dépendance externe) pour aider les clients à configurer leurs modules. Fonctionnel depuis
  2026-05-19, confirmé 2026-06-15. Chat standalone `/custom/dolassist/chat.php` + bouton flottant
  via hook `printCommonFooter`. Inscrit dans `ENTITYDOMAIN_MODULE_ALWAYS_ON`
  (`entitydomain/conf/modules_conf.php`) → activé sur toute nouvelle entité cliente.
  Dépôt : https://github.com/Francklauby/DolAssist.git (remote `originDolAssist`).
- **`smsgate`** — passerelle SMS. Squelette minimal (`admin/setup.php`, `class/smsgatesms.class.php`,
  `core/modules/modSmsgate.class.php`, langs), **un seul commit** (`acbb56c`, initial). Non
  documenté ailleurs, état d'avancement à établir.

---

## 10. Doc à corriger

Le `CLAUDE.md` du module facturex marque encore le **chantier 3 « à développer »** alors qu'il est
livré et testé en sandbox.
