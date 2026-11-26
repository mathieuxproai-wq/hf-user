# NOTES hf-user — scripting motion design & connexion (exploration, pas d'implémentation)

Date : 2026-11-26 · Sondes réalisées ce jour (chiffres réels, sources : sondes de cette session).

---

## Volet 1 — Améliorer/faciliter le scripting ET réduire les tokens (création vidéo motion design)

### Où partent les tokens aujourd'hui (observé sur test-02/test-03)
| Poste de coût | Observé | Part estimée |
|---|---|---|
| Pilotage dispersé (am/logcat/attentes en appels séparés) | ~8-10 appels par cycle rendu | ~35 % |
| Transfert vidéo (base64+split+assemblage) | 6 appels dont 2 échecs de parsing | ~25 % |
| Sondes CDP répétées / réécrites à chaque fois | 3+ appels, scripts réécrits | ~15 % |
| Sorties non filtrées (dumps bruts retranscrits) | occasionnel mais lourd | ~15 % |
| Redécouverte du protocole (commandes, extras, pièges) | à chaque session si non noté | ~10 % |

### Leviers identifiés (PROPOSITIONS — rien n'est implémenté)
1. **Client unique `hf.py` (stdlib, ~150 lignes)** — sous-commandes : `start/stop/status/eval/fps/scene/pull`.
   Un appel shell = une action composée (aujourd'hui : 3-6 appels). Estimation ÷3 sur un cycle complet.
2. **Scènes servies par HTTP depuis le sandbox** — PROUVÉ ce jour :
   `python3 -m http.server 8777 --bind 127.0.0.1 --directory <dir>` + `LOAD_URL http://127.0.0.1:8777/scene.html`
   → scène affichée en ~0,6 s. Éditer une scène = éditer un fichier local + 1 LOAD_URL (hot-reload,
   sans redémarrer le rendu). Permet CSS/JS/assets externes. **Ne pas utiliser `file:///sdcard`** (bloqué).
3. **Scène paramétrique + JSON de paramètres** — la scène lit `window.HF_PARAMS` au boot (ou reçoit
   les params par CDP `Runtime.evaluate`). Une variante motion design = ~200 tokens de JSON au lieu
   de 1-2k tokens de HTML/JS réécrit.
4. **Paramétrage à chaud par CDP** — déjà prouvé (test-02 : HUD lu/écrasable via evaluate) :
   couleurs, vitesse, nombre de particules modifiables pendant le rendu, sans LOAD_URL.
5. **Discipline de sortie** — toute mesure → JSON une ligne ou fichier + résumé ; logcat toujours
   `grep` + `tail` ; jamais de dump brut dans la conversation.
6. **Transfert standardisé** — la routine `fetch_t03.py` (base64 device + split 2 Mo + assemblage +
   **md5 device vs local**) marche du premier coup une fois écrite ; la généraliser en `pull <remote> <local>`
   plutôt que la réécrire (les 2 échecs d'aujourd'hui = parsing d'enveloppe, pas de transport).
   Note piège : le stdout du CLI contient le wrapping 76 cols du base64 → strip whitespace PUIS vérifier par md5, jamais par longueur.
7. **Batch par scénario connu** — quand le déroulé est acquis (start→wait→probe→stop), un script le fait
   en 1 appel ; réserver les appels unitaires aux imprévus.

### Gains attendus (estimation honnête, à valider à l'usage)
- Cycle rendu+mesure : ~10 appels → 2-3.
- Variante de scène : ~1-2k tokens → ~200.
- Transfert vidéo : 6 appels → 1.
- Soit roughly ÷3 sur une session type, l'essentiel du gain venant de (2)+(3) pour la créations de variantes.

---

## Volet 2 — Se connecter facilement et très rapidement

### Mesures réelles (06:10, device idle, screen off)
| Étape | Durée |
|---|---|
| `force-stop` → `am start MainActivity` → `/json/version` répond | **0,47 s** |
| Handshake WebSocket (sans header Origin) | **12 ms** |
| WS + `Runtime.evaluate` aller-retour | **14 ms** |
| **Total cold → première commande exécutée** | **0,48 s** |
| `LOAD_URL` scène HTTP visible | ~0,6 s |
| STOP → port 9222 fermé | immédiat |

→ **La latence brute n'est pas le problème** : la connexion complète coûte une demi-seconde. Le coût
est dans le **nombre d'étapes manuelles** : vérifier le port, (re)démarrer, attendre, choisir le target,
refaire le handshake sans Origin. Aujourd'hui : 3-5 appels avant la première commande utile.

### Chemin rapide proposé (non implémenté) — pseudo-code
```
connect():
    si http://127.0.0.1:9222/json/version répond (<200 ms) : prêt          # warm path
    sinon: shizuku am start MainActivity ; poll 100 ms jusqu'à 200        # cold path, garanti <1 s
    cible = /json/list[0] ; ws = /devtools/page/<id>  (SANS header Origin)
    eval("1+1") == 2 → prêt
```
- Warm path : ~50 ms. Cold path : ~0,5 s. Un seul appel shell dans les deux cas.
- Après STOP, CDP est fermé : **toute reconnexion suppose un rendu actif** — le proxy vit avec le service.
- `hf.py connect` (volet 1) fusionnerait les deux volets : connect + eval + status en une commande.

---

## Cheat-sheet factuelle (référence, réduit la redécouverte)

| Action | Commande |
|---|---|
| START (défauts 1280×720@60) | `am start -n com.hyperframe.render/.MainActivity` |
| LOAD_URL | `am start-foreground-service -n com.hyperframe.render/.HeadlessRenderService -a com.hyperframe.render.action.LOAD_URL --es url <URL>` |
| STOP | `am start-foreground-service -n com.hyperframe.render/.HeadlessRenderService -a com.hyperframe.render.action.STOP` |
| CDP HTTP | `http://127.0.0.1:9222/json/list` → target id |
| CDP WS | `/devtools/page/<id>` **sans header Origin** (403 avec) |
| STATS encodage | `logcat -d -s HyperframePoC | grep STATS` (+120 frames / Δt) |
| Vidéo de sortie | `/sdcard/Download/hyperframe_poc.mp4` (écrasée au START ; STOP = finalisation) |
| Écran off stable | `input keyevent 223` (SLEEP) — `26` = toggle + re-réveil GMS |
| Écran on | `input keyevent 224` |
| Serveur de scènes | `python3 -m http.server <port> --bind 127.0.0.1 --directory <dir>` (sandbox) |

Pièges : pas de `duration` (rendu jusqu'à STOP) ; fps demandé ignoré (rAF suit le vsync) ;
START dupliqué ignoré ; LOAD_URL invalide = silencieux (vérifier par CDP, pas par log) ;
force-stop pendant rendu = orphelin `.pending`.
