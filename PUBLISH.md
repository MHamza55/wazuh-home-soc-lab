# How to publish this to GitHub

A one-time walkthrough. Assumes you have a GitHub account.

## 1. Add your screenshots first

Copy your redacted images into `screenshots/` using the filenames listed in
`screenshots/README.md`. Double-check nothing shows a password.

## 2. Install git (if needed)

Download from https://git-scm.com/download/win and install with defaults, then
in PowerShell:

```powershell
git --version
```

## 3. Initialise the repo

From inside this folder:

```powershell
cd path\to\wazuh-home-soc-lab
git init
git add .
git status          # review what's staged — confirm NO credentials or .ova/.iso
git commit -m "Home SOC lab: Wazuh SIEM with SSH brute-force detection"
```

## 4. Create the GitHub repo

On github.com → **New repository** → name it `wazuh-home-soc-lab` →
**Public** (so employers can see it) → do **not** add a README/license (you already have them) → Create.

## 5. Push

Copy the commands GitHub shows under "…or push an existing repository", they look like:

```powershell
git branch -M main
git remote add origin https://github.com/<your-username>/wazuh-home-soc-lab.git
git push -u origin main
```

## 6. Polish the repo page

- Add a short **description** and topics (`wazuh`, `siem`, `soc`, `detection-engineering`, `mitre-attack`, `blue-team`).
- Confirm the README renders — the Mermaid architecture diagram displays automatically on GitHub.
- Pin the repo to your profile.

## 7. Put it to work

- Add the repo link to your CV and LinkedIn.
- When you add Phase 2 (Windows + Sysmon) and a custom rule, commit them as new
  detection write-ups — a repo with visible progress over time reads far better
  than a single dump.

## Ongoing commits

```powershell
git add .
git commit -m "Add Windows + Sysmon endpoint and process-creation detection"
git push
```
