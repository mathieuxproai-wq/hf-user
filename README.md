# hf-user — espace personnel d'exploration Hyperframe

Cet espace est **hors campagnes QA** (voir `../qa-hf/` pour les tests formels). Il sert à
préparer la suite : comment piloter Hyperframe efficacement pour créer des vidéos motion design
en limitant le coût de manipulation (tokens/temps), et comment s'y connecter sans friction.

**État** : sondes et mesures faites (2026-11-26). **Aucune implémentation** (client, templates,
serveur de scènes) — volontairement reportée. Voir `NOTES.md` pour les mesures et propositions.

## Fichiers
- `NOTES.md` — les deux volets d'exploration + cheat-sheet de commandes factuelle
- `SPEC_SERVICE_HYPERFRAME.md` — **contrat portable (HTTP) à consigner à l'agent de code** : endpoints,
  auth Bearer, architecture Termux vs sandbox, pièges QA, checklist de handover. Cross-platform :
  n'importe quel agent (Claude Code, Codex, Minis) appelle en `curl`.

## Découvertes déjà actées (mesurées, pas implémentées)
1. Connexion cold-start → CDP prêt : **0,48 s** ; evaluate WS : ~14 ms.
2. **Injection de scènes par HTTP** : un serveur dans le sandbox (`python3 -m http.server --directory`)
   est reachable depuis le WebView du device via `http://127.0.0.1:<port>/...` → `LOAD_URL` affiche
   la scène en ~0,6 s. `file:///sdcard` est **bloqué** (chrome-error, API 30+).
3. Écran : `keyevent 223` = extinction stable ; `26` = bascule + re-réveil GMS <8 s.
