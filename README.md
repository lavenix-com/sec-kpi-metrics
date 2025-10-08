# Cybersecurity KPIs Metrics

An open-source catalog of cybersecurity KPIs.

## Quick start

```bash
npm install
npm run dev
```

Additional scripts:

- `npm run build` – type-check and build the production bundle
- `npm run lint` – apply ESLint fixes to the source directory

## Data model

All metrics live inside `src/data` so if you like to contribute, you can focus on the relevant category files.

- `src/data/index.json` – list of categories with their slug and entry counts
- `src/data/metrics/*.json` – category-specific metric collections (one JSON file per category)

Each metric object supports the following fields:

```json
{
  "Category": "Security Program",
  "SubCategory": "Maturity",
  "MetricTitle": "Cyber Hygiene Score",
  "MetricDescription": "Summary of the organization's cyber posture…",
  "ReportPeriod": "Quarterly",
  "Target": "95% or higher",
  "Comment": "Great for benchmarking divisions",
  "Contributor": "Jane Doe",
  "Source": "CISA AWARE Scorecard"
}
```

### Adding or editing metrics

1. Pick the relevant category file under `src/data/metrics/`. If the category does not yet exist, create a new JSON file using a kebab-case slug (for example `threat-hunting.json`).
2. Add or update metric objects. Keep properties in the same order and omit keys you do not need.
3. Update `src/data/index.json` with the category name, slug, and the new count. (Tip: run `npm run build` – type checking will fail if the slugs are mismatched.)
4. Submit a pull request.

> **Tip:** The UI renders metric details using whitespace-aware formatting. Use `\n\n` for clear paragraph breaks and keep values concise.

## Contributing

Contributions are welcome! Please open an issue to propose new categories or structural changes. Run `npm run build` before submitting a PR to ensure both the type checker and the production build succeed.

## License

This project is licensed under the [CC zero license](LICENSE).

---

Curated by [Lavenix](https://lavenix.com)
