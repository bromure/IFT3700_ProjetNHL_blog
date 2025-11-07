---
layout: post
title: "Downloading NHL Play-by-Play Data Automatically"
author: xavier_drolet
---

This post explains how our team automated the download of NHL play-by-play datasets for multiple seasons.  
If you were looking for a clear guide on how to retrieve raw NHL data straight from the official API — this is it.

We’ll cover:
- How the data retrieval process works  
- How to use the Python module we developed  
- What makes it robust and reusable for multiple seasons

---

## Overview

The NHL provides open access to its play-by-play data through the following base API endpoint: https://api-web.nhle.com/v1/gamecenter/{GAME_ID}/play-by-play. 

Each dataset corresponds to a unique `GAME_ID`, which encodes the **season**, **game type** (regular, playoffs, etc.), and **game number**. For example, for a `GAME_ID = 2023020001`:
- **Season**: The first four digits, provide the starting year of the season. In this case: 2023-2024. 
- **Game type**: The next two digits, where `01 = preseason`, `02 = regular season`, `03 = playoffs`, `04 = all-star`. In this case, we are in the regular season.
- **Game number**: The last 4 digits provide the game number contained in the total amount of games played for the preseason and season games. For the playoffs, the 4 digits follow this template: `0-round-matchup-game`. For this game `0001`, Tampa Bay won 5-3 against Nashville in the first game of the regular season!

More details on this (and source for the above description) can be found at this [link](https://gitlab.com/dword4/nhlapi/-/blob/master/stats-api.md).
Our script automates the process of generating these game IDs, checking for existing local data, and downloading missing games efficiently.

---

## Implementation

Here’s the core logic we implemented, structured into reusable functions:

```python
def get_web_data(file_name: str, download_url: str, save_path: str) -> Path:
    """
    Download a dataset from a remote API and save it locally.
    """
    response = requests.get(download_url, stream=True)
    try:
        response.raise_for_status()
    except requests.exceptions.HTTPError as e:
        logger.warning(f"Failed to download from {download_url} - HTTP {response.status_code}: {e}")
        return None
    else:
        save_path = Path(save_path)
        save_path.mkdir(parents=True, exist_ok=True)
        file_path = save_path / file_name

        with open(file_path, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        logger.info(f"Downloaded and saved file to {file_path}")

    return file_path
```

This helper function streams the dataset directly from the API and writes it to disk in manageable chunks, ensuring low memory usage and allowing for very large seasons to be downloaded safely.

The higher-level logic is handled by `get_data`, `get_yearly_data`, and `load_data`, which orchestrate the full retrieval of all games in a given year (or across several seasons).

---

## Usage Example

To reproduce our workflow, clone the repository and run:

```bash
python Visualisation/dataset.py main
```

You can customize the range of seasons and the base API endpoint via command-line arguments or directly in the function call:

```python
from Visualisation.dataset import load_data

load_data(
    input_path="data/raw/",
    download_url="https://api-web.nhle.com/v1/",
    years=[2021, 2022, 2023]
)
```

This will:

**1.** Create a structured data directory (organized by year).
**2.** Check whether local files already exist.
**3.** Download only missing games using the NHL API.
**4.** Retry any failed downloads automatically.

The system logs all activity and errors via `loguru`, providing transparent feedback on download progress and missing data.

---

## Example Directory Structure

Once complete, your data folder will look like this:

```yaml
data/
└── raw/
    ├── 2021/
    │   ├── play_by_play_2021020001_data.json
    │   ├── play_by_play_2021020002_data.json
    │   └── ...
    ├── 2022/
    ├── 2023/
    └── ...
```

Each `.json` file represents one NHL game’s complete play-by-play record.

---

## Notes

- The script uses `tqdm` for progress bars, `requests` for HTTP requests, and `loguru` for consistent logging.

- It supports different game types (regular, playoff) automatically.

- A built-in heuristic ensures that incomplete or missing data can be reattempted safely.

- The `load_data()` function is fully modular and can be reused for other hockey datasets that share the same API format.