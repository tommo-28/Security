# Passwortwiederherstellung

> **Mac Sachen finden**
> 
> Rockyou.txt: find ~ -name "rockyou.txt"
> 
> Hashcat rules: find / -name "best64.rule" 2>/dev/null
>
> 2john Befehle: find / -name "pdf2john*" 2>/dev/null, eventuell mit python aufrufen: python3 /pfad/zu/john/run/pdf2john.py datei.pdf > hash.txt
>
> Mac nutzt BSD sed während Kali GNU sed nutzt. -> -i geht nicht -> In neue Datei leiten: sed 's/^[^:]*://' hash_raw.txt > hash_sauber.txt
>
> oder wenn Prefix und Suffix zu entfernen sind: sed 's/^prefix_hier// ; s/suffix_hier$//' hash_raw.txt > hash_sauber.txt oder sed -e 's/^prefix_hier//' -e 's/suffix_hier$//' hash_raw.txt > hash_sauber.txt
>
> ohne den genauen dateinahmen anzugeben: sed -e 's/^[^:]*://' -e 's/:.*$//' hash_raw.txt > hash_clean.txt

## Immer zuerst (3 Schritte)
1. **Was ist es?** 
   
   → Dateityp: `file` yourFile.encrypted (-> z.b. openssl) / `xxd` yourFile.encrypted`|head -n 20` / `binwalk -E` yourFile.encrypted
   
   → Hashtyp raten: `hashid -m` "Hash" (Ausgabe z.b.: SHA-1 Hashcat Mode: 100) / `hashid -m` hash.txt / `hashid -mj` hash.txt (-j zeigt an ob der hash ein Salt verwendet)
2. **Hash rausholen** → `*2john`-Tool (pdf2john, 7z2john, keepass2john, iwork2john, office2john…) 

    e.g. pdf2john document.pdf > hash.txt / 7z2john archive.7z > hash.txt (libreoffice2john nicht odt2john!!)

   → für Hashcat oft Dateinamen-Präfix/Suffix (von John hinzugefüt) mit `sed` abschneiden: 
    - alles vor dem ersten $ abschneiden (funktioniert fast immer): `sed 's/^[^:]*://' hash_raw.txt > hash_hashcat.txt`
    - explizit den Namen der Datei löschen: `sed 's/archive.7z://' hash_raw.txt > hash_hashcat.txt`, `sed -i 's/^prefix_here//; s/suffix_here$//' hashes.txt`
    - Suffixe am Ende entfernen (falls vorhanden): `sed 's/:[^:]*$//' hash_raw.txt > hash_hashcat.txt`

3. **Kandidaten testen** → Hashcat / John / eigenes Skript
   1. Hashcat
       - Standard Wörterbuch-Angriff: `hashcat -m [MODE] -a 0 hash_hashcat.txt /usr/share/wordlists/rockyou.txt`
       - Wörterbuch + Maske: `hashcat -m [MODE] -a 0 hash_hashcat.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule`
       - Brute-Force (8-stellige Passwörter aus Kleinbuchstaben und Zahlen): `hashcat -m [MODE] -a 3 hash_hashcat.txt ?l?l?l?l?l?l?l?d`
       - Fertig? `hashcat -m [mode] hash_clean.txt --show`
   2. John the Ripper (kann die Datei nach den 2john Tools direkt lesen, kein sed nötig)
       - Standard Wörterbuch: `john --wordlist=/usr/share/wordlists/rockyou.txt hash_raw.txt`
       - Geknacktes Passwort anzeigen: `john --show hash_raw.txt`

**Grundregel:** erst *Suchraum kleinmachen* mit Vorwissen, dann testen. Nicht blind bruteforcen, da es sonst zu viel Zeit benötigt.

<br>

## Modi
### Hashcat 

z.b. hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

Hashcat `-m` finden, wie oben beschrieben, mit `hashid -m` / [hier nachsehen](https://hashcat.net/wiki/doku.php?id=hashcat)

| Parameter | Modus / Wert | Name | Wann verwendest du diesen Modus? |
| :--- | :--- | :--- | :--- |
| **`-m`** (Hash-Typ) | **`0`** | MD5 | Für sehr alte Linux-Systeme, Legacy-Datenbanken oder schnelle Datei-Prüfsummen. |
| | **`10`** | md5($hash.$salt) | Wenn der Server zuerst das Passwort nimmt und das Salz hinten anhängt. |
| | **`20`** | md5($salt.$hash) | Wenn das Salz vor dem Passwort steht (sehr beliebt bei Web-Anwendungen). |
| | **`100`** | SHA-1 | Für ältere Git-Repositories, ältere Passwörter (z. B. MySQL 3/4) oder veraltete Zertifikate. |
| | **`1400`** | SHA-256 | Der moderne Standard für Standalone-Hashes, Bitcoins, TLS/SSL und viele Datei-Signaturen. |
| | **`1800`** | sha512crypt | **Der Klassiker für Uni-Prüfungen:** Standard für moderne Linux `/etc/shadow`-Dateien. |
| | **`3200`** | bcrypt | Für moderne Web-Anwendungen (z. B. PHP/Node.js). Extrem langsam (schwer zu cracken). |
| | **`18600`** | .odt openDocument Dateien | In seltenen fällen eventuell auch die ältere Version 18600 |
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
| **`--format`** | Format Mode | Wenn mann weiß dass format=raw-md5 ist, um die wordlist zu verkleinern. |

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

Nutze primär John the Ripper bei:
- LibreOffice / OpenDocument (.odt, .ods, .odp): Das Ausgabeformat von libreoffice2john macht bei Hashcat fast immer Token-Längen-Probleme.
- SSH-Keys (id_rsa / ssh2john): Funktioniert mit John direkt ohne Formatierung.
- GPG / PGP Keys (gpg2john): John liest die Extrakte nativ.
- Java Keystores / Hashcodes (keystore2john): Werden von John reibungslos verarbeitet.
- Immer, wenn Hashcat "Token length exception" meldet: Wenn Hashcat meckert, nimm sofort hash_raw.txt und John.

Nutze primär Hashcat bei:
- Standard-Hashes: Pure Hashes wie MD5, SHA256, NTLM oder Unix-Passwörter (/etc/shadow).
- MS Office & PDF: .docx, .xlsx oder .pdf (Modi 9400–9600 und 10500 laufen stabil in Hashcat).
- KeePass & 7-Zip: .kdbx (Modus 13400) und .7z (Modus 11600).
- Masken-Angriffen (-a 3): Wenn du ein bestimmtes Muster testen musst (z. B. ?u?l?l?l?d?d?d), ist Hashcat unschlagbar schnell.

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
