# Preservation Services Documentation

This project contains documentation for APTrust's Preservation Services, including ingest, restoration, deletion, and fixity checking.

This is a source repository. The public-facing documentation lives at https://aptrust.github.io/preserv-docs/.

## Local Editing

If you want to edit this documentation locally, you'll need Python 3.x and pip. To set it all up, just run
`pip install -r requirements.txt`. Then you can start the server with `mkdocs serve`.

If you get a message saying mkdocs is not installed, try running the server with this command: `python3 -m mkdocs serve`

## Deploying to GitHub pages

```
mkdocs gh-deploy
```

or

```
python3 -m mkdocs gh-deploy
```
