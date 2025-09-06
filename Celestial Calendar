# merge_ics.py
# Downloads several public .ics feeds and merges VEVENTs into one combined.ics
# Requires Python 3.8+. Uses only stdlib.

import ssl
import urllib.request
from urllib.parse import urlparse
from datetime import datetime, timezone

# --- CONFIG: feeds to include (you can add / remove) ---
FEEDS = [
    # Wolfram (moon phases)
    "https://ical.wolfram.com/MoonPhases.ics",
    # Wolfram (solstices & equinoxes)
    "https://ical.wolfram.com/Solstices.ics",
    # NASA eclipse feed
    "https://eclipse.gsfc.nasa.gov/SEpubs/EclipseICal.ics",
    # Add any other public .ics URLs you like (meteor showers etc.)
]

OUTPUT_PATH = "combined.ics"

def fetch_text(url):
    ctx = ssl.create_default_context()
    with urllib.request.urlopen(url, context=ctx) as r:
        raw = r.read()
        # try to decode
        for enc in ("utf-8","latin-1","iso-8859-1"):
            try:
                return raw.decode(enc)
            except:
                pass
        return raw.decode("utf-8", errors="ignore")

def extract_vevents(ics_text):
    vevents = []
    inside = False
    buf = []
    for line in ics_text.splitlines():
        if line.strip().upper() == "BEGIN:VEVENT":
            inside = True
            buf = [line]
        elif line.strip().upper() == "END:VEVENT" and inside:
            buf.append(line)
            vevents.append("\n".join(buf))
            inside = False
            buf = []
        elif inside:
            buf.append(line)
    return vevents

def uid_safe(s):
    return "".join(c if c.isalnum() or c in "-_." else "_" for c in s)

def main():
    all_vevents = []
    sources = []
    for url in FEEDS:
        try:
            print("Fetching", url)
            txt = fetch_text(url)
            sources.append(url)
            ve = extract_vevents(txt)
            print(" -> found", len(ve), "vevents")
            all_vevents.extend(ve)
        except Exception as e:
            print("Error fetching", url, e)

    # Remove duplicates by UID/DTSTAMP+SUMMARY heuristic
    seen = set()
    unique = []
    for v in all_vevents:
        # try find UID, SUMMARY, DTSTART
        uid = ""
        dt = ""
        summary = ""
        for l in v.splitlines():
            if l.upper().startswith("UID:"):
                uid = l[4:].strip()
            if l.upper().startswith("DTSTART"):
                dt = l.split(":")[-1].strip()
            if l.upper().startswith("SUMMARY:"):
                summary = l[8:].strip()
        key = uid or (dt + "||" + summary)
        if key not in seen:
            seen.add(key)
            unique.append(v)

    # Build calendar header
    now = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    header = [
        "BEGIN:VCALENDAR",
        "VERSION:2.0",
        "CALSCALE:GREGORIAN",
        "METHOD:PUBLISH",
        'PRODID:-//YourName Celestial Calendar//EN',
        f"X-WR-CALNAME:Celestial Events - Moon Phases & Eclipses",
        f"X-WR-TIMEZONE:Europe/Berlin",
    ]
    footer = ["END:VCALENDAR"]

    content = "\n".join(header) + "\n"
    # Append each VEVENT
    for v in unique:
        # ensure event has UID; if not add one
        if "UID:" not in v.upper():
            # crude summary extraction
            summary = ""
            for l in v.splitlines():
                if l.upper().startswith("SUMMARY:"):
                    summary = l[8:].strip()
                    break
            uid = f"{uid_safe(summary)}-{len(unique)}@merged"
            # insert UID after BEGIN:VEVENT
            v = v.replace("BEGIN:VEVENT\n", f"BEGIN:VEVENT\nUID:{uid}\n", 1)
        content += v + "\n"
    content += "\n".join(footer) + "\n"

    with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
        f.write(content)
    print("Wrote", OUTPUT_PATH, "with", len(unique), "events")

if __name__ == "__main__":
    main()
