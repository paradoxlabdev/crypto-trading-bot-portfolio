# Instrukcja - Jak Zrobić Screenshoty z Diagramów

## Pliki Gotowe do Screenshotów

Stworzyłem 2 pliki które możesz otworzyć i zrobić screenshoty:

1. **`database-schema-diagram.md`** - Diagram schematu PostgreSQL
2. **`redis-key-structure.md`** - Struktura kluczy Redis

---

## Opcja 1: Screenshot z Markdown (Najprostsze)

### Kroki:

1. **Otwórz pliki w edytorze:**
   - Otwórz `database-schema-diagram.md` w VS Code / Cursor
   - Otwórz `redis-key-structure.md` w VS Code / Cursor

2. **Zrób screenshot:**
   - Użyj narzędzia do screenshotów (Cmd+Shift+4 na Mac, Snipping Tool na Windows)
   - Lub użyj wbudowanego narzędzia w edytorze (jeśli ma)

3. **Zapisz jako PNG:**
   - `postgresql-schema-diagram.png`
   - `redis-key-structure.png`

---

## Opcja 2: Renderowanie Markdown (Lepsza Jakość)

### Użyj narzędzi do renderowania Markdown:

1. **Online Tools:**
   - https://dillinger.io - Wklej zawartość, zrób screenshot
   - https://stackedit.io - Podobnie
   - https://markdownlivepreview.com - Podgląd na żywo

2. **VS Code Extensions:**
   - Zainstaluj "Markdown Preview Enhanced"
   - Otwórz preview (Cmd+Shift+V)
   - Zrób screenshot z preview

3. **Export do PDF/HTML:**
   - Użyj `pandoc` do konwersji:
     ```bash
     pandoc database-schema-diagram.md -o schema.pdf
     pandoc redis-key-structure.md -o redis.pdf
     ```
   - Otwórz PDF i zrób screenshot

---

## Opcja 3: Stwórz Wizualne Diagramy (Najlepsze)

### PostgreSQL Schema - Użyj narzędzi:

1. **dbdiagram.io** (Darmowe, Online):
   - Wejdź na https://dbdiagram.io
   - Stwórz nowy diagram
   - Użyj SQL z `database/migrations/001_initial_schema.sql`
   - Export jako PNG

2. **draw.io** (Darmowe, Online):
   - Wejdź na https://draw.io
   - Stwórz ERD diagram ręcznie
   - Użyj kształtów: prostokąty dla tabel, linie dla relacji
   - Export jako PNG

3. **pgAdmin / DBeaver:**
   - Połącz się z bazą danych
   - Użyj wbudowanego ERD generatora
   - Zrób screenshot

### Redis Key Structure - Użyj narzędzi:

1. **Draw.io:**
   - Stwórz diagram drzewa (tree structure)
   - Każda kategoria kluczy jako gałąź
   - Dodaj przykłady kluczy
   - Export jako PNG

2. **Mermaid (w Markdown):**
   - Możesz dodać diagramy Mermaid do pliku
   - GitHub automatycznie renderuje Mermaid
   - Zrób screenshot z GitHub preview

---

## Opcja 4: Użyj Narzędzi CLI (Dla Zaawansowanych)

### PostgreSQL Schema:

```bash
# Użyj pg_dump do wygenerowania diagramu
pg_dump -h localhost -U bot_user -d crypto_bot --schema-only > schema.sql

# Lub użyj schemaspy (generuje HTML diagramy)
java -jar schemaspy.jar -t pgsql -host localhost -db crypto_bot -u bot_user -p YOUR_PASSWORD -o output/
```

### Redis Key Structure:

```bash
# Wygeneruj listę wszystkich kluczy
redis-cli -n 14 --scan --pattern "*" > redis-keys.txt

# Zrób screenshot z pliku tekstowego
```

---

## Rekomendowane Rozwiązanie

**Najprostsze i najszybsze:**

1. Otwórz `database-schema-diagram.md` w VS Code
2. Otwórz Markdown Preview (Cmd+Shift+V)
3. Zrób screenshot całego diagramu
4. Powtórz dla `redis-key-structure.md`

**Najlepsze jakościowo:**

1. Użyj https://dbdiagram.io dla PostgreSQL schema
2. Importuj SQL z `database/migrations/001_initial_schema.sql`
3. Export jako PNG
4. Dla Redis użyj draw.io i stwórz diagram drzewa ręcznie

---

## Gdzie Umieścić Screenshoty

```
portfolio-repo/
  └── images/
      ├── database/
      │   ├── postgresql-schema-diagram.png
      │   └── redis-key-structure.png
      └── ...
```

---

## Dodaj do README.md

Po zrobieniu screenshotów, dodaj do README.md:

```markdown
## 🗄️ Database Architecture

### PostgreSQL Schema

![PostgreSQL Schema](./images/database/postgresql-schema-diagram.png)

*Complete database schema with 10 tables (5 core + 5 analytics) and 6 views*

### Redis Key Structure

![Redis Key Structure](./images/database/redis-key-structure.png)

*Redis key naming conventions and data organization (DB 14, ~5M+ keys)*
```

---

## Tips

1. **Użyj wysokiej rozdzielczości** - minimum 1920x1080
2. **Zasłoń wrażliwe dane** - jeśli są jakieś przykładowe wartości
3. **Dodaj legendę** - jeśli diagram jest skomplikowany
4. **Użyj kolorów** - dla lepszej czytelności (np. różne kolory dla różnych kategorii)

---

**Gotowe!** 🎉

Po zrobieniu screenshotów, Twoje portfolio będzie miało profesjonalne diagramy architektury danych.

