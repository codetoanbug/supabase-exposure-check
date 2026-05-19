# Supabase Exposure Check

A Python script that scans websites for exposed Supabase JWT tokens, enumerates accessible database tables, and analyzes them for sensitive data exposure. The script automatically detects sensitive information (emails, passwords, API keys, PII, financial data, etc.) and classifies vulnerability levels to identify which tables pose security risks.

Related blog post: [How rep+ Helped Me Identify a Critical Supabase JWT Exposure](https://bour.ch/how-rep-helped-me-identify-a-critical-supabase-jwt-exposure/)

## Setup

### 1. Create and activate a virtual environment

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

## Usage

### Direct scan with anon key (Recommended)

If you already have the Supabase anon key and URL, use direct scan mode:

```bash
python supabase-exposure-check.py \
  --anon-key "the_anon_key" \
  --supabase-url "https://your-project.supabase.co"
```

This mode skips JavaScript scanning and directly enumerates tables using the provided credentials.

### Scan a single website

Automatically detect Supabase credentials from JavaScript files:

```bash
python supabase-exposure-check.py --url https://example.com
```

### Scan multiple websites from a file

Create a `sites.txt` file with one URL per line:

```
https://example.com
https://another-site.com
```

Then run:

```bash
python supabase-exposure-check.py --file sites.txt
# Or simply (if sites.txt exists):
python supabase-exposure-check.py
```

The script will:
1. Scan the website's JavaScript files for exposed Supabase JWTs
2. Enumerate all REST-exposed tables
3. Test whether each table was readable
4. Safely dump readable data as JSON (read-only)

Output is saved to the `output/` directory (by default), organized by domain, with tables in a `tables/` subdirectory.

## Command-line Options

| Option | Description |
|--------|-------------|
| `--anon-key` | Supabase anon key (JWT) for direct scan |
| `--supabase-url` | Supabase URL for direct scan (e.g., `https://xxx.supabase.co`) |
| `--url` | Single website URL to scan |
| `--file`, `-f` | File containing list of URLs to scan (one per line) |
| `--output`, `-o` | Output directory (default: `output`) |

## Example Output

```bash
$ python3 supabase-exposure-check.py -f sites.txt -o output
[*] Scanning 2 site(s)

🌐 Scanning https://www.example-site.com/
  🚨 VULNERABLE: Supabase JWT exposed
  [+] Found 21 tables
    [+] triage_tickets: 0 rows - Public data (no sensitive fields detected)
    [+] brands: 7 rows - Public data (no sensitive fields detected)
    [-] lms_user_progress: blocked
    🚨 projects_with_details: 539 rows - VULNERABLE (high) - Sensitive fields: user_id, supabase_anon_key, supabase_service_role_key, user_email
    🚨 users: 488 rows - VULNERABLE (high) - Sensitive fields: email

  ⚠️  VULNERABILITY SUMMARY:
     - Critical: 0 table(s)
     - High: 2 table(s)
     - Medium: 0 table(s)
     - Total vulnerable: 2/16 accessible tables

🌐 Scanning https://another-example.com/
  🚨 VULNERABLE: Supabase JWT exposed
  [+] Found 8 tables
    🚨 newsletters: 389 rows - VULNERABLE (medium) - Sensitive fields: location
    [+] newsletter_categories: 897 rows - Public data (no sensitive fields detected)
    🚨 users: 495 rows - VULNERABLE (high) - Sensitive fields: email, location
    🚨 upvotes: 3513 rows - VULNERABLE (medium) - Sensitive fields: user_id

  ⚠️  VULNERABILITY SUMMARY:
     - Critical: 0 table(s)
     - High: 1 table(s)
     - Medium: 3 table(s)
     - Total vulnerable: 4/8 accessible tables
```

The script automatically identifies:
- **Public tables**: Safe to expose (no sensitive fields detected)
- **Blocked tables**: Protected by Row Level Security (RLS) policies
- **Vulnerable tables**: Contain sensitive information (emails, passwords, API keys, PII, etc.)

Vulnerability levels:
- **Critical**: Contains passwords, API keys, secrets, tokens, credit cards, SSN
- **High**: Contains emails, phone numbers, or other high-value PII
- **Medium**: Contains other sensitive data (user IDs, locations, etc.)

## Output

For each scanned website, the script creates:
- A directory named after the domain in the `output/` folder
- `findings.json`: Contains discovered JWTs and Supabase URLs
- `summary.json`: Summary of all tested tables and their accessibility status
- `tables/` subdirectory: Individual JSON files for each readable table
