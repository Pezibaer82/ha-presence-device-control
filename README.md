# Präsenz-basierte Gerätesteuerung

**Version:** 2026.01.09e  
**Autor:** Pezibaer82  
**Typ:** Home Assistant Blueprint

---

## 📋 Beschreibung

Steuert Geräte basierend auf Bewegungs- und Präsenzmeldern mit erweiterten Funktionen für Tür-Kontrolle, Tag/Nacht-Unterscheidung und flexible Sensor-Auswahl.

### Hauptfunktionen:
- ✅ **Anwesenheitserkennung:** Mindestens ein Melder muss ON sein
- ✅ **Abwesenheitserkennung:** ALLE Melder müssen OFF sein
- ✅ **Flexible Sensor-Auswahl:** Manuell oder über Templates (Labels/Bereiche)
- ✅ **Tag/Nacht-Unterscheidung:** Unterschiedliche Actions für Tag und Nacht
- ✅ **Automatische Überwachung:** Schaltet versehentlich eingeschaltete Geräte bei Abwesenheit aus
- ✅ **Tür-Integration:** Erweiterte Steuerung mit Türkontakten
- ✅ **Automation-Aktivierung:** Optionale Ein-/Ausschaltung über Switches

---

## 🎯 Anwendungsfälle

### Beispiel 1: Badezimmer-Licht
- Bewegungsmelder erkennt Anwesenheit → Licht EIN
- Keine Bewegung für 5 Minuten → Licht AUS
- Tür schließt → Doppelte Bewegungsprüfung (Sicherheit)

### Beispiel 2: Küche mit Tag/Nacht
- **Tag:** Bewegung → Licht 100%
- **Nacht:** Bewegung → Licht 20% (gedimmt)
- Unterschiedliche Actions möglich

### Beispiel 3: Büro mit Türkontakt
- Tür öffnet + Licht AUS → Sofort EIN
- Tür geschlossen = Freeze-Modus (keine Bewegungsmelder-Trigger)

### Beispiel 4: Licht-Überwachung bei Abwesenheit
- Person verlässt Raum → Abwesenheit erkannt
- 2 Minuten später: Licht wird versehentlich per App eingeschaltet
- Automation erkennt dies → Doppelte Bewegungsprüfung
- Keine Bewegung → Licht wird automatisch ausgeschaltet

---

## ⚙️ Konfiguration

### 1. Melder auswählen

**Option A: Manuell**
```yaml
manual_sensors:
  - binary_sensor.bewegungsmelder_bad
  - binary_sensor.bewegungsmelder_flur
```

**Option B: Template (Labels/Bereiche)**
```yaml
filtered_sensors_template_on: >
  {{ expand(label_entities('bewegungsmelder_bad'))
     | selectattr('state', 'eq', 'on')
     | list | count > 0 }}
```

### 2. Verzögerungen einstellen

**Sensorverzögerung (sensor_delay):** 🆕
- **Wichtig bei Türkontakten!**
- Zeit die dein Bewegungsmelder braucht um Abwesenheit zu erkennen (Hardware-Verzögerung)
- Messanleitung: Verlasse den Raum mehrmals und miss die Zeit bis der Sensor auf OFF schaltet
- Nutze die längste gemessene Zeit
- Standard: 15 Sekunden
- **Zweck:** Verhindert zu frühes Ausschalten bei geschlossener Tür

**Einschaltverzögerung (delay_turn_on):**
- Verzögerung nach erkannter Anwesenheit
- Standard: 0 Sekunden

**Ausschaltverzögerung (delay_turn_off):**
- Verzögerung nach erkannter Abwesenheit
- **Gesamtzeit bis Ausschalten:** `sensor_delay + delay_turn_off`
- Standard: 15 Sekunden
- Beispiel: sensor_delay=15s + delay_turn_off=15s = 30s Gesamtverzögerung

### 3. Tag/Nacht-Unterscheidung (optional)
Wähle einen Switch/Binary Sensor:
- **ON/on/Active** = Tag
- **OFF/off/Inactive** = Nacht
- **sun.sun** (default) = Keine Unterscheidung

### 4. Actions definieren
Für jeden Modus separate Actions:
- **action_turn_on_day** - Einschalten bei Tag
- **action_turn_on_night** - Einschalten bei Nacht
- **action_turn_on_always** - Immer einschalten
- **action_turn_off_day** - Ausschalten bei Tag
- **action_turn_off_night** - Ausschalten bei Nacht
- **action_turn_off_always** - Immer ausschalten

### 5. Zu steuernde Entity (optional aber empfohlen)
```yaml
controlled_entity: light.badezimmer
```

**Wichtig:** Mit dieser Angabe werden **zwei wichtige Funktionen** aktiviert:

1. **Automatische Überwachung bei Abwesenheit:**
   - Wenn das Licht bei Abwesenheit versehentlich eingeschaltet wird (z.B. per App, Schalter, andere Automation)
   - Wird automatisch eine doppelte Bewegungsprüfung gestartet
   - Ohne Bewegung → Licht wird ausgeschaltet

2. **Tür-Trigger Steuerung:**
   - Ermöglicht die erweiterte Steuerung über Türkontakte
   - Tür öffnet + Licht AUS → Sofort EIN
   - Tür schließt → Doppelte Bewegungsprüfung

**Ohne diese Angabe:** 
- Funktionieren nur die Bewegungsmelder-Trigger
- Versehentlich eingeschaltete Geräte bleiben an
- Tür-Trigger sind deaktiviert

---

## 🚪 Tür-Integration

### Funktion 1: Bewegungsmelder-Bedingung
Bewegungsmelder triggern **nur** wenn mindestens **EINE Tür geöffnet** ist.

### Funktion 2: Tür-Trigger
Benötigt: `controlled_entity` (z.B. Licht)

**Verhalten:**
- **Tür öffnet + Entity AUS** → Sofort EIN
- **Tür schließt** → Doppelte Bewegungsprüfung:
  - Wait 1 (delay_turn_off): Bewegung? → EIN
  - Wait 2 (delay_turn_off / 2): Bewegung? → EIN, sonst → AUS
- **Tür geschlossen** = Freeze-Modus (keine Änderungen durch Bewegungsmelder)

---

## 📊 Interne Struktur

### Trigger (11):
1. Template: Anwesenheit erkannt
2. Template: Abwesenheit erkannt
3. Melder: Anwesenheit erkannt
4. Melder: Abwesenheit erkannt
5. Tag/Nacht: Wechsel zu Tag
6. Tag/Nacht: Wechsel zu Nacht
7. Automation: Ein-/Ausschalten
8. Tür: Geöffnet
9. Tür: Geschlossen
10. **Gesteuerte Entity: Eingeschaltet** ← NEU
11. Home Assistant: Neustart

### Hauptfälle (4):
1. **Tag/Nacht Wechsel** (3 Subfälle)
   - Nacht→Tag + Abwesenheit
   - Nacht→Tag + Anwesenheit
   - Tag→Nacht + Abwesenheit
   - Tag→Nacht + Anwesenheit

2. **Automation Aktivierung / Abwesenheit** (2 Subfälle)
   - Tag & Abwesenheit
   - Nacht & Abwesenheit

3. **Tür öffnet / Anwesenheit** (3 Subfälle)
   - Tag & Einschalten
   - Nacht & Einschalten
   - Kein Tag/Nacht

4. **Tür schließt** (11 Subfälle)
   - Wait 1: Bewegung erkannt (3x Tag/Nacht/Default)
   - Wait 2: Bewegung erkannt (3x Tag/Nacht/Default)
   - Wait 2: Timeout → Ausschalten (3x Tag/Nacht/Default)
   - Default: Entity bereits OFF

### Default:
- **Dry-Run Benachrichtigung** bei unerwarteten Triggern

---

## 🐛 Bekannte Besonderheiten

### 1. wait.trigger Prüfung
**Korrekt:** `{{ wait.trigger != none }}`  
**Falsch:** `{{ wait.trigger is defined }}`

Nach `wait_for_trigger` ist die Variable immer `defined`, aber der Wert ist `None` wenn kein Trigger erfolgte!

### 2. Template Syntax
**Korrekt:**
```jinja
{{ condition and (result1 if test else result2) }}
```

**Falsch:**
```jinja
{{ condition and ({% if test %} {{ result1 }} {% else %} {{ result2 }} {% endif %}) }}
```

Man kann keine `{% if %}` Blöcke INNERHALB von `{{ }}` verwenden!

### 3. Trigger Aliases
Alle Trigger haben Aliases um Frontend-Fehler zu vermeiden:
```
Error: Cannot read properties of undefined (reading 'includes')
```

---

## 📝 Changelog

### Version 2026.01.09c
- ✅ **NEU:** sensor_delay Input für präzise Türkontakt-Steuerung
- ✅ Verbesserte Abwesenheitserkennung bei geschlossener Tür
- ✅ delay_turn_off Beschreibung aktualisiert (Gesamtverzögerung erklärt)
- ✅ Wichtig: Verhindert zu frühes Ausschalten durch Sensor-Hardware-Verzögerung

### Version 2026.01.09b
- ✅ **NEU:** Automatische Überwachung versehentlich eingeschalteter Geräte
- ✅ **NEU:** Trigger für `controlled_entity` eingeschaltet (Template-basiert)
- ✅ Umbenennung: `door_controlled_entity` → `controlled_entity`
- ✅ Beschreibungen aktualisiert und klarer formuliert
- ✅ "(Experimental)" von Türkontakten entfernt
- ✅ Fall 4 erweitert um `controlled_entity_turned_on` Trigger

### Version 2026.01.08c
- ✅ is_state(controlled_entity, 'on') Fix für Fall 4
- ✅ YAML Syntax korrekt

### Version 2026.01.08
- ✅ wait.trigger != none Fix (kritischer Bug!)
- ✅ Alle Trigger mit Aliases (Frontend-Error Fix)
- ✅ Template Syntax korrigiert
- ✅ Choose-Strukturen optimiert
- ✅ Fall 4.11 korrekt benannt
- ✅ Dry-Run als Default
- ✅ Getestet und funktionsfähig

### Entwicklungsverlauf:
- 2026.01.07: Initiale Entwicklung
- 2026.01.08a-j: Bug-Fixes und Optimierungen
- 2026.01.08c: is_state Fix
- 2026.01.09a-b: controlled_entity Feature & Beschreibungen
- 2026.01.09c: sensor_delay Feature für Türkontakte
- 2026.01.09c: Production Release

---

## 🚀 Installation

1. Blueprint in Home Assistant importieren
2. Neue Automation erstellen
3. Blueprint auswählen: "Präsenz-basierte Gerätesteuerung"
4. Parameter konfigurieren
5. Actions definieren
6. Speichern und testen

---

## 📖 Weitere Informationen

### Template-Beispiele für Sensor-Auswahl:

Alle Templates geben `TRUE` zurück wenn mindestens ein Sensor ON ist.  
**Für Abwesenheit:** Am Ende `| count > 0` durch `| count == 0` ersetzen.

#### 1. Entities aus Bereich
```yaml
# Einzelner Bereich:
{{ area_entities('Badezimmer') | expand 
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}

# Mehrere Bereiche:
{{ (area_entities('Badezimmer') + area_entities('Flur')) | unique | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}
```

#### 2. Entities mit Labels
```yaml
# Einzelnes Label:
{{ label_entities('bewegungsmelder_bad') | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}

# Mehrere Labels (Entity muss ALLE besitzen):
{% set ns = namespace(entities=[]) %}
{% set ns.entities = label_entities('bewegungsmelder') | list %}
{% set ns.entities = ns.entities | select('in', label_entities('badezimmer')) | list %}
{{ ns.entities | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}
```

#### 3. Labels mit Ausschluss
```yaml
# Label 1 UND Label 2 ABER NICHT Label 3:
{% set ns = namespace(entities=[]) %}
{% set ns.entities = label_entities('bewegungsmelder') | list %}
{% set ns.entities = ns.entities | select('in', label_entities('badezimmer')) | list %}
{% set ns.entities = ns.entities | reject('in', label_entities('ignorieren')) | list %}
{{ ns.entities | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}
```

#### 4. Bereich mit Labels kombiniert
```yaml
# Bereich "Badezimmer" MIT Label "aktiv":
{% set ns = namespace(entities=[]) %}
{% set ns.entities = area_entities('Badezimmer') | list %}
{% set ns.entities = ns.entities | select('in', label_entities('aktiv')) | list %}
{{ ns.entities | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}
```

#### 5. Komplexes Beispiel
```yaml
# Bereiche "Badezimmer" UND "Flur" MIT Label "aktiv" UND "priorität" OHNE "nacht":
{% set ns = namespace(entities=[]) %}
{% set ns.entities = (area_entities('Badezimmer') + area_entities('Flur')) | unique | list %}
{% set ns.entities = ns.entities | select('in', label_entities('aktiv')) | list %}
{% set ns.entities = ns.entities | select('in', label_entities('priorität')) | list %}
{% set ns.entities = ns.entities | reject('in', label_entities('nacht')) | list %}
{{ ns.entities | expand
   | selectattr('attributes.device_class', 'defined')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy', 'presence'])
   | map(attribute='state') | select('eq', 'on')
   | list | count > 0 }}
```

### Wichtige Hinweise zu Templates:

**Für Abwesenheitserkennung:**
Am Ende jedes Templates `| count > 0` durch `| count == 0` ersetzen.

**Anpassungen:**
- Bereichsnamen durch eigene Namen ersetzen (z.B. "Badezimmer", "Küche")
- Label-Namen durch eigene Labels ersetzen (z.B. "bewegungsmelder_bad")

**Device Classes:**
Die Templates filtern automatisch nach:
- `motion` (Bewegungsmelder)
- `occupancy` (Belegungsmelder)
- `presence` (Präsenzmelder)

**Nach Änderungen an Labels:**
Home Assistant neu starten oder `homeassistant.reload_config_entry` ausführen!

---

### Template-Beispiele für Label-Filter (alt):
```yaml
# Anwesenheit (mindestens ein Sensor ON):
{{ expand(label_entities('bad_bewegung'))
   | selectattr('state', 'eq', 'on')
   | list | count > 0 }}

# Abwesenheit (alle Sensoren OFF):
{{ expand(label_entities('bad_bewegung'))
   | selectattr('state', 'eq', 'on')
   | list | count == 0 }}

# Mit Bereichen:
{{ expand(area_entities('Badezimmer'))
   | selectattr('domain', 'eq', 'binary_sensor')
   | selectattr('attributes.device_class', 'in', ['motion', 'occupancy'])
   | selectattr('state', 'eq', 'on')
   | list | count > 0 }}
```

---

## 💬 Support

- **GitHub:** https://github.com/Pezibaer82
- **Home Assistant Community:** Forum Thread (Link TBD)
- **Issues:** GitHub Issues

---

## 📄 Lizenz

MIT License - Frei verwendbar für private und kommerzielle Zwecke.

---

**Erstellt mit ❤️ von Pezibaer82**
