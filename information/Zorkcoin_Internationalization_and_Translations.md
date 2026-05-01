---
layout: default
title: Zorkcoin™ Internationalization
nav_order: 100
lang: en
---

# Zorkcoin™ Internationalization

This site is a static Jekyll project on **GitHub Pages**. Each locale uses its own URL prefix (folder) so everything stays plain HTML—no server-side language negotiation required.

## How locales work

- **English (default)** lives at the site root, for example `/information/Zorkcoin_Internationalization_and_Translations.html`.
- **Other languages** use a folder named with the locale slug, for example `/es/information/Zorkcoin_Internationalization_and_Translations.html` for Spanish.
- The sidebar **language** control lists every locale by its **native name** and jumps to the **same relative path** under the chosen locale. If a translated file does not exist yet, GitHub Pages returns 404 until that page is added—start from the matching English path and copy the structure.

### Locale slugs (URL segments)

These match `_data/languages.yml` and `_config.yml` defaults:

| Slug | Language |
|------|----------|
| `en` | English (root, no prefix) |
| `es` | Spanish |
| `fr` | French |
| `de` | German |
| `it` | Italian |
| `pt` | Portuguese |
| `ru` | Russian |
| `zh-hans` | Chinese (Simplified) |
| `zh-hant` | Chinese (Traditional) |
| `ja` | Japanese |
| `ko` | Korean |
| `vi` | Vietnamese |
| `fil` | Filipino (Tagalog) |
| `uk` | Ukrainian |
| `cs` | Czech |
| `hi` | Hindi |
| `ur` | Urdu |
| `id` | Indonesian |
| `ar` | Arabic |
| `ms` | Malay |
| `ta` | Tamil |
| `he` | Hebrew |
| `tr` | Turkish |
| `th` | Thai |
| `fa` | Persian |
| `bn` | Bengali |

### Adding or updating a translated page

1. Copy the English Markdown file’s path, e.g. `information/Some_Page.md`.
2. Create the same path under the locale folder, e.g. `es/information/Some_Page.md`.
3. Set front matter like the English page (`layout`, `title`, `nav_order`, etc.). You may add `lang: xx` explicitly; otherwise the site infers the locale from the first path segment (`es/…`, `zh-hans/…`, etc.). `_config.yml` also lists `defaults` by folder for the same slugs (useful if your Jekyll build merges them into `page.lang`).
4. Translate the body. Keep the same filename so the language switcher resolves the correct URL.
5. Right-to-left locales (`ar`, `he`, `fa`, `ur`) use `dir="rtl"` on `<html>` from `_data/languages.yml`.

### Contribute to the translation efforts

Much of the early translations have been done manually and/or with so-called AI, so some wording will be wrong or awkward. Please open issues or pull requests with corrections.

---

## 為翻譯工作做出貢獻

早期翻譯大部分是人工和/或所謂的人工智慧完成的，所以我確信有些地方會出錯，而且
很多地方可能不太理想。請隨時提出建議，
並/或為翻譯工作提供更新。

## Внесите свой вклад в усилия по переводу

Большая часть ранних переводов была выполнена вручную и/или с помощью
так называемого ИИ, поэтому я уверен, что некоторые вещи будут неверными, а
многие — неидеальными. Не стесняйтесь вносить предложения
и/или предоставлять обновления перевода.

## Mag-ambag sa Mga Pagsisikap sa Pagsasalin

Karamihan sa mga naunang pagsasalin ay ginawa nang manu-mano at/o gamit
tinatawag na AI, kaya sigurado ako na may mga bagay na mali at
maraming bagay ang hindi magiging perpekto. Mangyaring huwag mag-atubiling gumawa ng mga mungkahi
at/o magbigay ng mga update sa pagsasalin sa mga pagsisikap.

## Contribuir a los esfuerzos de traducción

Muchas de las primeras traducciones se realizaron manualmente o con la llamada IA, por lo que estoy seguro de que habrá errores y que muchos aspectos no serán ideales. No duden en hacer sugerencias o proporcionar actualizaciones de traducción. La versión mantenida en el sitio para español está en la ruta `/es/information/`.

## Зробіть свій внесок у перекладацькі зусилля

Багато ранніх перекладів було виконано вручну та/або за допомогою так званого штучного інтелекту, тому я впевнений, що деякі речі будуть неправильними, а багато речей не ідеальними. Будь ласка, не соромтеся вносити пропозиції та/або надавати оновлення перекладу до цих зусиль.
