# Plan : Multi-error recovery pour l'analyseur sémantique Vyper

> **Pour Hermes :** Utiliser `writing-plans` + `subagent-driven-development` pour implémenter tâche par tâche.

**Objectif :** Collecter toutes les erreurs sémantiques (typechecker, fonctions, imports, etc.) avant de les lever, au lieu de s'arrêter à la première.

**Architecture :** Propager un `ExceptionList` unique à travers tout le pipeline d'analyse (`module.py` → `local.py` → visiteurs), accumuler les erreurs dans chaque phase, lever à la fin.

---

## État des lieux

### Ce qui est déjà fait
- `vyper/ast/parse.py` : multi-error pour `python_ast.parse` (notre PR `fix/multi-error-collector`)
- `module.py:485 _visit_nodes_looping()` : collecte déjà les erreurs de déclarations module-level dans un `ExceptionList`
- `module.py:984 _compute_override_discrepancies()` : retourne déjà un `ExceptionList`
- `local.py:77` : `ExceptionList` utilisé par fonction, mais levé à la fin de chaque fonction

### Ce qui manque
- Les erreurs **dans les corps de fonction** sont levées une par une (par fonction)
- Les erreurs de **typechecking** (`utils.py:52`) sont levées immédiatement
- Les erreurs de **validation post-analyse** (`_validate_*` dans `module.py`) utilisent `raise` direct

---

## Approche

Modifier `analyze_functions` (local.py) pour collecter toutes les erreurs de toutes les fonctions dans un seul `ExceptionList`, et ne lever qu'à la fin. Puis remonter ce `err_list` dans `analyze_modules` (module.py).

### Fichiers à toucher

| Fichier | Rôle |
|---------|------|
| `vyper/semantics/analysis/local.py` | Cœur du changement : `analyze_functions` retourne un `ExceptionList` |
| `vyper/semantics/analysis/module.py` | `_analyze_module_bodies` collecte et lève |
| `vyper/semantics/types/module.py` | `validate_implements` (déjà partiellement ok) |
| `tests/functional/syntax/` | Ajouter des tests sémantiques multi-erreurs |

### Ce qu'on ne touche PAS
- `_visit_nodes_looping` dans module.py (déjà bon)
- `_compute_override_discrepancies` (déjà bon)
- Le parsing lui-même (déjà fait dans la PR `fix/multi-error-collector`)

---

## Tâches

### Tâche 1 : Explorer `analyze_functions` dans `local.py`

**Fichiers :**
- Lire : `vyper/semantics/analysis/local.py` (fonction `analyze_functions`, `_analyze_function_r`)

**Objectif :** Comprendre le flux exact : comment `err_list` est créé, peuplé, et levé.

### Tâche 2 : Modifier `analyze_functions` pour retourner un `ExceptionList`

**Fichiers :**
- Modifier : `vyper/semantics/analysis/local.py:77-88`

**Objectif :** Au lieu de lever `err_list.raise_if_not_empty()` à la fin de CHAQUE fonction, accumuler dans un `ExceptionList` global et le retourner.

### Tâche 3 : Modifier `_analyze_module_bodies` pour propager

**Fichiers :**
- Modifier : `vyper/semantics/analysis/module.py:125-141`

**Objectif :** Récupérer le `ExceptionList` retourné par `analyze_functions`, le fusionner avec les erreurs de `_validate_exports_uses`, `_validate_initialized_modules`, `_validate_used_modules`.

### Tâche 4 : Convertir les `raise` directs en `err_list.append`

**Fichiers :**
- Modifier : `vyper/semantics/analysis/module.py` (fonctions `_validate_*`)

**Objectif :** Remplacer les `raise XxxException(...)` par `err_list.append(XxxException(...))` dans les validateurs post-analyse.

### Tâche 5 : Ajouter des tests sémantiques multi-erreurs

**Fichiers :**
- Créer : `tests/functional/syntax/test_multi_semantic_error.py`

**Objectif :** Un fichier Vyper avec plusieurs erreurs sémantiques (variable inconnue, type mismatch, `while` invalide...) → vérifier qu'elles sont toutes remontées.

---

## Validation

```bash
cd /home/olyno/Documents/projects/vyper-audit
.venv/bin/python -m pytest tests/functional/syntax/test_multi_semantic_error.py -v
.venv/bin/python -m pytest tests/functional/syntax/ -v  # pas de régressions
```

---

## Risques

- **Régression de priorité d'erreur** : actuellement, `_visit_nodes_looping` donne la priorité aux erreurs bloquantes (`InvalidLiteral`, `InvalidType`). En collectant tout, on pourrait perdre cette priorité.
- **Erreurs en cascade** : une première erreur (ex: type inconnu) peut causer des erreurs en avalanche dans le reste du code. Il faudra peut-être filtrer les doublons ou les cascades.
- **Performance** : collecter toutes les erreurs au lieu de s'arrêter tôt peut ralentir la compilation de fichiers très invalides.

## Prochaine étape

Créer la branche `feat/multi-semantic-error` depuis `fix/multi-error-collector`, puis implémenter tâche par tâche.
