# GSOD Continual Learning Data Preparation

This standalone data preparation project converts NOAA Global Summary of the
Day (GSOD) station records into chronological seasonal tasks for continual
forecasting experiments. It prepares data for a separate model repository; it
does not train FedMixer or another model. NOAA's archive contains one
`YEAR.tar.gz` per year, with station CSV files inside. The archive reader can
extract only the selected stations.

Raw and generated files belong under `data/global_surface/` and are ignored by
Git. The NOAA archive directory can stay outside the repository, for example
`C:\Users\aarav\Downloads\GSOD` on Windows. GSOD and its successor SSOD v2
are different data products; do not mix them within one experiment.

GSOD observations end on 2025-08-28; NOAA stopped updating the product on
2025-08-29 and identifies SSOD v2 as its successor. See the
[NOAA archive index](https://www.ncei.noaa.gov/data/global-summary-of-the-day/archive/)
and [NOAA GSOD end-of-life notice](https://www.ncei.noaa.gov/operating-system-upgrade-outage).

The annual `.tar.gz` archives from 1929–2025 sum to about 4.72 GB compressed;
2010–2019 is about 1.04 GB. NOAA's bulk order interface also reports a roughly
40.55 GB output total for its separate bulk delivery. Extracted station CSVs
need additional storage, so download only needed years and extract only the
stations used in the experiment. A GPU is not needed for data preparation.

## Setup

Run commands from this repository's root. Python 3.10 or newer is recommended.

### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If PowerShell blocks activation, run `Set-ExecutionPolicy -Scope Process Bypass`
in that PowerShell window, or call `.\.venv\Scripts\python.exe` instead of
`python`.

### Linux / SSH

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Windows quick start

The folder you mentioned currently has the 1929 archive. That archive is only
about 49 KB and does not contain a full seasonal year for most stations. For a
useful first run, download a recent decade into that same folder. In
PowerShell, from the repository root:

```powershell
python -m pipeline.global_surface.download_archives --start-year 2010 --end-year 2019 --archive-dir "C:\Users\aarav\Downloads\GSOD"
python -m pipeline.global_surface.archive inventory --archive-dir "C:\Users\aarav\Downloads\GSOD" --start-year 2010 --end-year 2019 --output .\data\global_surface\station_inventory.csv
```

Choose station IDs from `data\global_surface\station_inventory.csv` and put
the IDs you want in `pipeline\global_surface\stations.txt`, one ID per line.
Then run:

```powershell
python -m pipeline.global_surface.archive extract --archive-dir "C:\Users\aarav\Downloads\GSOD" --start-year 2010 --end-year 2019 --stations-file .\pipeline\global_surface\stations.txt --raw-dir .\data\global_surface\raw
python -m pipeline.global_surface.season_tasks --raw-dir .\data\global_surface\raw --output-dir .\data\global_surface\processed --features TEMP --input-days 31 --forecast-days 7
```

For a quick low-resource check, use one or two complete years and a few
stations. The pipeline creates only seasons with enough complete station data.

## 1. Get the yearly archives

If you already downloaded archives, point `--archive-dir` at that folder. To
download chosen years directly on this computer or an SSH machine:

```bash
python -m pipeline.global_surface.download_archives \
  --start-year 2010 \
  --end-year 2019 \
  --archive-dir ./data/global_surface/archives
```

Downloads show bytes, rate, and ETA; completed archives are skipped on a rerun
and interrupted `.part` downloads resume when NOAA accepts HTTP Range requests.
Download only the years needed. The NOAA archive index currently lists about
4.72 GB for all 1929–2025 yearly archives combined; the 2010–2019 subset is
about 1.0 GB compressed. Extracted CSVs require additional disk space.

## 2. Inspect station IDs available in your archive folder

This writes the same inventory format as the direct NOAA index inspector:

```bash
python -m pipeline.global_surface.archive inventory \
  --archive-dir "C:\Users\aarav\Downloads\GSOD" \
  --start-year 2010 \
  --end-year 2019 \
  --output ./data/global_surface/station_inventory.csv
```

On an SSH machine, substitute the archive directory, for example
`./data/global_surface/archives`. Choose station IDs from the inventory and
put one 11-digit ID per line in `pipeline/global_surface/stations.txt`.
Keeping the same station list across years gives clients consistent IDs;
stations absent in some archive years simply have no file for those years.
For transfer efficiency, download the archives on the SSH machine rather than
copying the extracted tree from Windows. If transferring manually, copy only
the selected `YEAR.tar.gz` files and the station list.

## 3. Extract selected station files

```bash
python -m pipeline.global_surface.archive extract \
  --archive-dir "C:\Users\aarav\Downloads\GSOD" \
  --start-year 2010 \
  --end-year 2019 \
  --stations-file ./pipeline/global_surface/stations.txt \
  --raw-dir ./data/global_surface/raw
```

If `--stations-file` is omitted, all station CSVs in the chosen archives are
extracted. Existing files are skipped. Extraction reports progress and ETA by
year and station files. Missing yearly archives are reported and skipped.
The archive files themselves stay intact.

## 4. Build seasonal tasks

```bash
python -m pipeline.global_surface.season_tasks \
  --raw-dir ./data/global_surface/raw \
  --output-dir ./data/global_surface/processed \
  --features TEMP \
  --input-days 31 \
  --forecast-days 7
```

Seasonal preprocessing reads one station at a time to keep memory use bounded.
It inserts missing calendar dates, converts GSOD missing-value sentinels to
nulls, and keeps zero readings as valid values. It skips a station/season when
temperature coverage is below 90%, or when the first/last observation falls
more than 7 days inside the season boundary. Short interior gaps of up to two
days are interpolated within a season only. Windows that still contain missing
feature values are excluded.

Each included station/season task produces a cleaned daily CSV and a compressed
NPZ with chronological train/test windows (80/20 by time). `manifest.json`
lists included and skipped station/season pairs. Scaling is not applied here;
fit it on training windows in the training stage. NPZ input arrays have shape
`(number_of_windows, input_days, features)` and target arrays have shape
`(number_of_windows, forecast_days, features)`. Outputs are organized as
`processed/stations/<station_id>/` and `processed/windows/<station_id>/`.

## Existing direct station download

For a small station/year selection, the original per-station CSV downloader is
still available and avoids annual archives:

```bash
python -m pipeline.global_surface.inspect \
  --start-year 2010 \
  --end-year 2019 \
  --output ./data/global_surface/station_inventory.csv

python -m pipeline.global_surface.download \
  --stations-file ./pipeline/global_surface/stations.txt \
  --start-year 2010 \
  --end-year 2019 \
  --raw-dir ./data/global_surface/raw
```

## Notes

- A station is one client; station records are never merged together.
- Seasons recur in timestamp order across years rather than pooling all winters
  or summers into out-of-order buckets.
- Season names follow hemisphere: Northern Hemisphere winter is DJF;
  Southern Hemisphere winter is JJA. Stations within 23.5 degrees of the
  equator are flagged because four temperature seasons may fit poorly there.
- GSOD temperatures remain in Fahrenheit. Other source units are also retained.
- For transfer to another machine, transfer only the selected year archives
  (or download them on that machine) and the station list. Avoid copying the
  entire extracted station tree if only a subset of stations is needed.
- Raw archives, extracted CSVs, station inventories, and processed outputs are
  local experiment data and should not be committed to Git.
- NOAA's data description is at
  [GSOD dataset description and downloads](https://catalog.data.gov/dataset/global-surface-summary-of-the-day-gsod).

