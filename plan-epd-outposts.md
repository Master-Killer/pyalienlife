# Plan : rendre les outposts pyalienlife déplaçables avec Even Pickier Dollies

*Analyse produite le 2026-07-20 — branche `feature/move_all_caravans` de pyalienlife + mod `factorio-even-pickier-dollies` (EPD 3.0.x).*

## Constat principal

**Les outposts ne sont pas indéplaçables par nature — pyalienlife les blackliste explicitement.**
Dans `pyalienlife/control.lua:41-74`, la fonction `pickerdollies()` (appelée uniquement à `on_init`) enregistre via l'API remote `PickerDollies` une blacklist qui inclut les 4 types d'outposts :

- `outpost` (type `container`, 6×6)
- `outpost-fluid` (type `storage-tank`)
- `outpost-aerial` (container)
- `outpost-aerial-fluid` (storage-tank)

EPD implémente la même API remote que l'ancien PickerDollies (voir `factorio-even-pickier-dollies/API.md`), donc cette blacklist s'applique aussi à EPD. Côté moteur, `container` et `storage-tank` sont téléportables nativement depuis Factorio 2.0.21 (changelog EPD 2.4.0 : « the LuaEntity::teleport API makes all of the previous teleporting around no longer necessary ») — EPD utilisera un vrai teleport, **pas** le « transporter mode » (clone + destroy) réservé aux entités non téléportables. Conséquence clé : **le `unit_number` et la référence `LuaEntity` survivent au déplacement**.

Or les schedules de caravanes stockent des références `LuaEntity` (`sch.entity`), pas des positions figées, et `goto_entity` (`scripts/caravan/impl/control.lua:9`) passe `destination_entity` au pathfinder. Un watchdog existe déjà (`scripts/caravan/event-handlers/global.lua:615-623`) qui re-pathfind une caravane inactive si sa destination s'est éloignée de plus de ~31 tiles. **Déplacer physiquement un outpost est donc quasi gratuit** : les schedules suivent automatiquement. Il ne reste que du rafraîchissement cosmétique et un re-path immédiat à gérer.

À noter : cette fonctionnalité est **complémentaire** du bouton « Move all caravans » de la branche `feature/move_all_caravans` — EPD déplace le bâtiment physique de quelques tiles (schedules conservés automatiquement), tandis que le bouton ré-affecte les schedules vers un *autre* outpost existant. Les deux se combinent sans conflit.

## Ce qui devient périmé quand un outpost se téléporte

| État | Où | Impact |
|---|---|---|
| `sch.position` (snapshot de la position) | schedules dans `storage.caravans` | Utilisé seulement en fallback si `sch.entity` devient invalide, et par le watchdog. Faible impact, à rafraîchir quand même. |
| `sch.localised_name` (coordonnées figées dans le texte, ex. « Outpost (120, -45) ») | idem | Purement cosmétique mais visible dans la GUI — à rafraîchir. |
| Commande `go_to_location` en cours | pathfinder des caravanes en route | Le pathfinder vise la position calculée au départ ; le watchdog ne corrige qu'au-delà de ~31 tiles et seulement à l'arrêt. Re-path immédiat souhaitable. |
| GUI relative attachée à l'outpost (`rebuild_relative_panel`, indexée par `unit_number`) | `manager.lua` | `unit_number` inchangé par teleport → rien à faire. |

Aucune autre partie de pyalienlife ne référence les outposts en dehors des scripts caravane (vérifié par grep sur `scripts/`), et le prototype n'a pas d'entité compagnon cachée (le `create-entity` dans `outpost.lua` n'est qu'un trigger d'effet de dégâts).

## Plan d'implémentation

### Étape 1 — Retirer les 4 outposts de la blacklist

Dans `pyalienlife/control.lua`, supprimer les lignes 48-51 (`outpost`, `outpost-fluid`, `outpost-aerial`, `outpost-aerial-fluid`). Garder tout le reste blacklisté (caravanes, biofluid, beacons…).

⚠️ `pickerdollies()` n'est appelée qu'à `on_init` et la blacklist est persistée dans le storage d'EPD. Pour les **parties existantes**, il faut donc aussi appeler `remove_blacklist_name` dans une migration ou dans `on_configuration_changed` :

```lua
if remote.interfaces["PickerDollies"] then
    for _, name in pairs {"outpost", "outpost-fluid", "outpost-aerial", "outpost-aerial-fluid"} do
        remote.call("PickerDollies", "remove_blacklist_name", name)
    end
end
```

### Étape 2 — S'abonner à l'événement `dolly_moved_entity_id`

EPD fournit un événement custom (voir `API.md`) avec `moved_entity`, `start_pos`, `start_direction`, `start_unit_number`. L'abonnement doit se faire **à chaque chargement** (`on_init` ET `on_load`), car les handlers d'événements ne sont pas persistés :

```lua
-- control.lua (ou nouveau fichier scripts/caravan/event-handlers/picker-dollies.lua)
local function register_epd_event()
    if not (remote.interfaces["PickerDollies"] and remote.interfaces["PickerDollies"]["dolly_moved_entity_id"]) then return end
    script.on_event(remote.call("PickerDollies", "dolly_moved_entity_id"), Caravan.on_outpost_moved)
end
-- à appeler depuis py.on_event(py.events.on_init(), ...) ET script.on_load
```

Point d'attention : le framework `py.on_event` gère des IDs statiques ; l'ID EPD étant dynamique (`generated_event_name`), un `script.on_event` direct est plus sûr — vérifier que cela ne collisionne pas avec la mécanique de dispatch de pypostprocessing.

### Étape 3 — Handler `on_outpost_moved`

Dans les scripts caravane (proposition : `scripts/caravan/event-handlers/global.lua`, en réutilisant `assign_entity_destination` déjà factorisée par la branche) :

```lua
local outpost_names = {["outpost"] = true, ["outpost-fluid"] = true,
                       ["outpost-aerial"] = true, ["outpost-aerial-fluid"] = true}

function Caravan.on_outpost_moved(event)
    local moved = event.moved_entity
    if not (moved and moved.valid and outpost_names[moved.name]) then return end

    local affected = {}
    for unit_number, caravan_data in pairs(storage.caravans) do
        if CaravanImpl.validity_check(caravan_data) and caravan_data.schedule then
            for schedule_id, sch in pairs(caravan_data.schedule) do
                if sch.entity == moved then
                    -- rafraîchit sch.position + sch.localised_name
                    assign_entity_destination(sch, moved)
                    affected[unit_number] = true
                    -- caravane actuellement en route vers cet arrêt → re-path immédiat
                    if caravan_data.schedule_id == schedule_id and caravan_data.action_id == -1 then
                        CaravanImpl.goto_entity(caravan_data, moved)
                    end
                end
            end
        end
    end

    if next(affected) then refresh_guis(moved, affected) end
end
```

Notes de conception :

- **Coût** : itération complète de `storage.caravans` à chaque pression de touche EPD. C'est une action joueur rare et la boucle est triviale (comparaison de référence) — pas d'index nécessaire. C'est exactement le pattern déjà utilisé par `relocate_all_caravans` de la branche.
- **Caravane en pleine action à l'outpost** (`action_id > 0`) : les actions manipulent `sch.entity` directement (transferts d'inventaire) et un garde-fou de distance existe déjà (`action.lua:116`, ~31 tiles). EPD déplaçant d'1 tile par pression, on laisse faire — pas de cas spécial.
- **Interrupts** (mis à jour après le commit `0ac3adcc` « Move all caravans to outpost only, even interrupts ») :
  - `condition.entity` : simple référence `LuaEntity`, et les libellés des conditions sont calculés en direct depuis `entity.position` à la construction de la GUI (`gui/interrupt_conditions.lua:18`) → **rien à faire**, le teleport EPD est transparent.
  - `interrupt.schedule[*]` : porte des `sch.position`/`sch.localised_name` figés comme les schedules de caravanes → le handler doit aussi itérer `storage.interrupts` et `storage.edited_interrupts` et leur appliquer `assign_entity_destination(sch, moved)` quand `sch.entity == moved`, puis appeler `EditInterruptGui.update_conditions_pane`/`update_targets_pane` pour les joueurs en cours d'édition — même pattern que `relocate_all_caravans`.
  - La logique `outposts_are_interchangeable` (catégories item/fluid) ne s'applique pas ici : même entité, pas de question de compatibilité.
  - Arrêts temporaires in-flight : le commit a retiré le filtre `not sch.temporary` ; le handler ci-dessus n'en a jamais eu → alignés.

### Étape 4 — Garde-fou « transporter mode » (défensif)

Les containers/storage-tanks se téléportent, donc `start_unit_number == moved_entity.unit_number` en pratique. Par sécurité (changement futur d'EPD ou de prototype), ajouter en tête de handler :

```lua
if event.start_unit_number and event.start_unit_number ~= moved.unit_number then
    -- L'entité a été recréée : les références sch.entity sont mortes.
    -- Rattraper les entrées invalides dont la position correspond à event.start_pos.
    for _, caravan_data in pairs(storage.caravans) do
        for _, sch in pairs(caravan_data.schedule or {}) do
            if sch.entity and not sch.entity.valid
               and sch.position and py.distance_squared(sch.position, event.start_pos) < 1 then
                assign_entity_destination(sch, moved)
            end
        end
    end
end
```

### Étape 5 — Rien à faire pour la rotation

Les outposts sont carrés (6×6) et de type container/storage-tank sans direction significative → pas d'enregistrement `add_oblong_name`. (Attention quand même : EPD envoie le même événement pour une rotation ; le handler ci-dessus est idempotent, donc sans risque.)

### Étape 6 — Changelog et locale

- `changelog.txt` : « Outposts can now be moved with Even Pickier Dollies / Picker Dollies; caravan schedules follow automatically. »
- Pas de nouvelle clé de locale nécessaire (aucun message joueur ajouté), sauf si on veut un flying text de confirmation.

## Ordre de livraison suggéré

1. Étapes 1 + 2 + 3 dans un commit (fonctionnalité complète).
2. Étape 4 dans le même commit ou un second (durcissement).
3. Mettre à jour `scripts/caravan/test/test_plan.md` avec les cas ci-dessous.

## Plan de test manuel

1. Outpost avec 2 caravanes programmées → déplacer l'outpost de 5 tiles avec EPD → les entrées de schedule affichent les nouvelles coordonnées, les caravanes livrent au nouvel emplacement.
2. Caravane **en route** vers l'outpost → déplacer l'outpost pendant le trajet → la caravane re-pathfind immédiatement (sans attendre le watchdog).
3. Caravane **en train de charger** à l'outpost → déplacer l'outpost d'1 tile → l'action se termine normalement.
4. GUI relative de l'outpost ouverte pendant le déplacement → pas de crash, panneau toujours fonctionnel.
5. Outpost fluid avec du fluide stocké → déplacement → le fluide est conservé (géré par le moteur ≥ 2.0.21).
6. Partie sauvegardée avant le patch → chargement → migration retire bien la blacklist (l'outpost devient déplaçable).
7. Vérifier que caravanes, bioports, etc. restent bien **non** déplaçables.
8. Interaction avec le bouton « Move all caravans » : déplacer physiquement puis ré-affecter vers un autre outpost, et inversement.

## Choix faits (exécution autonome)

- Périmètre limité aux 4 outposts ; les entités biofluid (`bioport`, `provider-tank`, `requester-tank`) restent blacklistées — elles ont un état indexé différemment et mériteraient une analyse séparée.
- Handler placé dans les scripts caravane plutôt que dans `control.lua` pour réutiliser `assign_entity_destination`/`refresh_guis` introduits par la branche (il faudra les exporter ou déplacer le handler dans `global.lua`).
- Pas de migration vers le blacklisting mod-data d'EPD 2.7.0 (phase prototype) : pyalienlife doit rester compatible avec l'ancien PickerDollies 1.1, donc l'API runtime reste le bon choix.
