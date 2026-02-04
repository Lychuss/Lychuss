# ☕ Raphael Mharcus G. C – Coffee & Code

Hi there! I’m a 2nd Year BS Computer Engineering student at Southern Luzon State University.  
I enjoy building both **frontend** and **backend** projects, and I’m always learning new technologies. 🌱

---

## 🌐 Connect with Me
[![Email](https://img.shields.io/badge/Email-D14836?logo=gmail&logoColor=white)](mailto:raphaelsanjuan6@gmail.com)

---

## 💻 Tech Stack
![Java](https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white) 
![JavaScript](https://img.shields.io/badge/javascript-%23323330.svg?style=for-the-badge&logo=javascript&logoColor=%23F7DF1E) 
![TypeScript](https://img.shields.io/badge/typescript-%23007ACC.svg?style=for-the-badge&logo=typescript&logoColor=white) 
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) 
![Next JS](https://img.shields.io/badge/Next-black?style=for-the-badge&logo=next.js&logoColor=white) 
![TailwindCSS](https://img.shields.io/badge/tailwindcss-%2338B2AC.svg?style=for-the-badge&logo=tailwind-css&logoColor=white) 
![Spring](https://img.shields.io/badge/spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white) 
![Express.js](https://img.shields.io/badge/express.js-%23404d59.svg?style=for-the-badge&logo=express&logoColor=%2361DAFB) 
![JavaFX](https://img.shields.io/badge/javafx-%23FF0000.svg?style=for-the-badge&logo=javafx&logoColor=white) 
![MySQL](https://img.shields.io/badge/mysql-4479A1.svg?style=for-the-badge&logo=mysql&logoColor=white) 
![PostgreSQL](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white)

---
# 📊 GitHub Stats

![Lychuss's GitHub Stats](https://github-readme-stats.vercel.app/api?username=Lychuss&show_icons=true&count_private=true&theme=tokyonight)

name: Update language stats

on:
  schedule:
    - cron: '0 0 * * *'   # daily at 00:00 UTC
  workflow_dispatch:     # manual trigger

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout this repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install dependencies
        run: npm install @octokit/rest@19.0.0

      - name: Run language aggregator
        env:
          GITHUB_TOKEN: ${{ secrets.TARGET_REPO_PAT || secrets.GITHUB_TOKEN }}
        run: node ./scripts/update_languages.js --username Lychuss --output-file LANGUAGES.md --include-forks false

      - name: Commit and push changes
        if: ${{ steps['Run language aggregator'].conclusion == 'success' }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add LANGUAGES.md
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Update language stats (automated)"
            git push
          fi

#!/usr/bin/env node
/**
 * Minimal script that:
 * - lists repos for a username
 * - fetches /repos/{owner}/{repo}/languages for each
 * - aggregates bytes per language
 * - writes a Markdown file with percentages
 *
 * Usage:
 *   node scripts/update_languages.js --username Lychuss --output-file LANGUAGES.md
 *
 * Requires: env GITHUB_TOKEN (if you need to read private repos or to avoid rate limits)
 */

const { Octokit } = require("@octokit/rest");
const fs = require("fs");
const path = require("path");

function parseArgs() {
  const args = process.argv.slice(2);
  const out = {};
  for (let i = 0; i < args.length; i++) {
    if (args[i].startsWith("--")) {
      const key = args[i].slice(2);
      const val = args[i+1] && !args[i+1].startsWith("--") ? args[++i] : "true";
      out[key] = val;
    }
  }
  return out;
}

(async () => {
  try {
    const opts = parseArgs();
    const username = opts.username;
    if (!username) throw new Error("--username is required");
    const outputFile = opts["output-file"] || "LANGUAGES.md";
    const includeForks = (opts["include-forks"] || "false") === "true";

    const token = process.env.GITHUB_TOKEN || "";
    const octokit = new Octokit({ auth: token || undefined });

    // Get all repos for the user (owner)
    const repos = await octokit.paginate(octokit.repos.listForUser, {
      username,
      per_page: 100,
      type: "owner", // only repos owned by the user
    });

    const filtered = repos.filter(r => includeForks ? true : !r.fork);

    // Aggregate language bytes
    const totals = {};
    for (const repo of filtered) {
      try {
        const resp = await octokit.repos.listLanguages({
          owner: repo.owner.login,
          repo: repo.name,
        });
        const langs = resp.data || {};
        for (const [lang, bytes] of Object.entries(langs)) {
          totals[lang] = (totals[lang] || 0) + bytes;
        }
      } catch (err) {
        console.error(`failed to get languages for ${repo.full_name}: ${err.message}`);
      }
    }

    // Compute percentages
    const totalBytes = Object.values(totals).reduce((a,b) => a + b, 0);
    const rows = Object.entries(totals)
      .sort((a,b) => b[1] - a[1])
      .map(([lang, bytes]) => {
        const pct = totalBytes ? (bytes / totalBytes * 100) : 0;
        return { lang, bytes, pct };
      });

    // Top language
    const top = rows[0] ? rows[0].lang : "—";
    const topPct = rows[0] ? rows[0].pct.toFixed(1) : "0.0";

    // Build markdown
    let md = `# Language stats for ${username}\n\n`;
    md += `**Top language:** ${top} (${topPct}%)\n\n`;
    md += `Total repositories counted: ${filtered.length}\n\n`;
    md += `| Language | Bytes | % |\n|---|---:|---:|\n`;
    for (const r of rows) {
      md += `| ${r.lang} | ${r.bytes.toLocaleString()} | ${r.pct.toFixed(1)}% |\n`;
    }

    // Write to file
    fs.writeFileSync(path.resolve(process.cwd(), outputFile), md, "utf8");
    console.log(`Wrote ${outputFile}`);
    process.exit(0);
  } catch (err) {
    console.error(err);
    process.exit(2);
  }
})();

