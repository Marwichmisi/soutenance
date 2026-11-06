# Recherche — Quel LLM déporté pour comprendre missions FR + replanifier avec garanties, sans scriptage ?

Ticket : #4 (Part of #1). Date : 2026-11-06.
Exigence structurante : **Q10 — vrai LLM déporté qui décide, AUCUN scriptage par mots-clés, fallback mots-clés refusé.**

## 1. Question posée

Ordres simples en français via page web (« Explore A puis B, priorité victimes, rentre si batterie < 20 % »).
Cerveau sur PC portable (cloud API possible, ou local Ollama / Llama / Mistral / Phi). Bas niveau Arduino/ESP32 exécute.
Problématique : réviser les objectifs sous événements / incertitude / ressources **tout en garantissant exécutabilité et sécurité**.
3 événements démo : **obstacle, victime, batterie faible**.

## 2. Fondements — pourquoi le LLM seul ne suffit jamais (et c'est normal)

Trois papiers fondateurs disent la même chose : un LLM raisonne bien en langage, mais ne peut **pas** garantir à lui seul
qu'une action est faisable dans le corps et l'environnement du robot. Il faut un **ancrage (grounding) externe + validation**.

- **SayCan** (Ahn et al., 2022) : le LLM propose des actions en langage naturel, mais chaque proposition est **multipliée par une
  fonction de valeur d'affordance** apprise (ce que le robot *peut* faire ici et maintenant). Devise : « Do As I Can, Not As I Say ».
  Transposé à notre projet : le LLM propose un plan, et un **validateur externe** (schéma + whitelist + état capteurs) décide de ce qui
  est exécutable — le validateur ne fait que *rejeter/filtrer*, jamais choisir par mots-clés, donc Q10 est respectée.
- **ReAct** (Yao et al., 2022) : entrelacer **traces de raisonnement** et **actions** (appel d'outils / observation d'environnement).
  Le raisonnement aide à suivre et réviser le plan et à gérer les exceptions ; les actions ramènent de l'information du monde réel
  (au lieu d'halluciner). C'est exactement notre boucle démo : événement → observation → replanification → plan validé.
- **RAP** (Hao et al., 2023) : un LLM seul manque de « modèle du monde » interne pour simuler états futurs et conséquences ;
  RAP couple le LLM-agent à une recherche arborescente guidée par récompenses. Leçon pour nous : sans état explicite
  (batterie, position, zones interdites) fourni **dans le prompt à chaque replanification**, le LLM ne peut pas planifier sous ressources.

**Conclusion théorique** : garanties = LLM (décision sémantique) + schéma de sortie contraint + validateur symbolique pré-exécution
+ état monde explicite. Aucun de ces garde-fous n'est du « scriptage » : ils ne comprennent pas l'ordre, ils **vérifient** la sortie.

## 3. Mécanismes disponibles (sources primaires)

### 3.1 Sorties structurées / JSON Schema

- **Ollama** : paramètre `format` = JSON Schema sur `/api/chat` ; sortie contrainte au schéma. Conseils officiels :
  définir le schéma via Pydantic (`model_json_schema()`) ou Zod, ajouter « return as JSON » au prompt, `temperature: 0`
  pour du déterministe. Compatible OpenAI (`/v1` + `client.beta.chat.completions.parse`).
- **Mistral** : *Structured Outputs* + *Function Calling* documentés (5 étapes : déclaration d'outils → requête → arguments
  générés → exécution côté dev → réponse finale ; modèles Small/Medium/Large/Ministral compatibles).
- **OpenAI** : API *Function calling* (le modèle émet des arguments d'outil typés au lieu de texte libre).
- **Anthropic (Claude)** : API *Tool use* (même principe : outils déclarés, le modèle choisit et remplit les arguments).

Point clé : ces quatre mécanismes imposent la **forme** (JSON valide, champs typés), jamais le **fond** de la décision.
La décision reste du LLM → compatible Q10. Le fond est ensuite contrôlé par notre validateur (section 6).

### 3.2 Comparatif cloud (API) vs local (Ollama sur PC)

| Critère | Cloud API (OpenAI / Claude / Mistral) | Local (Ollama : Llama / Mistral / Phi, quantifiés) |
|---|---|---|
| Sortie plan JSON validé | Oui : function calling / structured outputs natifs, forte fiabilité FR nuancé | Oui : `format` = JSON Schema (blog officiel 12/2024), fiabilité très bonne avec `temperature: 0` + schéma Pydantic ; à tester par variante exacte au moment de l'implémentation |
| Whitelist d'actions + validation pré-exécution | Côté nous dans les deux cas (validateur Python identique) | Idem — le validateur est indépendant du backend LLM |
| Garde-fous (reformulation à valider humain, seuils batterie, zones interdites) | Côté nous dans les deux cas | Idem |
| Compréhension FR (« priorité victimes », « si batterie < 20 % ») | Excellente, y compris ordres ambigus | Bonne pour ordres simples et gabarités ; ordres ambigus/complexes = point à évaluer en tests (batterie de prompts FR) |
| Latence replanification | 1–5 s typiques + dépend du WiFi/salle (point sensible en démo 15 min) | Inférence locale seule (~secondes sur PC portable selon modèle/quantification) ; **pas de réseau** |
| Fonctionnement dégradé sans connexion | **Non** : pas de réseau = pas de cerveau → seul l'arrêt sécurisé local (ESP32) reste | **Oui** : cerveau complet offline ; mode dégradé = arrêt sécurisé + dernier plan valide |
| Coût (budget serré) | Facturation au token / abonnement ; coût récurrent | Gratuit après install ; coût = 0 € marginal |
| Confidentialité | Données (carte, positions) envoyées hors PC | Tout reste sur le PC |
| Risque projet | Clé API, quota, changement de modèle/pricing, panne réseau le jour J | PC portable un peu juste → choisir variante petite (8B quantifié) et valider latence |

## 4. Comportement sur les 3 événements démo (principe commun aux deux options)

État monde injecté à chaque appel (position, batterie %, obstacles perçus, victimes candidates + score, zones interdites) :

1. **Obstacle** (ultrasons) : l'ESP32 fait l'arrêt/évitement réflexe immédiat (sécurité, pas le LLM) ; le LLM reçoit
   `obstacle_en=(x,y)` et **replanifie l'objectif** (ex. contourner par waypoint C ou abandonner zone B et passer en A) en JSON validé.
2. **Victime** (YOLO + score) : le LLM reçoit `victime_candidate={position, confiance}` et, vu la consigne « priorité victimes »,
   **réordonne les objectifs** (interrompre l'exploration → approcher → signaler), toujours dans la whitelist d'actions.
3. **Batterie faible** (< 20 %, puis seuil critique) : le LLM reçoit `batterie=17%` et la règle « rentre si batterie < 20 % » ;
   il émet `RETURN_BASE`. **Garde-fou non-LLM en plus** : si batterie < seuil critique, l'ESP32/PC impose `STOP` + retour
   même si le plan LLM dit autre chose (sécurité > décision).

Dans les 3 cas : le LLM **décide** (vrai raisonnement, pas de `if "victime" in ordre`), le validateur **autorise ou rejette**.

## 5. Recommandation (1 choix)

> **LLM local via Ollama sur le PC (variante ouverte petite et quantifiée, ex. famille Llama 8B ou Mistral Small / Phi —
> variante exacte à figer en tests), sorties structurées JSON Schema (`temperature: 0`), validateur Python pré-exécution
> obligatoire, interface compatible OpenAI (`/v1`) conservée pour pouvoir basculer sur API cloud en secours/benchmark.**

Pourquoi ce choix par défaut :
- **Démo** : 15 min en salle universitaire, WiFi incertain → l'offline élimine le premier risque d'échec public.
- **Budget** : 0 € marginal, exigence du ticket.
- **Q10** : décision 100 % LLM (compréhension FR + arbitrage objectifs), garde-fous = vérification, pas scriptage.
- **Jury** : « tout tourne devant vous, sans Internet » est un argument fort ; le backend cloud reste montrable en annexe
  (même code, endpoint différent) pour prouver l'interchangeabilité.
- **Sécurité** : seuils batterie et zones interdites appliqués deux fois — (a) rappelés dans le prompt système,
  (b) **imposés** par le validateur et l'arrêt réflexe ESP32.

Condition de figeage (phase implémentation) : batterie de ~30 ordres FR (dont ambigus) × 3 événements ; mesurer taux de JSON
valides au 1er essai, respect des priorités, latence ; si < cible, monter en taille de variante ou passer le backend sur cloud.

## 6. Livrables types

### 6.1 Prompt système (exemple, FR)

```text
Tu es le cerveau de replanification d'un robot de recherche et sauvetage (salle connue, zones A et B).
Tu reçois : l'ordre de mission en français, l'état monde (position, batterie %, obstacles, victimes candidates avec score,
zones interdites), et la liste d'actions autorisées.
Tu décides : ordre des objectifs et actions. Réponds UNIQUEMENT avec un objet JSON conforme au schéma fourni (return as JSON).
Règles : ne propose que des actions de la liste ; si batterie < 20, le retour base est prioritaire sauf victime à portée
immédiate avec confiance > 0.8 ; n'envoie jamais le robot dans une zone interdite ; explique chaque étape en une phrase
dans le champ "pourquoi".
```

### 6.2 Schéma JSON du plan (exemple)

```json
{
  "type": "object",
  "properties": {
    "mission": { "type": "string" },
    "priorite": { "type": "string", "enum": ["victimes", "exploration", "retour_base"] },
    "etapes": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "action": { "type": "string", "enum": ["GOTO", "SCAN_ZONE", "APPROCHER_VICTIME", "SIGNALER", "ATTENDRE", "RETOUR_BASE", "STOP"] },
          "parametres": { "type": "object" },
          "pourquoi": { "type": "string" }
        },
        "required": ["action", "parametres", "pourquoi"]
      },
      "minItems": 1
    }
  },
  "required": ["mission", "priorite", "etapes"]
}
```

### 6.3 Validateur pré-exécution (pseudo-code Python — prouve l'exécutabilité)

```python
ACTIONS_AUTORISEES = {"GOTO", "SCAN_ZONE", "APPROCHER_VICTIME", "SIGNALER", "ATTENDRE", "RETOUR_BASE", "STOP"}
SEUIL_BATTERIE = 20          # % : retour base prioritaire
SEUIL_CRITIQUE = 10          # % : STOP imposé, non négociable
ZONES_INTERDITES = [...]     # polygones fournis par l'opérateur

def valider_plan(plan: dict, etat: dict) -> tuple[bool, str]:
    # 1. Conformité schéma (forme) — ex. jsonschema.validate(plan, SCHEMA)
    valider_schema(plan, SCHEMA_PLAN)                      # lève si JSON invalide
    # 2. Whitelist d'actions (fond vérifié, pas décidé par mots-clés)
    for etape in plan["etapes"]:
        if etape["action"] not in ACTIONS_AUTORISEES:
            return False, f"action inconnue: {etape['action']}"
        if cible_dans_zone(etape, ZONES_INTERDITES):
            return False, "cible en zone interdite"
    # 3. Ressource batterie (sécurité, indépendante du LLM)
    if etat["batterie"] < SEUIL_CRITIQUE:
        return False, "batterie critique: STOP imposé"
    if etat["batterie"] < SEUIL_BATTERIE and plan["priorite"] != "retour_base":
        if not victime_haute_confiance_a_portee(plan, etat):
            return False, "batterie < 20 %: RETOUR_BASE exigé"
    # 4. Reformulation à valider par l'humain (page web) avant envoi bas niveau
    afficher_reformulation(plan)                           # l'opérateur Accepte / Édite / Annule
    return True, "plan valide, en attente validation humaine"
```

En cas de rejet : on redemande au LLM **avec le motif de rejet dans le prompt** (boucle ReAct : raisonner → agir → observer),
max N essais, sinon repli sûr `STOP` + alerte opérateur. Le repli n'est pas un scriptage de compréhension : c'est l'arrêt de sécurité.

## 7. Limites et points à lever en implémentation

- Le LLM peut halluciner une cible hors carte : le validateur spatial (bornes + zones) est obligatoire, pas optionnel.
- Ordres FR ambigus (« explore vite ») : définir la sémantique des priorités dans le prompt + tester ; mesurer, pas supposer.
- Latence locale sur PC portable d'étudiant : quantifier (variante, quantification, tokens/s) avant de figer.
- Perte WiFi PC↔ESP32 : protocole + heartbeat + arrêt sécurisé = autre ticket (carte comm), pas celui-ci.

## 8. Sources

- Ollama — *Structured outputs* (blog, 06/12/2024) : `format` = JSON Schema, Pydantic/Zod, `temperature: 0`, compatibilité OpenAI. — <https://ollama.com/blog/structured-outputs>
- Ollama — *API Chat* (docs) : référence de `/api/chat`. — <https://docs.ollama.com/api/chat>
- OpenAI — *Function calling* (docs) : le modèle émet des arguments d'outil typés. — <https://developers.openai.com/api/docs/guides/function-calling>
- Anthropic/Claude — *Tool use* (docs) : outils déclarés, le modèle choisit et remplit les arguments. — <https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview>
- Mistral AI — *Function Calling* (docs) : 5 étapes, modèles Small/Medium/Large/Ministral compatibles. — <https://docs.mistral.ai/inference/conversations/function-calling>
- Ahn et al. — *Do As I Can, Not As I Say: Grounding Language in Robotic Affordances* (SayCan), arXiv:2204.01691. — <https://arxiv.org/abs/2204.01691>
- Yao et al. — *ReAct: Synergizing Reasoning and Acting in Language Models*, arXiv:2210.03629. — <https://arxiv.org/abs/2210.03629>
- Hao et al. — *Reasoning with Language Model is Planning with World Model* (RAP), arXiv:2305.14992. — <https://arxiv.org/abs/2305.14992>
