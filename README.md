#!/usr/bin/env python3
"""
github_super_report.py
A
A powerful script that fetches all public repositories of a GitHub user,
gathers rich metadata, optionally clones repos, computes summaries, builds
charts, and writes:
  - an Excel workbook with multiple 
  - a Markdown summary README
  - a simple HTML dashboard with embedded charts

Features:
  - Pagination support for GitHub API
  - Optional personal access token to increase rate limits
  - Threaded fetching for contributors and releases
  - Rate-limit awareness (inspects headers and sleeps if necessary)
  - Optional git cloning (subprocess -> requires 'git' on PATH)
  - Excel export (pandas + openpyxl)
  - Charts using matplotlib (no custom colors set)
  - Graceful error handling and logging

Usage:ah 
  python github_super_report.py --username torvalds --token YOUR_TOKEN --output-dir ./out --clone

Dependencies:
  pip install requests pandas openpyxl matplotlib tqdm python-dateutil
  (git required if using --clone)

Author: ChatGPT (sample advanced script)
"""

import os
import sys
import argparse
import requests
import math
import time
import logging
import subprocess
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
from dateutil import parser as dtparser
from collections import Counter, defaultdict

import pandas as pd
import matplotlib.pyplot as plt
from tqdm import tqdm

# ---------- Configuration ----------
API_BASE = "https://api.github.com"
DEFAULT_PER_PAGE = 100
MAX_WORKERS = 8
# -----------------------------------

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

# ---------- Helper functions ----------

def get_headers(token=None):
    headers = {
        "Accept": "application/vnd.github.v3+json",
        "User-Agent": "github-super-report-script"
    }
    if token:
        headers["Authorization"] = f"token {token}"
    return headers

def check_rate_limit(response):
    """
    Inspect response headers for rate-limit and optionally sleep.
    Returns remaining, reset_epoch.
    """
    remaining = response.headers.get("X-RateLimit-Remaining")
    reset = response.headers.get("X-RateLimit-Reset")
    try:
        remaining = int(remaining) if remaining is not None else None
        reset = int(reset) if reset is not None else None
    except ValueError:
        remaining, reset = None, None
    return remaining, reset

def maybe_wait_for_rate_limit(remaining, reset):
    if remaining is not None and remaining <= 2 and reset is not None:
        now = int(time.time())
        wait = max(reset - now + 5, 0)  # small buffer
        logging.warning("Rate limit nearly exhausted. Sleeping for %d seconds.", wait)
        time.sleep(wait)

def paged_get(url, params=None, token=None):
    """
    Generic pager for GitHub API that yields items from a paginated endpoint.
    """
    params = params or {}
    page = 1
    while True:
        params.update({"per_page": DEFAULT_PER_PAGE, "page": page})
        resp = requests.get(url, headers=get_headers(token), params=params, timeout=30)
        if resp.status_code != 200:
            logging.error("Failed GET %s (status %s): %s", url, resp.status_code, resp.text[:300])
            resp.raise_for_status()
        remaining, reset = check_rate_limit(resp)
        maybe_wait_for_rate_limit(remaining, reset)
        data = resp.json()
        if not isinstance(data, list):
            # Some endpoints return dicts (e.g., single resource). Yield and return.
            yield data
            return
        if not data:
            break
        for item in data:
            yield item
        # check if we have fewer than per_page -> done
        if len(data) < DEFAULT_PER_PAGE:
            break
        page += 1

def safe_get(url, token=None, params=None):
    """
    Simple GET with error handling and rate-limit awareness.
    """
    resp = requests.get(url, headers=get_headers(token), params=params, timeout=30)
    if resp.status_code not in (200, 201, 202, 204):
        logging.warning("GET %s returned %s", url, resp.status_code)
    remaining, reset = check_rate_limit(resp)
    maybe_wait_for_rate_limit(remaining, reset)
    try:
        return resp.json()
    except ValueError:
        return resp.text

def ensure_dir(path):
    if not os.path.exists(path):
        os.makedirs(path, exist_ok=True)
    return path

# ---------- Repo Data Collectors ----------

def fetch_repos(username, token=None, include_forks=True, max_repos=None):
    url = f"{API_BASE}/users/{username}/repos"
    repos = []
    for repo in paged_get(url, token=token, params={"type": "all"}):
        if not include_forks and repo.get("fork"):
            continue
        repos.append(repo)
        if max_repos and len(repos) >= max_repos:
            break
    logging.info("Fetched %d repositories for user %s", len(repos), username)
    return repos

def fetch_latest_release(owner, repo_name, token=None):
    url = f"{API_BASE}/repos/{owner}/{repo_name}/releases/latest"
    resp = requests.get(url, headers=get_headers(token))
    if resp.status_code == 404:
        return None
    remaining, reset = check_rate_limit(resp)
    maybe_wait_for_rate_limit(remaining, reset)
    return resp.json()

def fetch_contributors(owner, repo_name, token=None, top_n=10):
    url = f"{API_BASE}/repos/{owner}/{repo_name}/contributors"
    contributors = []
    try:
        for c in paged_get(url, token=token):
            contributors.append(c)
            if len(contributors) >= top_n:
                break
    except Exception as e:
        logging.warning("Contrib fetch failed for %s/%s: %s", owner, repo_name, str(e))
    return contributors

def clone_repo_to(repo_url, target_dir):
    try:
        subprocess.check_call(["git", "clone", "--depth", "1", repo_url], cwd=target_dir)
        return True
    except Exception as e:
        logging.warning("Git clone failed for %s: %s", repo_url, str(e))
        return False

# ---------- Reporting / Charts ----------

def create_language_pie_chart(lang_counter, out_path):
    labels = []
    sizes = []
    for lang, count in lang_counter.most_common():
        labels.append(f"{lang} ({count})" if lang else "Unknown")
        sizes.append(count)
    if not sizes:
        logging.info("No language data to plot.")
        return None
    fig = plt.figure(figsize=(8, 6))
    plt.pie(sizes, labels=labels, autopct="%1.1f%%")
    plt.title("Repository language distribution")
    plt.tight_layout()
    fig.savefig(out_path)
    plt.close(fig)
    logging.info("Saved language pie chart to %s", out_path)
    return out_path

def create_top_stars_bar(repos, out_path, top_n=15):
    df = sorted(repos, key=lambda r: r.get("stargazers_count", 0), reverse=True)[:top_n]
    names = [r["name"] for r in df]
    stars = [r.get("stargazers_count", 0) for r in df]
    if not stars:
        logging.info("No star data to plot.")
        return None
    fig = plt.figure(figsize=(10, max(4, len(names)*0.4)))
    plt.barh(range(len(names))[::-1], stars)
    plt.yticks(range(len(names)), names[::-1])
    plt.xlabel("Stars")
    plt.title(f"Top {len(names)} repos by stars")
    plt.tight_layout()
    fig.savefig(out_path)
    plt.close(fig)
    logging.info("Saved stars bar chart to %s", out_path)
    return out_path

# ---------- Main workflow ----------

def build_report(username, token=None, output_dir="output", include_forks=True, max_repos=None, clone=False):
    ensure_dir(output_dir)
    repos = fetch_repos(username, token=token, include_forks=include_forks, max_repos=max_repos)

    # Collect extended info in parallel: contributors and latest release
    extended = {}
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as ex:
        futures = {}
        for repo in repos:
            owner = repo["owner"]["login"]
            name = repo["name"]
            futures[ex.submit(fetch_contributors, owner, name, token, 5)] = ("contributors", name)
            futures[ex.submit(fetch_latest_release, owner, name, token)] = ("release", name)

        # initialize
        for repo in repos:
            extended[repo["name"]] = {"contributors": None, "release": None}

        for fut in as_completed(futures):
            kind, repo_name = futures[fut]
            try:
                result = fut.result()
            except Exception as e:
                logging.warning("Background fetch failed for %s %s: %s", repo_name, kind, str(e))
                result = None
            extended[repo_name][kind] = result

    # Build DataFrame rows
    rows = []
    language_counter = Counter()
    topic_counter = Counter()
    license_counter = Counter()
    for repo in repos:
        name = repo["name"]
        owner = repo["owner"]["login"]
        language = repo.get("language")
        language_counter.update([language or "Unknown"])
        topics = safe_get(f"{API_BASE}/repos/{owner}/{name}/topics", token=token, params=None)
        topics_list = topics.get("names") if isinstance(topics, dict) else []
        topic_counter.update(topics_list or [])
        license_info = (repo.get("license") or {}).get("name") if repo.get("license") else None
        license_counter.update([license_info or "No license"])
        latest_release = extended[name]["release"]
        latest_release_tag = latest_release.get("tag_name") if isinstance(latest_release, dict) else None
        contributors = extended[name]["contributors"] or []
        top_contribs = [(c.get("login"), c.get("contributions")) for c in contributors[:5]]
        row = {
            "name": name,
            "full_name": repo.get("full_name"),
            "html_url": repo.get("html_url"),
            "description": repo.get("description"),
            "language": language,
            "stargazers_count": repo.get("stargazers_count"),
            "forks_count": repo.get("forks_count"),
            "open_issues_count": repo.get("open_issues_count"),
            "watchers_count": repo.get("watchers_count"),
            "size_kb": repo.get("size"),
            "created_at": repo.get("created_at"),
            "updated_at": repo.get("updated_at"),
            "pushed_at": repo.get("pushed_at"),
            "topics": ", ".join(topics_list) if topics_list else "",
            "license": license_info or "",
            "default_branch": repo.get("default_branch"),
            "is_fork": repo.get("fork"),
            "latest_release_tag": latest_release_tag,
            "top_contributors": ", ".join(f"{t[0]} ({t[1]})" for t in top_contribs if t[0])
        }
        rows.append(row)

    df = pd.DataFrame(rows)
    df_sorted = df.sort_values(by="stargazers_count", ascending=False)

    # Excel writer with multiple sheets
    excel_path = os.path.join(output_dir, f"{username}_repos_report.xlsx")
    with pd.ExcelWriter(excel_path, engine="openpyxl") as writer:
        df_sorted.to_excel(writer, sheet_name="Repositories", index=False)
        # languages
        lang_df = pd.DataFrame(language_counter.items(), columns=["Language", "Count"]).sort_values("Count", ascending=False)
        lang_df.to_excel(writer, sheet_name="Languages", index=False)
        # topics
        topics_df = pd.DataFrame(topic_counter.items(), columns=["Topic", "Count"]).sort_values("Count", ascending=False)
        topics_df.to_excel(writer, sheet_name="Topics", index=False)
        # licenses
        lic_df = pd.DataFrame(license_counter.items(), columns=["License", "Count"]).sort_values("Count", ascending=False)
        lic_df.to_excel(writer, sheet_name="Licenses", index=False)
    logging.info("Excel report written to %s", excel_path)

    # Charts
    charts_dir = os.path.join(output_dir, "charts")
    ensure_dir(charts_dir)
    lang_chart = create_language_pie_chart(language_counter, os.path.join(charts_dir, "language_distribution.png"))
    stars_chart = create_top_stars_bar(repos, os.path.join(charts_dir, "top_stars.png"))

    # Markdown summary
    md_lines = []
    md_lines.append(f"# GitHub Super Report for `{username}`")
    md_lines.append("")
    md_lines.append(f"Generated on {datetime.utcnow().isoformat()} UTC")
    md_lines.append("")
    md_lines.append(f"Total repositories analyzed: **{len(repos)}**")
    md_lines.append("")
    md_lines.append("## Top repositories by stars")
    md_lines.append("")
    top_by_stars = df_sorted.head(10)
    for _, r in top_by_stars.iterrows():
        md_lines.append(f"- [{r['name']}]({r['html_url']}) — ⭐ {r['stargazers_count']} — 🍴 {r['forks_count']} — {r['language'] or 'Unknown'}")
    md_lines.append("")
    if lang_chart:
        md_lines.append("## Language distribution")
        md_lines.append("")
        md_lines.append(f"![language distribution]({os.path.relpath(lang_chart, output_dir)})")
        md_lines.append("")
    if stars_chart:
        md_lines.append("## Stars chart")
        md_lines.append("")
        md_lines.append(f"![top stars]({os.path.relpath(stars_chart, output_dir)})")
        md_lines.append("")

    md_lines.append("## Repository table")
    md_lines.append("")
    md_lines.append("| Name | Stars | Forks | Language | Top Contributors | Latest Release |")
    md_lines.append("|---|---:|---:|---|---|---|")
    for _, r in df_sorted.iterrows():
        md_lines.append(
            f"| [{r['name']}]({r['html_url']}) | {r['stargazers_count']} | {r['forks_count']} | {r['language'] or ''} | {r['top_contributors']} | {r['latest_release_tag'] or ''} |"
        )

    md_content = "\n".join(md_lines)
    md_path = os.path.join(output_dir, f"{username}_README_summary.md")
    with open(md_path, "w", encoding="utf-8") as f:
        f.write(md_content)
    logging.info("Markdown summary written to %s", md_path)

    # Simple HTML dashboard
    html_lines = []
    html_lines.append("<!doctype html>")
    html_lines.append("<html><head><meta charset='utf-8'><title>GitHub Super Report</title></head><body>")
    html_lines.append(f"<h1>GitHub Super Report for {username}</h1>")
    html_lines.append(f"<p>Generated at {datetime.utcnow().isoformat()} UTC</p>")
    html_lines.append(f"<p>Total repositories: {len(repos)}</p>")
    if lang_chart:
        html_lines.append(f"<h2>Language distribution</h2><img src='{os.path.relpath(lang_chart, output_dir)}' style='max-width:800px'/>")
    if stars_chart:
        html_lines.append(f"<h2>Top repos by stars</h2><img src='{os.path.relpath(stars_chart, output_dir)}' style='max-width:1000px'/>")
    html_lines.append("<h2>Top repositories</h2><ul>")
    for _, r in top_by_stars.iterrows():
        html_lines.append(f"<li><a href='{r['html_url']}' target='_blank'>{r['name']}</a> — ⭐ {r['stargazers_count']} — {r['language'] or 'Unknown'}</li>")
    html_lines.append("</ul>")
    html_lines.append("</body></html>")
    html_path = os.path.join(output_dir, f"{username}_dashboard.html")
    with open(html_path, "w", encoding="utf-8") as f:
        f.write("\n".join(html_lines))
    logging.info("HTML dashboard written to %s", html_path)

    # Optional cloning
    clones_dir = os.path.join(output_dir, "clones")
    if clone:
        ensure_dir(clones_dir)
        logging.info("Starting cloning of %d repositories to %s", len(repos), clones_dir)
        for repo in tqdm(repos):
            clone_repo_to(repo["clone_url"], clones_dir)

    # Return summary info
    summary = {
        "username": username,
        "total_repos": len(repos),
        "excel": excel_path,
        "markdown": md_path,
        "html": html_path,
        "charts": {"language": lang_chart, "stars": stars_chart}
    }
    return summary

# ---------- CLI ----------

def parse_args():
    p = argparse.ArgumentParser(description="Generate a super report for a GitHub user's repositories.")
    p.add_argument("--username", "-u", required=True, help="GitHub username to analyze")
    p.add_argument("--token", "-t", help="GitHub personal access token (optional but recommended)")
    p.add_argument("--output-dir", "-o", default="github_super_report_output", help="Directory to place outputs")
    p.add_argument("--no-forks", action="store_true", help="Exclude forked repositories")
    p.add_argument("--max-repos", type=int, help="Maximum number of repositories to process (for testing)")
    p.add_argument("--clone", action="store_true", help="Clone repositories (requires git installed)")
    p.add_argument("--silent", action="store_true", help="Reduce logging verbosity")
    return p.parse_args()

def main():
    args = parse_args()
    if args.silent:
        logging.getLogger().setLevel(logging.WARNING)
    token = args.token or os.environ.get("GITHUB_TOKEN")
    try:
        summary = build_report(
            username=args.username,
            token=token,
            output_dir=args.output_dir,
            include_forks=(not args.no_forks),
            max_repos=args.max_repos,
            clone=args.clone
        )
        print("\nReport generation complete.")
        print(f"Excel: {summary['excel']}")
        print(f"Markdown: {summary['markdown']}")
        print(f"HTML dashboard: {summary['html']}")
        if summary["charts"]["language"]:
            print(f"Language chart: {summary['charts']['language']}")
        if summary["charts"]["stars"]:
            print(f"Stars chart: {summary['charts']['stars']}")
    except KeyboardInterrupt:
        logging.error("Interrupted by user")
        sys.exit(1)
    except Exception as e:
        logging.exception("Unhandled exception: %s", str(e))
        sys.exit(2)

if __name__ == "__main__":
    main()
