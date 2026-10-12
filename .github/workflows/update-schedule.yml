#!/usr/bin/env python3
"""
B2B L'office — génère schedule.json (saison régulière NBA 2026-27).

Lancé automatiquement chaque jour par GitHub Actions
(.github/workflows/update-schedule.yml). Peut aussi se lancer à la main :
    python3 build_schedule.py

Sources, dans l'ordre :
  1. CDN officiel NBA (souvent bloqué en 2026 : "Access Denied")
  2. ESPN, UNE requête PAR JOUR (ESPN refuse les plages de dates depuis oct. 2026)

Sécurité : si le calendrier obtenu est incomplet, le script s'arrête en erreur
SANS toucher au schedule.json existant. Le site garde donc la dernière bonne version.
"""
import datetime as dt
import json
import re
import sys
import time
import urllib.request
from concurrent.futures import ThreadPoolExecutor

SEASON_START = dt.date(2026, 10, 20)   # ouverture
SEASON_END = dt.date(2027, 4, 11)      # dernier jour de saison régulière
MIN_GAMES = 1150                       # 1200 matchs fixes (+30 de NBA Cup ajoutés en décembre)

UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/124.0 Safari/537.36")
NBA_URLS = [
    "https://cdn.nba.com/static/json/staticData/scheduleLeagueV2.json",
    "https://cdn.nba.com/static/json/staticData/scheduleLeagueV2_1.json",
]
NBA_HEADERS = {"User-Agent": UA, "Referer": "https://www.nba.com/",
               "Origin": "https://www.nba.com", "Accept": "application/json, text/plain, */*"}
ESPN_DAY = "https://site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard?dates="
ESPN_ALIAS = {"NY": "NYK", "GS": "GSW", "SA": "SAS", "UTAH": "UTA",
              "WSH": "WAS", "NO": "NOP", "PHO": "PHX"}
TEAMS = {"ATL", "BOS", "BKN", "CHA", "CHI", "CLE", "DAL", "DEN", "DET", "GSW", "HOU", "IND",
         "LAC", "LAL", "MEM", "MIA", "MIL", "MIN", "NOP", "NYK", "OKC", "ORL", "PHI", "PHX",
         "POR", "SAC", "SAS", "TOR", "UTA", "WAS"}


def get_json(url, headers=None, tries=4):
    last = None
    for k in range(tries):
        try:
            req = urllib.request.Request(url, headers=headers or {"User-Agent": UA,
                                                                   "Accept": "application/json"})
            with urllib.request.urlopen(req, timeout=30) as r:
                return json.loads(r.read().decode("utf-8"))
        except Exception as e:  # noqa: BLE001
            last = e
            time.sleep(1.5 * (k + 1))
    raise last


def us_to_iso(s):
    m = re.search(r"(\d{1,2})/(\d{1,2})/(\d{4})", str(s))
    return "%s-%02d-%02d" % (m.group(3), int(m.group(1)), int(m.group(2))) if m else str(s)[:10]


def from_nba():
    for url in NBA_URLS:
        try:
            raw = get_json(url, NBA_HEADERS, tries=2)
        except Exception as e:  # noqa: BLE001
            print("NBA  ", url.rsplit("/", 1)[-1], "->", e)
            continue
        games = []
        for gd in raw.get("leagueSchedule", {}).get("gameDates", []):
            iso = us_to_iso(gd.get("gameDate"))
            for g in gd.get("games", []):
                if str(g.get("gameId", ""))[:3] != "002":       # 002 = saison régulière
                    continue
                h = (g.get("homeTeam") or {}).get("teamTricode")
                a = (g.get("awayTeam") or {}).get("teamTricode")
                if h in TEAMS and a in TEAMS:
                    games.append({"date": iso, "home": h, "away": a,
                                  "utc": g.get("gameDateTimeUTC"),
                                  "arena": ", ".join(x for x in (g.get("arenaName"), g.get("arenaCity")) if x),
                                  "label": (g.get("gameLabel") or "").strip()})
        print("NBA  ", len(games), "matchs")
        if len(games) >= MIN_GAMES:
            return games
    return None


def espn_day(day):
    raw = get_json(ESPN_DAY + day.strftime("%Y%m%d"))
    out = []
    for ev in raw.get("events", []):
        season = ev.get("season")
        if season and str(season.get("type")) != "2":          # saison régulière seulement
            continue
        status = str(((ev.get("status") or {}).get("type") or {}).get("name", ""))
        if "POSTPONED" in status or "CANCELED" in status:        # match reporté/annulé
            continue
        comp = (ev.get("competitions") or [{}])[0]
        cs = comp.get("competitors") or []
        home = next((c for c in cs if c.get("homeAway") == "home"), None)
        away = next((c for c in cs if c.get("homeAway") == "away"), None)
        if not home or not away:
            continue
        h = (home.get("team") or {}).get("abbreviation", "")
        a = (away.get("team") or {}).get("abbreviation", "")
        h, a = ESPN_ALIAS.get(h, h), ESPN_ALIAS.get(a, a)
        if h not in TEAMS or a not in TEAMS:                      # ex. "TBD" de la NBA Cup
            continue
        utc = comp.get("date") or ev.get("date") or ""
        if re.search(r"T\d\d:\d\dZ$", utc):
            utc = utc[:-1] + ":00Z"
        v = comp.get("venue") or {}
        out.append({"date": day.isoformat(), "home": h, "away": a, "utc": utc,
                    "arena": ", ".join(x for x in (v.get("fullName"), (v.get("address") or {}).get("city")) if x),
                    "label": ""})
    return out


def from_espn():
    days = [SEASON_START + dt.timedelta(n) for n in range((SEASON_END - SEASON_START).days + 1)]
    games, failed = [], []
    with ThreadPoolExecutor(max_workers=6) as ex:
        futures = {ex.submit(espn_day, d): d for d in days}
        for f, d in futures.items():
            try:
                games += f.result()
            except Exception as e:  # noqa: BLE001
                failed.append(d.isoformat())
                print("ESPN ", d, "->", e)
    seen, uniq = set(), []
    for g in games:
        k = (g["date"], g["home"], g["away"])
        if k not in seen:
            seen.add(k)
            uniq.append(g)
    print("ESPN ", len(uniq), "matchs sur", len(days), "jours,", len(failed), "jour(s) en échec")
    if failed or len(uniq) < MIN_GAMES:
        return None
    return uniq


def sanity(games):
    """Chaque équipe doit avoir ~80-82 matchs et jamais deux le même jour."""
    per_team, per_day = {}, set()
    for g in games:
        for t in (g["home"], g["away"]):
            per_team[t] = per_team.get(t, 0) + 1
            if (t, g["date"]) in per_day:
                raise SystemExit("ERREUR : %s joue deux fois le %s" % (t, g["date"]))
            per_day.add((t, g["date"]))
    low = {t: n for t, n in per_team.items() if n < 78}
    if len(per_team) != 30 or low:
        raise SystemExit("ERREUR : répartition anormale %s" % (low or per_team))


def main():
    games, source = from_nba(), "NBA"
    if not games:
        games, source = from_espn(), "ESPN"
    if not games:
        raise SystemExit("ÉCHEC : aucune source complète. schedule.json existant conservé.")
    sanity(games)
    games.sort(key=lambda g: (g["date"], g["home"]))
    data = {"generated_at": dt.datetime.now(dt.timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
            "source": source, "season": "2026-27", "games": games}
    with open("schedule.json", "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, separators=(",", ":"))
    print("OK   schedule.json :", len(games), "matchs (", games[0]["date"], "->",
          games[-1]["date"], ") source", source)


if __name__ == "__main__":
    main()
