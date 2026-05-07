# 🚀 Setup & Deploy Guide — GCP PCA Notes on GitHub Pages

## Step 1 — Install MkDocs + Material Theme

```bash
pip install mkdocs mkdocs-material
```

---

## Step 2 — Set Up Your Folder Structure

```
your-repo/
├── mkdocs.yml          ← config file (provided)
└── docs/
    ├── index.md        ← master index (provided)
    ├── 01-cloud-fundamentals-core-infrastructure.md
    ├── 02-essential-core-services.md
    ├── 03-elastic-scaling-automation.md
    ├── 04-network-fundamentals.md
    ├── 05-network-architecture.md
    ├── 06-hybrid-multicloud-connectivity.md
    ├── 07-managing-security-gcp.md
    ├── 09-kubernetes-engine-gke.md
    ├── 10-terraform-gcp.md
    ├── 11-logging-monitoring-gcp.md
    └── 13-reliable-infrastructure.md
```

**Actions:**
1. Create a folder (e.g. `gcp-pca-notes/`)
2. Place `mkdocs.yml` at the root
3. Create a `docs/` subfolder
4. Move all your `.md` files into `docs/`
5. Place `index.md` into `docs/`

---

## Step 3 — Update mkdocs.yml with Your Repo Details

Open `mkdocs.yml` and replace these two lines:

```yaml
site_url: https://your-username.github.io/your-repo-name/
repo_url: https://github.com/your-username/your-repo-name
repo_name: your-username/your-repo-name
```

---

## Step 4 — Preview Locally (Optional)

```bash
cd your-repo/
mkdocs serve
```

Open `http://127.0.0.1:8000` in your browser to preview before publishing.

---

## Step 5 — Create GitHub Repo

1. Go to [github.com/new](https://github.com/new)
2. Name it (e.g. `gcp-pca-notes`)
3. Set to **Public** (required for free GitHub Pages)
4. Do NOT initialise with README

```bash
cd your-repo/
git init
git add .
git commit -m "Initial commit — GCP PCA notes"
git remote add origin https://github.com/your-username/your-repo-name.git
git push -u origin main
```

---

## Step 6 — Deploy to GitHub Pages

```bash
mkdocs gh-deploy
```

That's it. This command:
- Builds the site
- Pushes it to a `gh-pages` branch automatically
- Your site is live at: `https://your-username.github.io/your-repo-name/`

---

## Step 7 — Every Time You Update Notes

```bash
# After editing any .md file:
mkdocs gh-deploy
```

Or push to GitHub and re-run deploy. No CI/CD needed unless you want it.

---

## Adding Modules 08 and 12 Later

1. Add the `.md` file to `docs/`
2. Add an entry to `mkdocs.yml` under `nav:`
3. Add a row to the table in `docs/index.md`
4. Run `mkdocs gh-deploy`
