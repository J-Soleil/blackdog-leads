"""Checks the Greater Philadelphia Film Office crew-call board and writes feed.json.
Standard library only. Low volume: one or two list pages plus new posts."""
import json, re, html, urllib.request
from datetime import datetime, timezone, timedelta

BASE = "https://film.org"
LIST_URLS = [BASE + "/job-type/crew/", BASE + "/job-type/crew/page/2/"]
UA = "BlackDogStudiosLeadBoard/1.0 (personal job-board checker; low volume)"
MAX_DAYS, MAX_FETCH = 45, 30

def get(url):
    req = urllib.request.Request(url, headers={"User-Agent": UA})
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read().decode("utf-8", "replace")

def clean(s):
    return re.sub(r"\s+", " ", html.unescape(re.sub(r"<[^>]+>", " ", s))).strip()

def meta(page, prop):
    p = re.escape(prop)
    m = (re.search(r'<meta[^>]+(?:property|name)=["\']%s["\'][^>]+content=["\']([^"\']*)' % p, page, re.I)
         or re.search(r'<meta[^>]+content=["\']([^"\']*)["\'][^>]+(?:property|name)=["\']%s["\']' % p, page, re.I))
    return html.unescape(m.group(1)) if m else ""

STOP = r"(?=\s+(?:Compensation|Job Category|Project Name|Contact|Phone|Email|Job Description|Job Title):|$)"
def field(text, label):
    m = re.search(label + r":\s*(.+?)" + STOP, text)
    return m.group(1).strip() if m else ""

def parse_post(url, page):
    title = clean(meta(page, "og:title") or (re.search(r"<title>(.*?)</title>", page, re.S) or [None, ""])[1])
    title = re.sub(r"\s*[-|–]\s*Greater Philadelphia Film Office.*$", "", title, flags=re.I).strip()
    posted = meta(page, "article:published_time")
    text = clean(re.sub(r"<(script|style)[^>]*>.*?</\1>", " ", page, flags=re.S | re.I))
    return {"id": url, "url": url, "title": title, "posted": posted[:10],
            "comp": field(text, "Compensation"), "cat": field(text, "Job Category"),
            "project": field(text, "Project Name"),
            "source": "Greater Philadelphia Film Office", "region": "Philadelphia"}

def main():
    try:
        feed = json.load(open("feed.json"))
    except Exception:
        feed = {}
    items = {i["id"]: i for i in feed.get("items", [])}
    seen = feed.get("seen", {})
    links = []
    for lu in LIST_URLS:
        try:
            page = get(lu)
        except Exception as e:
            print("list fetch failed:", lu, e); continue
        for m in re.finditer(r'href=["\'](https://film\.org/job/[^"\'#?]+)', page):
            u = m.group(1).rstrip("/") + "/"
            if u not in links: links.append(u)
    print("found", len(links), "links")
    if not links:
        print("No links found; keeping existing feed."); return
    cutoff = (datetime.now(timezone.utc) - timedelta(days=MAX_DAYS)).strftime("%Y-%m-%d")
    fetched = 0
    for u in links:
        if u in seen or fetched >= MAX_FETCH: continue
        try:
            post = parse_post(u, get(u)); fetched += 1
        except Exception as e:
            print("post failed:", u, e); continue
        if post["posted"] and post["posted"] >= cutoff and post["title"]:
            items[u] = post; seen[u] = post["posted"]
        else:
            seen[u] = post["posted"] or "undated"
    keep = [i for i in items.values() if i["posted"] >= cutoff]
    keep.sort(key=lambda i: i["posted"], reverse=True)
    out = {"updated": datetime.now(timezone.utc).strftime("%Y-%m-%d"), "items": keep, "seen": seen}
    json.dump(out, open("feed.json", "w"), indent=1)
    print("saved", len(keep), "recent items")

if __name__ == "__main__":
    main()
