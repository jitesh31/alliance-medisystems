# Alliance Medisystems static website

A self-contained static website for Alliance Medisystems, updated with the 2026 product catalogue and the supplied Alliance Medisystems logo.

## Run locally

Open `index.html` directly in a browser, or serve the folder with any static server:

```bash
python3 -m http.server 8000
```

Then visit `http://localhost:8000`.

## Notes

- Built with plain HTML and CSS; there is no database, backend, framework, or dynamic product data.
- `catalogue.html` contains the detailed 2026 catalogue sections, model specifications, reagent menus, PCR kits and rapid card tests.
- `Alliance_Medisystems_Product_Catalogue_2026.pdf` is included as the original downloadable catalogue reference.
- `alliance-logo-transparent.png` is the transparent Alliance MediSystems logo used in the header, footer, hero and favicon.
- `kavita-sharma.png` is the founder portrait used on the dedicated contact page.
- The enquiry form uses the catalogue email address in a `mailto:` action and can be connected to a real form endpoint before launch.
- Product availability and regulatory status should be verified before purchase or tender submission.
- The CSS imports Google Fonts when online and falls back to system fonts offline.
