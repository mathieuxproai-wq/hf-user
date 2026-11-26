# SPEC — Service de contrôle Hyperframe (contrat portable, agent-agnostique)

**But** : permettre à N'IMPORTE QUEL agent (Minis, Claude Code, Codex, script CI…) de piloter
Hyperframe Render et de créer des vidéos motion design, sans dépendance à un outil local.
**Principe** : le téléphone expose une API HTTP ; les agents ne font que du `curl`.
**Statut** : contrat + architecture — implémentation volontairement laissée à l'agent de code.

## Architecture

```
[Agent PC / Claude Code / Codex / Minis]  --HTTP+Bearer-->  [Service HF sur le téléphone]
                                                                |  shizuku: am / logcat / input
                                                                |  CDP WS (127.0.0.1:9222, sans Origin)
                                                                v
                                                         [Hyperframe Render]
[WebView scène]  --http://127.0.0.1:<port>/scene/...-->  [serveur de scènes (même service)]
```

- **Hôte du service** : Termux **recommandé** (persistant, `termux-wake-lock`, SSH déjà en place
  sur 192.168.1.135:8022). Le sandbox Minis est un fallback **session-scoped** (toute app tuée en
  arrière-plan = service mort) — ne pas s'appuyer dessus pour un usage cross-agent permanent.
- **Bind** : `127.0.0.1` par défaut ; `0.0.0.0` + **token Bearer obligatoire** pour le LAN.
- **Scènes** : le service sert aussi `GET /scene/<nom>.html` sur 127.0.0.1 — le WebView peut charger
  `http://127.0.0.1:<port>/scene/<nom>.html` via LOAD_URL (prouvé : ~0,6 s). `file://` est BLOQUÉ.

## Contrat d'API (proposé)

| Méthode | Chemin | Rôle | Réponse clé |
|---|---|---|---|
| GET | `/health` | liveness | `{ok, version, device}` |
| POST | `/render/start` | cold start (défauts 1280×720@60) | `{pid, displayId}` |
| POST | `/render/stop` | STOP propre (finalise le MP4) | `{frames, file}` |
| POST | `/render/load` | LOAD_URL à chaud `{url}` | `{ok, title}` (vérifier title !) |
| GET | `/render/status` | état pipeline + HUD | `{running, frames, fps, chrono, visibility, canvas}` |
| GET | `/render/fps?window=3` | mesure rAF/HUD | `{raf_fps, hud_fps}` |
| GET | `/video/latest` | métadonnées du MP4 | `{path, duration, avg_fps, size, md5}` |
| POST | `/screen` | `{state:"off"\|"on"}` | keyevent 223 / 224 |
| GET | `/scene/<f>` | scène motion design | fichier HTML/JS/CSS |

**Auth** : `Authorization: Bearer <token>` — obligatoire dès que le bind n'est pas 127.0.0.1.
**Erreurs** : codes HTTP + `{error:{code,message}}` ; ne jamais renvoyer 200 sur échec LOAD_URL.

## Tiers de canal — audit vs contrôle (la négociation n'est pas un goulot)

**Ne pas choisir un transport unique.** Superposer 3 tiers sur **un seul port** (modèle Chrome DevTools) :

| Tier | Protocole | Usage | Coût |
|---|---|---|---|
| 0 — découverte | `GET /json/list` (HTTP) | trouver la cible, liveness | 5-15 ms |
| 1 — contrôle | `POST /render/*`, `GET /status` (HTTP) | start/stop/load/état | ~10 ms |
| 2 — audit/montage | **WebSocket CDP brut** (Upgrade HTTP → pipe) | `Runtime.evaluate`, `Page.captureScreenshot`, DOM | handshake 12 ms + 14 ms/eval |

- **SSH / MCP / fonction d'app = façades optionnelles au-dessus**, pas des remplacements.
  MCP qui n'enveloppe que le Tier 1 = audit frame-par-frame perdu (ne pas faire).
- **Négociation** = 1 GET + 1 Upgrade ≈ **25 ms one-time** ; ensuite 14 ms/commande.
  Tunnel `ssh -L 9222:127.0.0.1:9222 -p 8022 …` : +1-3 ms RTT LAN, pas de goulot.
- **Montage** : le goulot n'est pas le canal mais `captureScreenshot` (~50-150 ms/frame).
  → tirer le **MP4 finalisé** (base64 + md5, secondes) et monter en **ffmpeg local**.
  CDP = audit live + 1 screenshot de contrôle, pas l'extraction de masse.

## Résilience réseau (LAN ↔ cellulaire)

**Le modèle « l'agent appelle le téléphone » casse sur cellulaire** : CGNAT opérateur (pas d'inbound)
+ IP change. Le CDP 9222 (device-localhost) et le rendu local ne sont PAS affectés — seul le lien
agent↔téléphone bascule. Solution = **rendez-vous sortant** (le téléphone appelle, jamais l'inverse) :

| Option | IP/URL | Résilience switch | Coût |
|---|---|---|---|
| **Tailscale** (recommandé, intégré Minis, à activer) | `100.x.y.z` stable | mesh WireGuard, reconnexion auto ~1-2 s | agent sur le même tailnet |
| ngrok / Cloudflare Tunnel | URL HTTPS publique | tunnel sortant, 0 config | exposition publique → token obligatoire |
| VPS + `ssh -R` | port sur VPS | auto-hébergé | maintenance |

- Requêtes HTTP **stateless** (nouveau TCP à chaque appel) → pas de goulot au switch.
- WS CDP : reconnexion via Tier 0 (`/json/list`) après tout changement d'interface.
- Token Bearer **obligatoire** dès que le lien n'est plus 127.0.0.1 (LAN, Tailscale, tunnel public).

## Pièges à coder dès la v1 (issus des campagnes QA test-01→03)

1. CDP WS **sans header `Origin`** (403 sinon) ; le proxy 9222 meurt avec le rendu → `/render/start` d'abord.
2. `fps` demandé **ignoré** par l'app (rAF suit le vsync) — ne pas exposer de promesse de fps.
3. Pas d'extra `duration` : le rendu va **jusqu'à STOP**.
4. `LOAD_URL` invalide = **silencieux** → `/render/load` doit vérifier `document.title`/href par CDP.
5. STOP = `start-foreground-service -a com.hyperframe.render.action.STOP` (le broadcast n'a aucun effet).
6. START dupliqué ignoré ; `force-stop` pendant rendu = orphelin `.pending` (jamais nettoyé).
7. Écran : `input keyevent 223` = off stable ; `26` = bascule + re-réveil GMS <8 s.
8. Transfert vidéo : base64 device + split + **vérif md5** (le stdout CLI wrappe à 76 cols — strip puis md5, jamais la longueur).
9. Scènes : `http://127.0.0.1:<port>/scene/...` uniquement ; `file:///sdcard` → chrome-error.

## Checklist de handover (pour l'agent de code)

- [ ] Service Python stdlib (http.server + subprocess shizuku + client WS CDP) — ~250 lignes.
- [ ] Token au boot (env `HF_TOKEN`), bind configurable, CORS `*` si usage navigateur.
- [ ] `termux-wake-lock` + log rotation ; healthcheck toutes les 30 s.
- [ ] Tests : smoke `curl /health`, cycle start→load→status→stop, assert md5 du MP4.
- [ ] README avec exemples curl prêts à coller pour Claude Code / Codex.