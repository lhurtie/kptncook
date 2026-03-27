# Roadmap: KptnCook → Mealie v3.14.0 Export

**Ziel:** Rezepte aus der lokalen KptnCook-Datenbank zuverlässig in eine lokale Mealie-Instanz (v3.14.0) synchronisieren.

---

## Phase 1 — Bestandsaufnahme & Diagnose

**Ziel:** Verstehen, was zwischen dem aktuellen Code und Mealie v3.14.0 bricht.

- [ ] Mealie v3-Instanz lokal aufsetzen (Docker) und API-Endpunkte manuell testen
- [ ] Differenz der API-Responses zwischen der bisherigen Mealie-Version und v3.14.0 dokumentieren
- [ ] Pydantic-v1-Kompatibilitätsprobleme in `mealie.py` identifizieren (Liste liegt bereits vor, siehe `spec.md`)
- [ ] `logged_in`-Bug in `MealieApiClient` bestätigen und verstehen

**Deliverable:** Klare Liste aller Breaking Changes (API + Code).

---

## Phase 2 — Pydantic-v2-Bereinigung

**Ziel:** Alle veralteten Pydantic-v1-Aufrufe in `mealie.py` ersetzen, bevor API-Anpassungen beginnen.

- [ ] `parse_obj_as(set[T], ...)` → `TypeAdapter(set[T]).validate_python(...)` migrieren
- [ ] `recipe.dict()` → `recipe.model_dump()` migrieren
- [ ] `item.json()` / `recipe.json()` → `item.model_dump_json()` migrieren
- [ ] `logged_in`-Property: `"access_token"` → `"authorization"` korrigieren
- [ ] Tests grün halten (`just test`)

**Deliverable:** `mealie.py` ohne Pydantic-v1-Altlasten.

---

## Phase 3 — Mealie v3 API-Kompatibilität

**Ziel:** Den HTTP-Client an die geänderte API von Mealie v3.14.0 anpassen.

- [ ] Auth-Endpunkt prüfen: `/auth/token` noch gültig? Ggf. auf `/auth/token` (Bearer) anpassen
- [ ] Rezept-Erstellung: POST `/recipes` — Response-Format prüfen (gibt v3 weiterhin den Slug zurück?)
- [ ] Paginierung: `total_pages` im Response — in v3 ggf. umbenannt zu `pageCount` o.ä.
- [ ] `household_id` vs. `group_id`: v3 führt Households ein; `_update_user_and_group_id` anpassen
- [ ] Asset-Upload: `/recipes/{slug}/assets` — Endpunkt und Response-Format prüfen
- [ ] `Content-Type: application/json` bei JSON-Payloads explizit setzen (aktuell fehlt das Header)
- [ ] Mealie-Modelle (`RecipeSummary`, `Recipe`) gegen v3-Schema validieren

**Deliverable:** `mealie.py` funktioniert gegen lokale Mealie v3.14.0-Instanz.

---

## Phase 4 — Tests & Qualitätssicherung

**Ziel:** Sicherstellen, dass die Änderungen stabil sind und keine Regression eingeführt wird.

- [ ] `tests/mealie_test.py` an neue API-Responses anpassen
- [ ] `tests/to_mealie_test.py` auf Korrektheit nach Refactoring prüfen
- [ ] Fixtures aktualisieren, falls Mealie v3 andere Felder liefert
- [ ] `just lint && just typecheck && just test` — alle grün

**Deliverable:** Vollständige Testsuite, alle Gates grün.

---

## Phase 5 — Dokumentation & Release

- [ ] `CHANGELOG.md` mit Breaking Change und Mealie-v3-Kompatibilitätsvermerk aktualisieren
- [ ] `README.md` Mealie-Setup-Abschnitt für v3 aktualisieren (neue Mealie-Version im Docker-Compose-Beispiel)
- [ ] Optional: `MEALIE_HOUSEHOLD_ID` als neues Konfigurations-Feld ergänzen, falls nötig
- [ ] Version bumpen und Release erstellen

---

## Abhängigkeiten & Risiken

| Risiko | Wahrscheinlichkeit | Maßnahme |
|---|---|---|
| Mealie v3 hat `/auth/token` geändert | Mittel | Manuell testen, ggf. neuen Endpunkt suchen |
| `total_pages` umbenannt in v3 | Mittel | Response-Schema diff |
| `household_id` ist Pflichtfeld in v3 | Hoch | `config.py` + `mealie.py` erweitern |
| Step-Images funktionieren nicht mehr | Niedrig | Asset-Endpunkt manuell verifizieren |
| Pydantic TypeAdapter bricht weitere Stellen | Niedrig | `just typecheck` fängt das auf |
