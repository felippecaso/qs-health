# qs-health

A "Quantified Self" project for Health-related data, using the Modern Data Stack.

## Installation

```bash
# Install meltano
pipx install meltano
# Initialize meltano within this directory
cd qs-health
meltano install
```

Then, edit the `meltano.yml` file with the appropriate configuration for each extractor and loader.

### Evidence

To run visuals with Evidence, execute:

```bash
cd evidence
npm install 
npm run dev 
```

## Environment

The current state of the project expects the following environment variables, which can be provided with a local `.env` file:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
MOTHERDUCK_TOKEN
```