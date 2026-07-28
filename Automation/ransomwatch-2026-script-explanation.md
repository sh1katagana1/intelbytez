# Ransomwatch 2026 Script

***

## Goal
To create a new script to automate the process of me supplying a list of vendor names and having it check against the New Victims list from ransomware.live website. I want it to also curate my vendor list a bit and ignore common words and specific keywords I find redundant and not useful. 

## Script
```
import requests
import re

COMPANIES_FILE = "test-vendors.txt"
API_URL = "https://api.ransomware.live/v2/recentvictims"

NOISE_WORDS = {
    "llc", "inc", "ltd", "corp", "corporation", "co",
    "group", "solutions", "services", "service",
    "company", "plc", "holdings", "technologies", "tech",
    "international", "global"
}

LOW_VALUE_TOKENS = {
    "smith", "johnson", "brown", "wilson", "lee", "taylor",
    "and", "the", "of", "for"
}

GENERIC_TOKENS = {
    "global",
    "systems",
    "solutions",
    "technology",
    "technologies",
    "services",
    "consulting",
    "software",
    "medical",
    "health",
    "digital",
    "energy"
}


def load_companies(filename):
    with open(filename, "r") as f:
        return [line.strip() for line in f if line.strip()]


def fetch_recent_victims():
    r = requests.get(API_URL, timeout=30)
    r.raise_for_status()
    return r.json()


def extract_victim_name(victim: dict) -> str:
    return (
        victim.get("name")
        or victim.get("victim")
        or victim.get("title")
        or ""
    ).strip()


def is_masked_name(name: str) -> bool:
    """
    Detect partial/redacted victim names like:
    G*
    H**
    Acm*
    Micros***
    """
    name = name.strip()

    patterns = [
        r"[A-Za-z]\*+",     # G*
        r"[A-Za-z]{2,}\*+", # Acm*
        r".+\*+$"          # anything ending in ***
    ]

    return any(re.fullmatch(p, name) for p in patterns)


def normalize(text: str) -> str:
    text = text.lower()
    text = re.sub(r"[^\w\s]", " ", text)
    return text


def build_tokens(name: str):
    name = normalize(name)
    return [
        t for t in name.split()
        if t not in NOISE_WORDS and t not in LOW_VALUE_TOKENS
    ]


def is_high_value_single_token(token: str) -> bool:
    """
    Only allow single-token matching if the token is distinctive enough.
    """
    return (
        len(token) >= 7
        and token not in GENERIC_TOKENS
    )


def match(vendor: str, victim: str) -> bool:
    v_tokens = build_tokens(vendor)
    k_tokens = build_tokens(victim)

    if not v_tokens or not k_tokens:
        return False

    v_set = set(v_tokens)
    k_set = set(k_tokens)

    # CASE 1: exact match
    if v_set == k_set:
        return True

    # CASE 2: subset match (multi-token entities only)
    if len(v_set) >= 2 and (v_set.issubset(k_set) or k_set.issubset(v_set)):
        return True

    # CASE 3: single-token overlap (strict)
    if len(v_set) == 1 or len(k_set) == 1:
        common = list(v_set & k_set)

        if not common:
            return False

        token = common[0]

        if is_high_value_single_token(token):
            return True

        return False

    # CASE 4: partial overlap scoring
    overlap = len(v_set & k_set)
    score = overlap / max(len(v_set), len(k_set))

    return score >= 0.6


def match_companies(companies, victims):
    for company in companies:

        for victim in victims:

            victim_name = extract_victim_name(victim)

            # 🔥 FIX: skip masked / invalid names early
            if not victim_name or is_masked_name(victim_name):
                continue

            if match(company, victim_name):

                print("⚠️ MATCH FOUND")
                print(f"Vendor : {company}")
                print(f"Victim : {victim_name}")
                print(f"Group  : {victim.get('group','')}")
                print(f"Domain : {victim.get('domain','')}")
                print(f"Date   : {victim.get('attackdate','')}")
                print("-" * 50)


def main():
    companies = load_companies(COMPANIES_FILE)
    victims = fetch_recent_victims()
    match_companies(companies, victims)


if __name__ == "__main__":
    main()

```

## Script Explanation
### Importing Libraries
```
import requests
import re
```
A library is a collection of code that someone else already wrote so you don't have to.
1. The requests library is used to communicate with websites or APIs. For example:
```
requests.get("https://example.com")
```
asks a website for information. Think of it like: "Go to this website and bring me the data."

2. re stands for Regular Expressions. Regular expressions are patterns used to search text. Example:
```
re.search(r"\d+", "abc123")
```
finds numbers inside text. This script uses it to:
1. remove punctuation
2. detect masked names like A*, Micros***, H**

### Constants (configuration)
```
COMPANIES_FILE = "test-vendors.txt"
API_URL = "https://api.ransomware.live/v2/recentvictims"
```
These are variables whose values aren't expected to change while the program runs.
1. COMPANIES_FILE - stores the filename containing your vendors. Example file:
```
Microsoft
Cisco
Acme Inc
Dell
```
2. API_URL - stores the website where ransomware.live publishes recent victims. Later the program will download JSON data from this address.

### Lists of words to ignore
```
NOISE_WORDS = {
    "llc",
    "inc",
    "ltd",
    ...
}
```
A vendor list can have a ton of names that end in llc, corp, plc, etc. These are a bit broad to include for a keyword match, so we list all the words we dont want to be considered for a match. This Python code is called a list.

```
LOW_VALUE_TOKENS = {
    "smith",
    "johnson",
    ...
}
```
These are common words that appear in many company names. Example:
```
Smith Engineering
Smith Manufacturing
Smith Medical
```
Matching only the word "Smith" would create lots of false matches.

```
GENERIC_TOKENS = {
    "global",
    "services",
    "technology",
    ...
}
```
These words are too generic. For example,
1. Global
2. Solutions
3. Services

could describe thousands of companies. This is similar to my Keywords list.


### Loading the company list
```
def load_companies(filename):
```
This defines a function. A function is simply a reusable block of code. Imagine it as a small machine:

filename -> load_companies -> list of companies
```
with open(filename, "r") as f:
```
This opens the file. "r" means read mode and f is the opened file.
```
return [line.strip() for line in f if line.strip()]
```
This is called a list comprehension or a for loop. It means for every line, remove spaces, ignore blank lines and put it into a list. Example file:
```
Microsoft

Cisco

Dell
```
becomes
```
[
    "Microsoft",
    "Cisco",
    "Dell"
]
```

### Downloading the victims
```
def fetch_recent_victims():
```
This function talks to the ransomware API. 
```
r = requests.get(API_URL, timeout=30)
```
This sends an HTTP GET request. Imagine the program calls the API and it returns data. The timeout says: "If nothing happens after 30 seconds, stop waiting."
```
r.raise_for_status()
```
This checks for errors. For example:
```
404
500
403
```
If the website failed, Python raises an exception instead of continuing with bad data.
```
return r.json()
```
The API sends JSON. JSON looks like
```
[
  {
    "name": "Acme"
  },
  {
    "name": "Cisco"
  }
]
```
Python converts it into dictionaries and lists.

### Extracting the victim name
```
def extract_victim_name(victim):
```
Each victim is a dictionary. Example:
```
{
    "name": "Microsoft",
    "group": "LockBit"
}
```
Sometimes different APIs use different field names. This code tries several possibilities:
```
victim.get("name")
```
If that doesn't exist...
```
victim.get("victim")
```
If that doesn't exist...
```
victim.get("title")
```
If none exist:
```
""
```
an empty string is returned.

### Detecting masked names
```
def is_masked_name(name):
```
Some ransomware groups hide names. Examples:
1. G*
2. A**
3. Micros***

These aren't useful for matching. The patterns are:
```
r"[A-Za-z]\*+"
```
matches
1. G*
2. A***
```
r"[A-Za-z]{2,}\*+"
```
matches
1. Mic*
2. Acm**
```
r".+\*+$"
```
matches
1. Anything***
```
return any(...)
```
means: "If any pattern matches, return True."


### Normalizing text
```
def normalize(text):
```
This prepares text so comparisons are easier.
```
text.lower()
```
changes
```
MICROSOFT
```
to
```
microsoft
```
so uppercase/lowercase no longer matters.
```
re.sub(r"[^\w\s]", " ", text)
```
removes punctuation. Example:
```
AT&T
```
becomes
```
AT T
```

### Building tokens
```
def build_tokens(name):
```
A token is just a word. Example: Microsoft Global Services becomes
```
["microsoft", "global", "services"]
```
```
if t not in NOISE_WORDS
```
removes
1. services
2. inc
3. llc

The final result might be
```
["microsoft"]
```

### High-value single tokens
```
def is_high_value_single_token(token):
```
Suppose two names share only one word. Example:
1. Microsoft
2. Microsoft UK

The common word is Microsoft. That's distinctive enough but Global isn't. The rules are:
```
len(token) >= 7
```
The word must be at least seven letters long AND
```
token not in GENERIC_TOKENS
```
It cannot be generic.


### The matching algorithm (the heart of the program)
```
def match(vendor, victim):
```
This decides whether two names probably refer to the same company. It follows four increasingly flexible checks. First we build token sets
```
v_tokens = build_tokens(vendor)
k_tokens = build_tokens(victim)
```
Example: Acme Inc becomes {"acme"} and Acme Corporation also becomes {"acme"}. Using sets ignores duplicate words and makes comparisons easy. Let's say you have an exact match:
```
if v_set == k_set:
```
If both sets contain exactly the same words, it's a match. Example: {"acme"} equals {"acme"}. Let's say you have a subset match:
```
issubset()
```
Example: Vendor {"general", "electric"} and Victim {"general", "electric", "aviation"} Everything in the vendor name appears in the victim name, so it's considered a match. Let's say you have single-word matches. If one company has only one meaningful word like Microsoft and another has Microsoft Germany the shared word is Microsoft. Since it's long and distinctive, it's accepted. If the shared word were Global it would be rejected. let's say you had an overlap score. Suppose:
1. Vendor: United Health Systems
2. Victim: United Health Group

After removing generic words:
1. Vendor {"united", "health"}
2. Victim {"united", "health"}
3. Overlap: 2
4. Maximum size: 2
5. Score: 2 / 2 = 1.0
6. Since score >= 0.6 they match. This rule also helps when names are similar but not identical.

### Comparing every company to every victim
```
def match_companies(companies, victims):
```
This function checks every company against every victim using nested loops.
```
for company in companies:
```
Take one company from your file. Example: Cisco
```
for victim in victims:
```
Compare Cisco against every victim downloaded from the API. If there are: 100 companies and 500 victims the program performs: 100 × 500 = 50,000 comparisons. Inside the loop, it gets the victim's name:
```
victim_name = extract_victim_name(victim)
```
Then skips bad entries:
```
if not victim_name or is_masked_name(victim_name):
    continue
```
The continue statement means "skip the rest of this loop iteration and move on to the next victim." Finally, it checks:
```
if match(company, victim_name):
```
If the names match, it prints details such as the vendor, victim name, ransomware group, domain, and attack date.

### The main() function
```
def main():
```
This is the program's main workflow. It performs three steps in order:
```
companies = load_companies(COMPANIES_FILE)
```
Read your vendor list from the file. Then:
```
victims = fetch_recent_victims()
```
Download the latest ransomware victim data. Finally:
```
match_companies(companies, victims)
```
Compare every company against every victim and print any matches.


### The program entry point
```
if __name__ == "__main__":
    main()
```
This is a common Python pattern. When you run the file directly (for example, python script.py), Python sets a special variable named __name__ to "__main__". That makes the condition true, so:
```
main()
```
is executed. If another Python file imports this script, main() will not run automatically. That allows the functions to be reused without starting the whole program.

























