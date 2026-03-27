# Spec: KptnCook → Mealie v3.14.0 Export

## Ziel

Die `sync-with-mealie`-Funktion soll zuverlässig gegen eine lokale Mealie-Instanz v3.14.0 funktionieren.

---

## Betroffene Dateien

| Datei | Änderungsumfang |
|---|---|
| `src/kptncook/mealie.py` | Hauptarbeit: Pydantic-v2-Migration + Mealie-v3-API |
| `src/kptncook/config.py` | Evtl. neues Feld `mealie_household_id` |
| `tests/mealie_test.py` | Fixtures/Mocks an v3-Responses anpassen |
| `tests/to_mealie_test.py` | Prüfen nach Refactoring |
| `CHANGELOG.md` | Eintrag für Mealie-v3-Kompatibilität |

---

## Bekannte Bugs in `mealie.py` (unabhängig von v3)

### 1. `logged_in`-Property falsch

```python
# mealie.py:175 — FALSCH
def logged_in(self):
    return "access_token" in self.headers

# login() setzt aber:
self.headers = {"authorization": f"Bearer {access_token}"}
```

**Fix:** `"access_token"` → `"authorization"`

---

### 2. Pydantic v1-Compat-Aufrufe

| Zeile | Aktuell (v1) | Korrekt (v2) |
|---|---|---|
| 11 | `from pydantic import ... parse_obj_as` | `from pydantic import TypeAdapter` |
| 290 | `recipe.dict()` | `recipe.model_dump()` |
| 312 | `item.json()` | `item.model_dump_json()` |
| 317 | `parse_obj_as(set[T], data)` | `TypeAdapter(set[T]).validate_python(data)` |
| 361 | `recipe.json()` | `recipe.model_dump_json()` |

---

### 3. Fehlender `Content-Type`-Header bei JSON-Requests

`_post_recipe_trunk_and_get_slug` (Z. 269) und `_create_item` (Z. 312) senden `data=json.dumps(...)` bzw. `data=item.json()` ohne `Content-Type: application/json`. httpx encodiert das dann als Form-Data.

**Fix:** `content=` statt `data=` verwenden (httpx sendet dann raw bytes), oder `headers={"Content-Type": "application/json"}` ergänzen.

---

## Mealie v3.14.0 — Zu prüfende API-Änderungen

### Auth
- Endpunkt `/auth/token` (POST, form-data) — in v3 unverändert, aber verifizieren
- Token-Response: `{"access_token": "..."}` — prüfen ob noch so

### Rezept-Erstellung
- `POST /recipes` gibt in v1/v2 den Slug als plain string zurück
- In v3 möglicherweise JSON-Objekt `{"slug": "..."}` — `_post_recipe_trunk_and_get_slug` anpassen falls nötig

### Paginierung
- Aktuelle Annahme: Response enthält `total_pages` (Z. 400)
- Mealie v3 könnte das in `pageCount` oder `page` umbenannt haben — gegen Live-API verifizieren

### Households (neu in v3)
- Mealie v3 führt "Households" als Konzept unterhalb von Groups ein
- `_update_user_and_group_id` holt `userId` und `groupId` — evtl. kommt `householdId` hinzu
- `RecipeSummary`-Modell ggf. um `household_id: UUID4 | None` ergänzen
- Neues Config-Feld `MEALIE_HOUSEHOLD_ID` könnte nötig werden

### Rezept-Update
- `PUT /recipes/{slug}` — Payload-Schema prüfen, ob alle aktuellen Felder noch akzeptiert werden
- Besonders: `extras`-Feld, `settings`-Feld

### Asset-Upload
- `POST /recipes/{slug}/assets` — Response-Feld `fileName` prüfen (wird in `enrich_recipe_with_step_images` verwendet)

---

## Testplan

1. Mealie v3.14.0 lokal via Docker starten
2. Manuell einen API-Token erstellen
3. `kptncook sync-with-mealie` ausführen und Response-Fehler loggen
4. Jeden Breaking Change einzeln fixen und Test-Suite danach laufen lassen
5. End-to-End: mind. 1 Rezept vollständig in Mealie v3 landen (mit Bild + Steps + Zutaten)

---

## Nicht in Scope

- Update bestehender Mealie-Rezepte (nur Neuanlage, wie bisher)
- Migration bestehender Mealie-Daten auf v3
- Andere Exporter (Paprika, Tandoor)
