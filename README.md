# Alliance Medisystems static website

A self-contained static website inspired by the visual language of a modern in-vitro diagnostics company.

## Run locally

Open `index.html` directly in a browser, or serve the folder with any static server:

```bash
python3 -m http.server 8000
```

Then visit `http://localhost:8000`.

## Notes

- Built with plain HTML and CSS; there is no database, backend, framework, or dynamic product data.
- The enquiry form uses a placeholder `mailto:` action and should be connected to a real form endpoint before launch.
- Product names, partner names, phone number and email are dummy content for the Alliance Medisystems concept.
- The CSS imports Google Fonts when online and falls back to system fonts offline.
