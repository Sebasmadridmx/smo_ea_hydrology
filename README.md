# smo_ea_hydrology

Pure Ruby client for the [Environment Agency Hydrology API](https://environment.data.gov.uk/hydrology/doc/reference) — fetches active 15-minute rainfall stations, their coverage dates, and timestamped readings over any date range.

No external dependencies. Uses only Ruby stdlib (`net/http`, `uri`, `json`, `date`, `time`).
Compatible with **InfoWorks ICM 2027** embedded Ruby.

[![Gem Version](https://badge.fury.io/rb/smo_ea_hydrology.svg)](https://rubygems.org/gems/smo_ea_hydrology)

---

## Installation

```bash
gem install smo_ea_hydrology
```

Or add to your Gemfile:

```ruby
gem "smo_ea_hydrology"
```

---

## Quick start

```ruby
require "smo_ea_hydrology"

client = SmoEaHydrology::Client.new

# List active 15-min rainfall stations
stations = client.rainfall_15min_stations
puts stations.first.label            # "Ulpha  Duddo"
puts stations.first.station_reference # "589359"
puts stations.first.measure_label    # "Rainfall 15min Total (mm)"

# Find a station by name or reference
matches = client.find_stations("Cosford")
station = matches.first

# Fetch readings for a date range
measure  = client.measures(station.station_reference).first
readings = client.readings(measure.id, from: "2024-06-01", to: "2024-06-07")
puts readings.size        # 672
puts readings.first.value # 0.0 mm
```

---

## Features

### Stations

```ruby
stations = client.rainfall_15min_stations
# Returns Array<Station> — no coverage dates (fast)

stations = client.rainfall_15min_stations_with_coverage
# Returns Array<Station> with coverage_from / coverage_to populated (slow — 2 API calls per station)

matches = client.find_stations("Dartmoor")   # partial name match
matches = client.find_stations("589359")     # exact reference match
```

Each `Station` has:

| Field | Description |
|---|---|
| `label` | Station name |
| `station_reference` | Unique reference e.g. `"589359"` |
| `lat` / `long` | WGS84 coordinates |
| `easting` / `northing` | OSGB36 coordinates |
| `date_opened` | e.g. `"1990-10-04"` |
| `measure_label` | `"Rainfall 15min Total (mm)"` |
| `measure_id` | Full URI of the 15-min measure |
| `coverage_from` | `Time` of earliest reading (nil unless fetched) |
| `coverage_to` | `Time` of latest reading (nil unless fetched) |

### Readings

```ruby
measures = client.measures("589359")
readings = client.readings(measures.first.id, from: "2024-06-01", to: "2024-06-07")

# Time-of-day filtering (times are UTC)
readings = client.readings(measures.first.id,
  from: "2024-06-01 09:00",
  to:   "2024-06-01 17:00")

# Date / Time objects also accepted
readings = client.readings(measures.first.id,
  from: Date.new(2024, 6, 1),
  to:   Time.utc(2024, 6, 7, 23, 45))
```

Each `Reading` has: `datetime` (Time), `value` (Float, mm), `quality` (String), `completeness` (String).

### Download to CSV

```ruby
# Single station
count = client.readings_to_csv(
  station_reference: "589359",
  from: "2024-06-01",
  to:   "2024-06-07",
  path: "ulpha_june2024.csv"
)

# Batch — multiple stations to individual files
results = client.batch_download(
  from:       "2024-06-01",
  to:         "2024-06-07",
  output_dir: "rainfall_data",
  refs:       %w[589359 1712 603111]   # nil = all stations
)

# Full inventory with coverage dates
client.rainfall_15min_inventory_to_csv("inventory.csv")
```

### Inventory

```ruby
entries = client.rainfall_15min_inventory
# Array<InventoryEntry> — station + measure + coverage_from + coverage_to
# Makes 2 API calls per station — use rainfall_15min_inventory_to_csv for bulk export
```

---

## Examples

| Script | What it does |
|---|---|
| `examples/01_stations.rb` | List stations with coverage dates |
| `examples/02_readings.rb` | Fetch readings and show daily totals |
| `examples/03_inventory.rb` | Full inventory export to CSV |
| `examples/04_single_download.rb` | Download one station to CSV (edit variables at top) |
| `examples/05_batch_download.rb` | Batch download multiple stations (edit variables at top) |

Run any example:

```bash
ruby examples/01_stations.rb
ruby examples/04_single_download.rb   # edit STATION / FROM / TO at the top first
```

---

## API coverage

| EA API endpoint | Method |
|---|---|
| `/id/stations?observedProperty=rainfall` | `rainfall_15min_stations` |
| `/id/measures?periodName=15min` | `measures(station_reference)` |
| `/id/measures/{id}/readings` | `readings(measure_id, from:, to:)` |
| Flood Monitoring `latestReading` | `rainfall_15min_stations_with_coverage` |

---

## License

MIT — see [LICENSE](LICENSE).

Built by [Sebastian Madrid Ontiveros](https://github.com/Sebasmadridmx).
If this gem saves you time, [buy me a coffee ☕](https://buymeacoffee.com/smadrid)
