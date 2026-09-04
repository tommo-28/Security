# Passwortwiederherstellung

> Auf Mac Rockyou.txt finden: find ~ -name "rockyou.txt", Hashcat rules finden: find / -name "best64.rule" 2>/dev/null

## Immer zuerst (3 Schritte)
1. **Was ist es?** 
   
   → Dateityp: `file` yourFile.encrypted (-> z.b. openssl) / `xxd` yourFile.encrypted`|head -n 20` / `binwalk -E` yourFile.encrypted
   
   → Hashtyp raten: `hashid -m` "Hash" (Ausgabe z.b.: SHA-1 Hashcat Mode: 100) / `hashid -m` hash.txt / `hashid -mj` hash.txt (-j zeigt an ob der hash ein Salt verwendet)
2. **Hash rausholen** → `*2john`-Tool (pdf2john, 7z2john, keepass2john, iwork2john, office2john…) 

    e.g. pdf2john document.pdf > hash.txt / 7z2john archive.7z > hash.txt

   → für Hashcat oft Dateinamen-Präfix/Suffix (von John hinzugefüt) mit `sed` abschneiden: 
    - alles vor dem ersten $ abschneiden (funktioniert fast immer): `sed 's/^[^:]*://' hash_raw.txt > hash_hashcat.txt`
    - explizit den Namen der Datei löschen: `sed 's/archive.7z://' hash_raw.txt > hash_hashcat.txt`, `sed -i 's/^prefix_here//; s/suffix_here$//' hashes.txt`
    - Suffixe am Ende entfernen (falls vorhanden): `sed 's/:[^:]*$//' hash_raw.txt > hash_hashcat.txt`

3. **Kandidaten testen** → Hashcat / John / eigenes Skript
   1. Hashcat
       - Standard Wörterbuch-Angriff: `hashcat -m [MODE] -a 0 hash_hashcat.txt /usr/share/wordlists/rockyou.txt`
       - Wörterbuch + Maske: `hashcat -m [MODE] -a 0 hash_hashcat.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule`
       - Brute-Force (8-stellige Passwörter aus Kleinbuchstaben und Zahlen): `hashcat -m [MODE] -a 3 hash_hashcat.txt ?l?l?l?l?l?l?l?d`
   2. John the Ripper (kann die Datei nach den 2john Tools direkt lesen, kein sed nötig)
       - Standard Wörterbuch: `john --wordlist=/usr/share/wordlists/rockyou.txt hash_raw.txt`
       - Geknacktes Passwort anzeigen: `john --show hash_raw.txt`

**Grundregel:** erst *Suchraum kleinmachen* mit Vorwissen, dann testen. Nicht blind bruteforcen, da es sonst zu viel Zeit benötigt.

<br>

## Modi
### Hashcat 

z.b. hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

Hashcat `-m` finden, wie oben beschrieben, mit `hashid -m`.

| Parameter | Modus / Wert | Name | Wann verwendest du diesen Modus? |
| :--- | :--- | :--- | :--- |
| **`-m`** (Hash-Typ) | **`0`** | MD5 | Für sehr alte Linux-Systeme, Legacy-Datenbanken oder schnelle Datei-Prüfsummen. |
| | **`10`** | md5($hash.$salt) | Wenn der Server zuerst das Passwort nimmt und das Salz hinten anhängt. |
| | **`20`** | md5($salt.$hash) | Wenn das Salz vor dem Passwort steht (sehr beliebt bei Web-Anwendungen). |
| | **`100`** | SHA-1 | Für ältere Git-Repositories, ältere Passwörter (z. B. MySQL 3/4) oder veraltete Zertifikate. |
| | **`1400`** | SHA-256 | Der moderne Standard für Standalone-Hashes, Bitcoins, TLS/SSL und viele Datei-Signaturen. |
| | **`1800`** | sha512crypt | **Der Klassiker für Uni-Prüfungen:** Standard für moderne Linux `/etc/shadow`-Dateien. |
| | **`3200`** | bcrypt | Für moderne Web-Anwendungen (z. B. PHP/Node.js). Extrem langsam (schwer zu cracken). |
| **`-a`** (Angriff) | **`0`** | Straight (Wordlist) | Wenn du eine Passwortliste (z. B. `rockyou.txt`) hast und diese 1:1 durchtesten willst. |
| | **`3`** | Mask (Brute-Force) | Wenn du keine Wortliste hast, aber das Muster kennst (z. B. „Immer 6 Kleinbuchstaben“ -> `?l?l?l?l?l?l`). |

---

<br>

### John the Ripper

- Single Crack Mode: `john --single shadow.txt`
- Wordlist Mode: `john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`
- Wordlist + Rules Mode: `john --wordlist=/usr/share/wordlists/rockyou.txt --rules hash.txt`
- Incremental Mode: `john --incremental hash.txt`
- Rules Mode: `john --wordlist=/usr/share/wordlists/rockyou.txt --rules hash.txt`

| Modus / Argument | Name | Wann verwendest du diesen Modus? |
| :--- | :--- | :--- |
| **`--single`** | Single Crack Mode | **Immer als allererstes bei Linux-Usern!** Nutzt Kontoinformationen (Username, Home-Verzeichnis) und wandelt sie ab. *(Knackt extrem schnell schwache Passwörter wie `root123`)*. |
| **`--wordlist=[Pfad]`** | Wordlist Mode | Wenn du eine gezielte Wörterbuch-Attacke mit einer Datei fahren willst (äquivalent zu Hashcat `-a 0`). |
| **`--incremental`** | Incremental Mode | Für einen intelligenten Brute-Force-Angriff. John testet Zeichenkombinationen basierend auf sprachlichen Wahrscheinlichkeiten (CPU-optimiert). Wie Hashcat `-a 3`, nur dass John kombinationen die oft in einer Sprache vorkommen priorisiert testet. |
| **`--rules`** | Rules Mode | Wenn du Wörter aus einer Wortliste nach bestimmten Mustern verändern willst (z. B. "Hänge an jedes Wort ein Ausrufezeichen an"). |

---

<br>

## Achtung Skript!
- Salt + Passwort + Salt -> skript, hashcat kann nur salt davor oder salt danach, nicht doppelt
- verschachtelte Hashes wie SHA256(MD5())
- dynamisches Salt Hash = SHA256(Passwort + Username) -> salt wird jede zeile neu berechnet, hashcat kann nur mit Salts umgehen wenn es der gleiche Salt für alle ist oder im Hash String selbst steckt (z. B. hash:salt).
- komplexes formatting wie in Hex-Zeichen umwandeln oder rückwärts gelesen und dann erst gehasht
- komplexe mathematische bedingungen wie 6-stellige Zahl muss die Quersumme von 15 haben, hashcat kann nur alle 6-Stelligen Zahlen generieren aber generiert auch all die, die eine andere Quersumme haben
- gekürzte hashes, es werden nur die ersten 8 Zeichen des berechneten Hashes gespeichert, hashcat kann nur mit der vollen länge des hashes umgehen

<br>

## Strategie wählen (Trigger → Angriff)
| Was du weißt | Strategie | Hashcat |
|---|---|---|
| kurze feste Struktur (PIN, „4 Ziffern“) | **Maske** | `-a3` |
| ein Wort, keine Ahnung | **Wörterbuch** (rockyou) | `-a0` |
| ein Wort mit Variation (Zahl dran, Leet, Capitalize) | **Wörterbuch + Regeln** | `-a0 -r best64` |
| zwei Wörter aneinander (BerlinHamburg) | **Kombinator** | `-a1` |
| Wort + Anhängsel (Zahlen/Sonderzeichen) | **Hybrid** | `-a6` (Wort+Maske) |
| aus Bausteinen zusammengesetzt | **princeprocessor** → Hashcat | |
| exotischer/nicht-krypto Hash, kein Tool | **eigenes Python-Skript** | |

**Merke:** ein Wort→Wörterbuch · zwei Wörter→Kombinator · Wort+Anhang→Hybrid · Bausteine→prince · nichts passt→Skript.

**Schnell vs. langsam:** MD5/SHA = schnell → Regeln/große Listen ok. PBKDF2/bcrypt/KeePass/*crypt = langsam → nur wenige, gezielte Kandidaten.

<br>

## Merke:
- „Ich reduziere erst den Suchraum mit Vorwissen, statt breit zu bruteforcen.“
- „Langsamer Hash → wenige, gezielte Kandidaten.“
- „Salt verhindert Rainbow-Tables; muss beim Nachrechnen mit rein.“ ??
- „Erfolg prüfe ich über Hash-Match / geöffneten Container / plausiblen Klartext.“ ??
- „Kein Standardtool → kurzes Eigen-Skript ist schneller als lange Tool-Suche.“ ?? (ist dies erwünscht/erlaubt generieren zu lassen)??
- Wenn's nicht klappt: **begründeter Weg reicht** — sagen, welches Vorwissen fehlt + was du als Nächstes probieren würdest.

---
