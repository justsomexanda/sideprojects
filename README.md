# Sideprojects: Kronland und Logigate

Zwei Projekte, die ich in den letzten Wochen nebenbei gebaut habe. Beide laufen live, beide
benutze ich selbst, und beide sind mit Coding-Agenten entstanden.

Der Code liegt in meiner eigenen Forge:

| Projekt | Repo | Live |
|---|---|---|
| Kronland, Backend | https://forgejo.blax.at/blax-software/game-village-politics-backend | https://server-village.blax.at |
| Kronland, Frontend | https://forgejo.blax.at/blax-software/game-village-politics-frontend | https://village.blax.at |
| Logigate | https://forgejo.blax.at/blax-software/logigate | https://logicate.blax.at |

## Laravel-Pakete

Was in mehreren Projekten vorkommt, wandert bei mir in ein Paket. Die hier laufen in Produktion,
acht davon liegen auf Packagist:

| Paket | Was es macht |
|---|---|
| [laravel-websockets](https://github.com/blax-software/laravel-websockets) | WebSocket-Server mit nativen Controllern statt Broadcast-Events. Kronland und Learn ATC laufen darauf. |
| [laravel-ws](https://github.com/blax-software/laravel-ws) | Nachfolger auf dem ReactPHP-Kernel: RFC6455 auf der geteilten Event-Loop, Message-Handler, Channel-Pub/Sub. |
| [laravel-webrtc](https://github.com/blax-software/laravel-webrtc) | WebRTC-Signaling auf derselben Loop, mit austauschbarer Media-Engine. |
| [reactphp-kernel](https://github.com/blax-software/reactphp-kernel) | Der protokollunabhängige Unterbau darunter: Loop-Lebenszyklus, Socket-Accept, IPC, Signalbehandlung, sauberes Herunterfahren. |
| [laravel-shop](https://github.com/blax-software/laravel-shop) | Headless E-Commerce mit Lagerführung, Stripe und Produktaktionen. |
| [laravel-invoicing](https://github.com/blax-software/laravel-invoicing) | Rechnungen mit lückenloser Nummerierung, USt. oder österreichischer Kleinunternehmerregelung, lokalisierte PDFs. |
| [laravel-files](https://github.com/blax-software/laravel-files) | Dateiverwaltung: hochladen, optimieren, ausliefern, an jedes Model hängen. |
| [laravel-mail](https://github.com/blax-software/laravel-mail) | SMTP und IMAP pro Postfach, Nachrichten als Threads, Öffnungs- und Klick-Tracking. |
| [laravel-addresses](https://github.com/blax-software/laravel-addresses) | Adressen weltweit, von der Flurbezeichnung bis zur Zimmernummer im Hochhaus. |
| [laravel-roles](https://github.com/blax-software/laravel-roles) | Rollen und Rechte, auch mit Ablaufdatum. |
| [laravel-workkit](https://github.com/blax-software/laravel-workkit) | Die Helfer, die sonst in jedem Projekt neu entstehen. |
| [docker-laravel](https://github.com/blax-software/docker-laravel) | PHP- und Nginx-Images für Laravel, PHP 8.0 bis 8.4, auf Docker Hub. |

Kronland zieht `laravel-websockets`, `laravel-workkit` und das Image `blaxsoftware/laravel`
daraus. Wer wissen will, wie ich Code schreibe, den niemand außer mir liest, schaut am besten
hier hinein.

## Wie es dazu kam

Ich habe keine Demo-Applikation für eine Bewerbung gebaut. Ich habe im Oktober angefangen, mit
Coding-Agenten an eigenen Sachen zu spielen, zwei Spiele gestartet, sie durchgetestet und
benutze sie seitdem selbst. Logigate öffne ich fast täglich, Kronland läuft auf einer eigenen
Welt. Das hier ist also kein Wochenendprojekt, das für einen Bewerbungsprozess entstanden ist,
sondern das, was ohnehin passiert ist.

---

## Kronland

Ein Browser-Strategiespiel in der Art von Die Stämme, mit einem Unterschied: ein Dorf hat eine
feste Menge Bauplatz statt einer festen Gebäudeliste. Der Spieler entscheidet, was hineinkommt,
und mehrere Instanzen desselben Gebäudetyps sind erlaubt. Ein Dorf kann so zum Bauerndorf
werden und das nächste zum Militärdorf.

**Backend: Laravel 13, PHP 8.4**, MySQL, Redis, Sanctum, Pest.
**Frontend: Nuxt 4, TypeScript**, Tailwind.

Gemessen am 11.11.2026:

- 116 Commits im Backend, 134 im Frontend, beides zwischen 28.10. und 03.11.
- 128 PHP-Dateien, 11.466 Zeilen in `app/`
- 25 Migrationen, 11 WebSocket-Controller
- **327 Pest-Tests, alle grün, 2.176 Assertions, 8,4 Sekunden**

### Architekturentscheidungen

**Server-autoritativ.** Der Client schickt nur Absichten, etwa `village.build {village_id, type}`.
Der Server prüft Besitz, Kosten, Bauplatz, Warteschlange und Voraussetzungen und rechnet jede
Kosten-, Dauer- und Zeitangabe selbst. Ein manipulierter Client gewinnt nichts.

**Lazy State statt Ticks.** Rohstoffe, Loyalität und Rekrutierungsfortschritt sind
`stored + rate × elapsed`. Vor jeder Änderung wird der aktuelle Wert in die Zeile geschrieben,
dann die Änderung angewandt. Es gibt keinen Cronjob, der alle zehn Sekunden über alle Dörfer
läuft. Ein Dorf, das niemand anschaut, kostet nichts.

**Zeitgesteuertes sind Zeilen in `game_events`,** die ein Worker in `(fire_at, id)`-Reihenfolge
abarbeitet. Bewusst keine verzögerten Queue-Jobs: die garantieren keine Reihenfolge, und bei
Spiellogik ist die Reihenfolge das Spiel. Ein Angriff, der vor dem Gebäudeausbau landet, hat ein
anderes Ergebnis als andersherum.

**`app/Game/` ist pures PHP.** Kein Eloquent, keine Facade, kein `now()`. Zeit und Konfiguration
kommen als Argumente herein. Dadurch ist jede Formel als Unit-Test prüfbar, ohne Datenbank und
ohne Zeitreise. Darüber liegen `app/Actions/` als Use Cases mit Transaktion und
`lockForUpdate()`, `app/Engine/` für die Event-Verarbeitung und dünne WS-Controller, die
validieren, autorisieren, eine Action aufrufen und eine Resource zurückgeben.

**Konfiguration statt Literale.** Spielwerte stehen in `config/game/*.php` plus Welt-Overrides.
Ein `ConfigLint` beweist, dass die Konfiguration keine Sackgassen enthält: ein Dorf darf
scheitern, aber es darf nie feststecken.

**WebSocket statt REST.** HTTP macht nur Login, Registrierung und Health-Check. Alles andere
läuft über WebSocket-Controller aus meinem eigenen Paket `blax-software/laravel-websockets`.
Nachdem die Engine ein Event verarbeitet hat, schiebt sie die Änderung mit `ws_whisper` an die
Sockets des Besitzers. Das Frontend ist deshalb ohne SSR gebaut: es ist ein Spiel hinter einem
Login, SSR bringt dort nichts, und derselbe statische Build trägt später einen Steam-Wrapper.

### Qualitätssicherung

Jede Formel in `app/Game/` hat einen Pest-Unit-Test. Jede Action hat einen Feature-Test, der die
Zeit mit `Carbon::setTestNow()` bewegt, statt zu warten. Die Feature-Suite deckt die 29 Bereiche
ab, die das Spiel ausmachen: Bauwarteschlange, Rekrutierung, Kampf, Beute, Eroberung, Markt,
Barbarenwellen, Hunger, Lazarett, Stämme, Quests, Ranglisten, Saisons.

Tests laufen auf SQLite, Produktion auf MySQL. Nach jedem Deploy fährt
`scripts/e2e-edgerealm.sh` die WS-API einmal durch (beitreten, anzeigen, bauen samt Push, Karte)
in einem Wegwerf-Container mit eigener Datenbank, damit die Produktionsdaten unangetastet
bleiben. Auf dem Live-Backend werden keine Testaccounts registriert.

---

## Logigate

Ein Editor und Simulator für Logikschaltungen, im Geist von Turing Complete. NAND ist das
einzige Gatter. Man zeichnet damit Schaltungen, verpackt sie als Bauteile und benutzt die in
größeren Schaltungen weiter, bis hinauf zu einer CPU. Alles rechnet live beim Zeichnen: grüne
Leitungen führen eine 1, graue eine 0.

**Nuxt 4** mit `ssr: false`, TypeScript, Tailwind v4, Web Audio für die Klänge, keine
Sound-Dateien. Der Server-Teil ist die API für Teilen-Links und Accounts.

- 72 Commits zwischen 17.10. und 28.10.
- **399 Tests in 37 Dateien, alle grün, 1,1 Sekunden**
- Transistoren werden ehrlich gezählt, ein CMOS-NAND zu vier Transistoren, über jede
  Verschachtelungstiefe hinweg. Jede Challenge merkt sich den Rekord.
- Kampagne mit Freischaltlogik, Rätsel der Woche aus der ISO-Kalenderwoche, öffentliche
  Bauteilbibliothek, Bestenliste.

Die interessante Einschränkung dort heißt Kompatibilität: ich benutze die App täglich, also
müssen Projekte aus älteren Releases weiter laden und identisch rechnen. `tests/compat.test.ts`
sichert das mit eingefrorenen Fixtures ab. Neue Projektfelder sind optional mit Default.

---

---

## Weitere Arbeiten, die öffentlich laufen

Nicht alles davon ist mein Code allein, und nicht alles ist quelloffen. Die Seiten laufen aber
live, man kann sie anschauen:

- **[learn-atc.com](https://www.learn-atc.com)**: Trainingsplattform für Flugverkehrskontrolle.
  Laravel und Nuxt, WebSockets, LLM in Produktion mit RAG und Finetuning. Mein größtes Projekt,
  ein Joint Venture, ich mache dort die technische Leitung.
- **[autism.blax.at](https://autism.blax.at)**: Verzeichnis für Autismus-Hilfe in Österreich,
  Beratung, Diagnose, Therapie, Gruppen, Arbeit, Termine, dazu zwei im Browser ausgewertete
  Selbsttests. Nuxt 4, Atkinson Hyperlegible, kein Tracking, keine Drittanfragen, bewusst ohne
  Bewegung im Layout. Eigenes Projekt, noch nicht öffentlich beworben.
- **[holzschlaegerung-hochstoeger.at](https://www.holzschlaegerung-hochstoeger.at)**:
  Kundenseite, Nuxt 4. Den Quellcode lege ich nicht offen, das ist nicht meiner allein.
- **[blax.at](https://www.blax.at)**: die eigene Firmenseite, Nuxt 4, zweisprachig.

## KI-Werkzeuge und Vorgehen

Beide Projekte sind mit **Claude Code** entstanden, überwiegend Opus, dazu Codex für
Zweitmeinungen bei Architekturfragen. 116 Commits in sieben Tagen gehen nicht von Hand.

Was bei mir funktioniert:

**Die Regeln stehen im Repo, nicht im Prompt.** Jedes Projekt hat eine `CLAUDE.md`, die die
Spielregeln für den Code festhält: server-autoritativ, lazy state, keine Queue-Jobs für
Spiellogik, `app/Game/` bleibt pures PHP. Dazu `docs/architecture.md` und `docs/roadmap.md`. Der
Agent liest das zu Beginn jeder Sitzung. Das ersetzt die Hälfte der Korrekturen, die man sonst
in jeden einzelnen Prompt schreibt.

**Tests sind die Leine.** Ein Meilenstein gilt als fertig, wenn seine Tests grün sind. Das ist
der eigentliche Grund, warum sich der Agent über 116 Commits nicht wegdreht: er kann nicht
behaupten, etwas funktioniere, wenn die Suite rot ist. Die Entscheidung, `app/Game/` ohne
Eloquent und ohne `now()` zu bauen, ist auch eine Entscheidung für Agenten-Tauglichkeit, weil
jede Formel ohne Infrastruktur prüfbar wird.

**Kleine Commits, je ein Anliegen.** Das macht Reviews möglich und Reverts billig. Die
Commit-Messages sind durchgehend beschreibend, man kann die Entwicklung daran nachlesen.

**Eigene Pakete statt Copy-Paste.** Was in mehreren Projekten vorkommt, wandert in ein Paket auf
Packagist. Kronland zieht `laravel-websockets`, `laravel-workkit` und das Docker-Image
`blaxsoftware/laravel` aus meiner eigenen Sammlung. Dadurch hat der Agent in jedem Projekt
dieselbe API vor sich, und ich pflege sie einmal.

**Was ich nicht delegiere:** Datenmodell und Spielregeln. Die lege ich fest, der Agent baut sie
aus. Wo ich das umgedreht habe, kam regelmäßig etwas heraus, das funktioniert und trotzdem
falsch ist.

## Bekannte Grenzen

- **Kronland ist nicht fertig.** Die Roadmap in `docs/roadmap.md` ist nicht abgehakt. Das
  Balancing ist grob, die Oberfläche ist an mehreren Stellen Zweckmäßigkeit statt Gestaltung.
- **Eine Welt, wenig Last.** Das Lazy-State-Modell ist für Gleichzeitigkeit gebaut, aber nie
  unter echter Spielerlast gelaufen. Lasttests fehlen.
- **Die Kampfsimulation ist nicht ausbalanciert.** Die Formeln sind getestet, die Zahlen
  dahinter sind Platzhalter.
- **Logigate speichert vorwiegend lokal.** Projekte liegen im localStorage, im Account nur, wenn
  man angemeldet ist. Wer den Browserspeicher leert, verliert sie.
- **Keine CI-Pipeline.** Tests laufen lokal und vor dem Deploy, nicht auf einem Runner. Für ein
  Einpersonenprojekt reicht das, für ein Team nicht.
- **Der Code liegt auf Forgejo, nicht hier.** Ich hoste meine Repos selbst. Dieses Repo ist der
  Index darauf.

## Startanleitung

**Kronland Backend**

```bash
git clone https://forgejo.blax.at/blax-software/game-village-politics-backend.git
cd game-village-politics-backend
docker compose up -d
docker compose exec app php artisan migrate:fresh --seed
docker compose exec app php artisan test        # 327 Tests
```

Ohne Docker genügen PHP 8.3+ und Composer, die Tests laufen auf SQLite im Speicher:

```bash
composer install && vendor/bin/pest
```

**Kronland Frontend**

```bash
git clone https://forgejo.blax.at/blax-software/game-village-politics-frontend.git
cd game-village-politics-frontend
pnpm install
pnpm run build && pnpm run preview
```

**Logigate**

```bash
git clone https://forgejo.blax.at/blax-software/logigate.git
cd logigate
pnpm install
pnpm test        # 399 Tests
```

Oder einfach auf https://logicate.blax.at klicken, es braucht keinen Account.

---

Fabian Wagner, Blax Software e.U., Oberösterreich.
fabian.wagner@blax.at
