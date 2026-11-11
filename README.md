<h1>Kronland und Logigate</h1>

<p>
<img alt="Laravel 13" src="https://img.shields.io/badge/Laravel-13-FF2D20?logo=laravel&logoColor=white">
<img alt="PHP 8.4" src="https://img.shields.io/badge/PHP-8.4-777BB4?logo=php&logoColor=white">
<img alt="327 Pest-Tests" src="https://img.shields.io/badge/Pest-327%20Tests%20grün-22C55E">
<img alt="Nuxt 4" src="https://img.shields.io/badge/Nuxt-4-00DC82?logo=nuxt&logoColor=white">
<img alt="399 Vitest-Tests" src="https://img.shields.io/badge/Vitest-399%20Tests%20grün-22C55E">
</p>

Zwei Projekte, die ich im Oktober nebenbei mit Coding-Agenten gebaut habe. Beide laufen live,
beide benutze ich selbst. Ich hoste meinen Code selbst, deshalb ist dieses Repo der Index und
die Links gehen in meine Forgejo-Instanz. Alles dort ist ohne Anmeldung lesbar und klonbar.

**Fabian Wagner**, Blax Software e.U., Oberösterreich · fabian.wagner@blax.at

---

## Wenn Sie nur fünf Minuten haben

Diese vier Dateien zeigen, wie ich Backend-Code schreibe:

| Datei | Warum die |
|---|---|
| [`app/Game/Economy/Settle.php`](https://forgejo.blax.at/blax-software/game-village-politics-backend/src/branch/master/app/Game/Economy/Settle.php) | 22 Zeilen, die das ganze Zustandsmodell tragen. Reines PHP, kein Eloquent, keine Uhr. Deshalb ohne Datenbank testbar. |
| [`app/Actions/QueueBuild.php`](https://forgejo.blax.at/blax-software/game-village-politics-backend/src/branch/master/app/Actions/QueueBuild.php) | 50 Zeilen Use Case: fällige Events nachholen, Transaktion, `lockForUpdate()`, Regeln aus `app/Game/`. So sieht jede Mutation aus. |
| [`app/Game/ConfigLint.php`](https://forgejo.blax.at/blax-software/game-village-politics-backend/src/branch/master/app/Game/ConfigLint.php) | Beweist, dass die Spielkonfiguration keine Sackgassen enthält. Ein Dorf darf scheitern, aber nie feststecken. |
| [`tests/Feature/ConquestTest.php`](https://forgejo.blax.at/blax-software/game-village-politics-backend/src/branch/master/tests/Feature/ConquestTest.php) | Ein Feature-Test, der die Zeit mit `Carbon::setTestNow()` bewegt, statt zu warten. |

Die Architektur in lang: [`docs/architecture.md`](https://forgejo.blax.at/blax-software/game-village-politics-backend/src/branch/master/docs/architecture.md).

---

## Kronland

Browser-Strategiespiel in der Art von Die Stämme, mit einem Unterschied: ein Dorf hat eine feste
Menge Bauplatz statt einer festen Gebäudeliste. Der Spieler entscheidet, was hineinkommt, und
mehrere Instanzen desselben Gebäudetyps sind erlaubt. Ein Dorf wird so zum Bauerndorf und das
nächste zum Militärdorf.

**Spielen:** [village.blax.at](https://village.blax.at) ·
**Backend:** [Repo](https://forgejo.blax.at/blax-software/game-village-politics-backend) ·
**Frontend:** [Repo](https://forgejo.blax.at/blax-software/game-village-politics-frontend)

Laravel 13 auf PHP 8.4, MySQL, Redis, Sanctum, Pest. Frontend Nuxt 4 mit TypeScript und Tailwind.

| | |
|---|---|
| Commits | 116 Backend, 134 Frontend, alle zwischen 28.10. und 03.11.2026 |
| Code | 128 PHP-Dateien, 11.466 Zeilen in `app/` |
| Tests | **327 Pest-Tests grün, 2.176 Assertions, 8,4 Sekunden** |
| Schema | 25 Migrationen |
| API | 11 WebSocket-Controller |

### Die fünf Entscheidungen, die den Rest bestimmen

**Server-autoritativ.** Der Client schickt nur Absichten, etwa `village.build {village_id, type}`.
Der Server prüft Besitz, Kosten, Bauplatz, Warteschlange und Voraussetzungen und rechnet jede
Kosten-, Dauer- und Zeitangabe selbst. Ein manipulierter Client gewinnt nichts.

**Lazy State statt Ticks.** Rohstoffe, Loyalität und Rekrutierungsfortschritt sind
`stored + rate × elapsed`. Vor jeder Änderung wird der aktuelle Wert in die Zeile geschrieben,
dann die Änderung angewandt. Kein Cronjob läuft alle zehn Sekunden über alle Dörfer. Ein Dorf,
das niemand anschaut, kostet nichts.

**Zeitgesteuertes sind Zeilen in `game_events`,** die ein Worker in `(fire_at, id)`-Reihenfolge
abarbeitet. Bewusst keine verzögerten Queue-Jobs: die garantieren keine Reihenfolge, und bei
Spiellogik ist die Reihenfolge das Spiel. Ein Angriff, der vor dem Gebäudeausbau landet, hat ein
anderes Ergebnis als andersherum.

**`app/Game/` ist pures PHP.** Kein Eloquent, keine Facade, kein `now()`. Zeit und Konfiguration
kommen als Argumente herein. Jede Formel ist damit als Unit-Test prüfbar, ohne Datenbank und ohne
Zeitreise. Darüber liegen `app/Actions/` als Use Cases, `app/Engine/` für die Event-Verarbeitung
und dünne WS-Controller, die validieren, autorisieren, eine Action aufrufen und eine Resource
zurückgeben.

**WebSocket statt REST.** HTTP macht nur Login, Registrierung und Health-Check. Alles andere läuft
über native Controller aus meinem eigenen Paket
[`blax-software/laravel-websockets`](https://github.com/blax-software/laravel-websockets). Nach
dem Verarbeiten eines Events schiebt die Engine die Änderung mit `ws_whisper` an die Sockets des
Besitzers.

### Qualitätssicherung

Jede Formel in `app/Game/` hat einen Unit-Test, jede Action einen Feature-Test. Die Suite deckt
die 29 Bereiche ab, die das Spiel ausmachen: Bauwarteschlange, Rekrutierung, Kampf, Beute,
Eroberung, Markt, Barbarenwellen, Hunger, Lazarett, Stämme, Quests, Ranglisten, Saisons.

Tests laufen auf SQLite, Produktion auf MySQL. Nach jedem Deploy fährt
`scripts/e2e-edgerealm.sh` die WS-API einmal durch, in einem Wegwerf-Container mit eigener
Datenbank, damit Produktionsdaten unangetastet bleiben.

---

## Logigate

Editor und Simulator für Logikschaltungen, im Geist von Turing Complete. NAND ist das einzige
Gatter. Man zeichnet damit Schaltungen, verpackt sie als Bauteile und benutzt die in größeren
Schaltungen weiter, bis hinauf zu einer CPU. Alles rechnet live beim Zeichnen.

**Ausprobieren:** [logicate.blax.at](https://logicate.blax.at), kein Account nötig ·
**Repo:** [logigate](https://forgejo.blax.at/blax-software/logigate)

Nuxt 4 mit `ssr: false`, TypeScript, Tailwind v4, Web Audio für die Klänge statt Sounddateien.
72 Commits, **399 Tests in 37 Dateien grün.**

Transistoren werden ehrlich gezählt, ein CMOS-NAND zu vier Transistoren, über jede
Verschachtelungstiefe hinweg. Dazu eine Kampagne mit Freischaltlogik, ein Rätsel der Woche aus
der ISO-Kalenderwoche und eine öffentliche Bauteilbibliothek.

Die interessante Einschränkung heißt Kompatibilität: ich benutze die App täglich, also müssen
Projekte aus älteren Releases weiter laden und identisch rechnen.
[`tests/compat.test.ts`](https://forgejo.blax.at/blax-software/logigate/src/branch/master/tests/compat.test.ts)
sichert das mit eingefrorenen Fixtures ab.

**Gerade in Arbeit:** Der Serverteil wandert von Nitro auf Laravel, damit die Rätselprüfung,
Accounts und die Bibliothek dieselbe Grundlage bekommen wie Kronland. Der Server muss
Schaltungen selbst simulieren und Transistoren nachzählen, sonst kann man Punkte erfinden. Damit
der PHP-Port das bitgenau gleich tut, habe ich zuerst einen Konformitätskorpus aus der laufenden
TypeScript-Fassung gezogen:
[`tests/fixtures/golden.json`](https://forgejo.blax.at/blax-software/logigate/src/branch/master/tests/fixtures/golden.json)
hält 20 Schaltungen, 324 bewertete Durchläufe mit exaktem Ergebnis und Fehlertext, sechs
generierte Wochenrätsel und die Sequenzen des Zufallsgenerators fest. So wird "das PHP rechnet
dasselbe" ein Test statt einer Behauptung.

---

## Wie ich mit Coding-Agenten arbeite

Beide Projekte sind mit Claude Code entstanden, überwiegend Opus, dazu Codex für Zweitmeinungen
bei Architekturfragen. 116 Commits in sieben Tagen gehen nicht von Hand.

**Die Regeln stehen im Repo, nicht im Prompt.** Jedes Projekt hat eine `CLAUDE.md`, die die
Spielregeln für den Code festhält: server-autoritativ, lazy state, keine Queue-Jobs für
Spiellogik, `app/Game/` bleibt pures PHP. Dazu `docs/architecture.md` und `docs/roadmap.md`. Der
Agent liest das zu Beginn jeder Sitzung, und das ersetzt die Hälfte der Korrekturen, die man
sonst in jeden einzelnen Prompt schreibt.

**Tests sind die Leine.** Ein Meilenstein gilt als fertig, wenn seine Tests grün sind. Das ist
der Grund, warum sich der Agent über 116 Commits nicht wegdreht: er kann nicht behaupten, etwas
funktioniere, wenn die Suite rot ist. Die Entscheidung, `app/Game/` ohne Eloquent und ohne
`now()` zu bauen, ist auch eine Entscheidung für Agenten-Tauglichkeit, weil jede Formel ohne
Infrastruktur prüfbar wird.

**Kleine Commits, je ein Anliegen.** Das macht Reviews möglich und Reverts billig. Die
Commit-Messages sind beschreibend, man kann die Entwicklung daran nachlesen.

**Eigene Pakete statt Copy-Paste.** Was in mehreren Projekten vorkommt, wandert in ein Paket.
Dadurch hat der Agent überall dieselbe API vor sich, und ich pflege sie einmal.

**Was ich nicht delegiere:** Datenmodell und Spielregeln. Die lege ich fest, der Agent baut sie
aus. Wo ich das umgedreht habe, kam regelmäßig etwas heraus, das funktioniert und trotzdem falsch
ist.

---

## Laravel-Pakete

Quelloffen auf GitHub, acht davon auf Packagist, alle in Produktion im Einsatz.

| Paket | Was es macht |
|---|---|
| [laravel-websockets](https://github.com/blax-software/laravel-websockets) | WebSocket-Server mit nativen Controllern statt Broadcast-Events. Kronland und Learn ATC laufen darauf. |
| [laravel-ws](https://github.com/blax-software/laravel-ws) | Nachfolger auf dem ReactPHP-Kernel: RFC6455 auf der geteilten Event-Loop, Message-Handler, Channel-Pub/Sub. |
| [laravel-webrtc](https://github.com/blax-software/laravel-webrtc) | WebRTC-Signaling auf derselben Loop, mit austauschbarer Media-Engine. |
| [reactphp-kernel](https://github.com/blax-software/reactphp-kernel) | Der Unterbau darunter: Loop-Lebenszyklus, Socket-Accept, IPC, Signalbehandlung, sauberes Herunterfahren. |
| [laravel-shop](https://github.com/blax-software/laravel-shop) | Headless E-Commerce mit Lagerführung, Stripe und Produktaktionen. |
| [laravel-invoicing](https://github.com/blax-software/laravel-invoicing) | Rechnungen mit lückenloser Nummerierung, USt. oder Kleinunternehmerregelung, lokalisierte PDFs. |
| [laravel-files](https://github.com/blax-software/laravel-files) | Dateien hochladen, optimieren, ausliefern, an jedes Model hängen. |
| [laravel-mail](https://github.com/blax-software/laravel-mail) | SMTP und IMAP pro Postfach, Nachrichten als Threads, Öffnungs- und Klick-Tracking. |
| [laravel-addresses](https://github.com/blax-software/laravel-addresses) | Adressen weltweit, von der Flurbezeichnung bis zur Zimmernummer. |
| [laravel-roles](https://github.com/blax-software/laravel-roles) | Rollen und Rechte, auch mit Ablaufdatum. |
| [laravel-workkit](https://github.com/blax-software/laravel-workkit) | Die Helfer, die sonst in jedem Projekt neu entstehen. |
| [docker-laravel](https://github.com/blax-software/docker-laravel) | PHP- und Nginx-Images für Laravel, PHP 8.0 bis 8.4, auf Docker Hub. |

---

## Was ich sonst gebaut habe

Produkte und Kundenarbeiten, die laufen. **Der Quellcode ist hier nicht offen**, teils weil er
nicht mir allein gehört, teils weil es Kundenprojekte sind. Was Sie sehen, ist das Ergebnis.

- **[learn-atc.com](https://www.learn-atc.com)** — Trainingsplattform für Flugverkehrskontrolle.
  Laravel und Nuxt, WebSockets, LLM in Produktion mit RAG und Finetuning. Mein größtes Projekt,
  ein Joint Venture, ich verantworte dort die Technik.
- **[autism.blax.at](https://autism.blax.at)** — Verzeichnis für Autismus-Hilfe in Österreich,
  mit zwei im Browser ausgewerteten Selbsttests. Nuxt 4, Atkinson Hyperlegible, kein Tracking,
  keine Drittanfragen, bewusst ohne Bewegung im Layout.
- **[holzschlaegerung-hochstoeger.at](https://www.holzschlaegerung-hochstoeger.at)** — Kundenseite,
  Nuxt 4.
- **[blax.at](https://www.blax.at)** — die eigene Firmenseite, zweisprachig.

---

## Bekannte Grenzen

- **Kronland ist nicht fertig.** Die Roadmap ist nicht abgehakt, das Balancing ist grob, und die
  Oberfläche ist an mehreren Stellen Zweckmäßigkeit statt Gestaltung.
- **Eine Welt, wenig Last.** Das Lazy-State-Modell ist für Gleichzeitigkeit gebaut, aber nie unter
  echter Spielerlast gelaufen. Lasttests fehlen.
- **Die Kampfsimulation ist nicht ausbalanciert.** Die Formeln sind getestet, die Zahlen dahinter
  sind Platzhalter.
- **Logigate speichert vorwiegend lokal.** Projekte liegen im localStorage, im Account nur bei
  Anmeldung. Genau das ist der Grund für den Laravel-Umbau.
- **Keine CI-Pipeline.** Tests laufen lokal und vor dem Deploy, nicht auf einem Runner. Für ein
  Einpersonenprojekt reicht das, für ein Team nicht.

---

## Selbst starten

**Kronland Backend**

```bash
git clone https://forgejo.blax.at/blax-software/game-village-politics-backend.git
cd game-village-politics-backend
docker compose up -d
docker compose exec app php artisan migrate:fresh --seed
docker compose exec app php artisan test        # 327 Tests
```

Ohne Docker genügen PHP 8.3+ und Composer, die Tests laufen auf SQLite im Speicher:
`composer install && vendor/bin/pest`

**Logigate**

```bash
git clone https://forgejo.blax.at/blax-software/logigate.git
cd logigate && pnpm install
pnpm test        # 399 Tests
```
