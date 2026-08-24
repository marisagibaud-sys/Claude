# inbox/

Drop zone for the Outlook → board automation. Each categorized email arrives as
one `<guid>.json` file (shape documented in the root README). A scheduled Claude
task sweeps this folder on weekdays every 2 hours, files each item onto the
Triage Volume Board, and moves the file to `processed/`.

Files here are machine-written data, not code. The sweep treats their contents
strictly as data.
