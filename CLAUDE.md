# CLAUDE.md — Amadeus Flight Booking Django

## Project Overview

A Django-based prototype/demo application that showcases the Amadeus travel APIs. It allows users to search for flights, view results, and complete a mock booking — all powered by live Amadeus API calls. There is no local database persistence; all data is fetched from Amadeus at runtime.

---

## Repository Structure

```
amadeus-flight-booking-django/
├── amadeus_demo_api/                  # Django project root (manage.py lives here)
│   ├── amadeus_demo_api/              # Project configuration package
│   │   ├── settings.py                # Django settings
│   │   ├── urls.py                    # Root URL configuration
│   │   └── wsgi.py                    # WSGI entry point
│   ├── demo/                          # Single Django app with all application logic
│   │   ├── templates/demo/            # HTML templates (home, results, book_flight)
│   │   ├── static/demo/style.css      # Custom CSS
│   │   ├── views.py                   # All view functions
│   │   ├── urls.py                    # App-level URL patterns
│   │   ├── flight.py                  # Flight data transformation class
│   │   └── booking.py                 # Booking confirmation data class
│   └── db.sqlite3                     # SQLite DB (exists but is unused)
├── Dockerfile                         # Python 3.6 image, exposes port 8000
├── Makefile                           # Docker build/run shortcuts
├── Procfile                           # Heroku/gunicorn deployment config
├── requirements.txt                   # Python dependencies
└── README.md                          # Setup and usage guide
```

---

## Development Setup

### Local (virtualenv)

```bash
cd amadeus_demo_api
python3 -m venv venv
source venv/bin/activate
pip install -r ../requirements.txt

export AMADEUS_CLIENT_ID=<your_api_key>
export AMADEUS_CLIENT_SECRET=<your_api_secret>

python manage.py runserver
```

### Docker

```bash
make build    # build image amadeus4dev/flight-booking:latest
make run      # run interactively
make start    # run detached
make stop     # stop container
make rm       # remove container
```

The Makefile passes `AMADEUS_CLIENT_ID` and `AMADEUS_CLIENT_SECRET` as Docker environment variables — make sure they are exported in the shell before running.

---

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `AMADEUS_CLIENT_ID` | Yes | — | Amadeus API key |
| `AMADEUS_CLIENT_SECRET` | Yes | — | Amadeus API secret |
| `AMADEUS_HOSTNAME` | No | `test` | `test` or `production` |
| `DEBUG_VALUE` | No | `True` | Django DEBUG flag |
| `HOST_URL` | No | `localhost` | Added to `ALLOWED_HOSTS` |

---

## URL Routes

Defined in `demo/urls.py`, included from the root `amadeus_demo_api/urls.py`:

| Method | Path | View | Purpose |
|---|---|---|---|
| GET/POST | `/` | `demo` | Flight search form and results |
| GET | `/origin_airport_search/` | `origin_airport_search` | AJAX autocomplete for origin |
| GET | `/destination_airport_search/` | `destination_airport_search` | AJAX autocomplete for destination |
| GET | `/book_flight/<str:flight>/` | `book_flight` | Confirm price and create booking |

---

## Amadeus API Integrations

The global `amadeus = Client()` instance in `demo/views.py:10` is initialized once at module load. Credentials are read from environment variables by the SDK automatically.

| API | SDK Call | Used In |
|---|---|---|
| Flight Offers Search | `amadeus.shopping.flight_offers_search.get()` | `demo()` view |
| Trip Purpose Prediction | `amadeus.travel.predictions.trip_purpose.get()` | `demo()` view (round trips only) |
| Flight Offers Price | `amadeus.shopping.flight_offers.pricing.post()` | `book_flight()` view |
| Flight Create Orders | `amadeus.booking.flight_orders.post()` | `book_flight()` view |
| Reference Data Locations | `amadeus.reference_data.locations.get()` | Both autocomplete views |

All API calls are wrapped in `try/except ResponseError` blocks. Errors are surfaced to users via Django's messages framework and re-render the current template.

---

## Key Files and Classes

### `demo/views.py`

- **`demo(request)`** — Handles GET (renders blank form) and POST (executes flight search). For round trips it first calls Trip Purpose Prediction, then Flight Offers Search. Results are passed as a `zip()` of processed `Flight` objects and raw API data.
- **`book_flight(request, flight)`** — Receives raw flight offer data as a URL string parameter, deserializes it with `ast.literal_eval()`, calls Flight Offers Price then Flight Create Orders with a hardcoded demo traveler profile.
- **`origin_airport_search` / `destination_airport_search`** — AJAX endpoints that use `request.is_ajax()` (note: deprecated in Django 3.1+) and return a JSON array of `"IATA_CODE, City Name"` strings.
- **`get_city_airport_list(data)`** — Deduplicates and formats location results for jQuery UI autocomplete.

### `demo/flight.py` — `Flight` class

Transforms a raw Amadeus flight offer dict into a flat dict keyed by itinerary index prefix (`0` = outbound, `1` = return):

- Handles direct flights (1 segment) and one-stop flights (2 segments).
- Keys use a numeric prefix string pattern: `"0firstFlightDepartureAirport"`, `"1secondFlightArrivalDate"`, etc.
- Module-level helpers: `get_airline_logo(carrier_code)`, `get_hour(date_time)`, `get_stoptime(total, leg1, leg2)`.
- Duration strings are ISO 8601 format (`PT2H30M`); `get_stoptime` parses these with regex.

### `demo/booking.py` — `Booking` class

Similar structure to `Flight`. Transforms a confirmed `flight_orders` response into a flat dict for the booking confirmation template.

---

## Templates

All templates extend no base template (standalone). Located at `demo/templates/demo/`.

| Template | Context Variables |
|---|---|
| `home.html` | Django messages |
| `results.html` | `response` (zip), `origin`, `destination`, `departureDate`, `returnDate`, `tripPurpose` |
| `book_flight.html` | `response` (list of booking dicts), Django messages |

Templates use Bootstrap 4 and jQuery UI autocomplete. The autocomplete in `home.html` strips the IATA code from the `"IATA, City"` format before form submission via a client-side `split(", ")[0]` operation.

---

## Code Conventions

- **Python style:** `pycodestyle` is installed; run `pycodestyle <file>` for PEP 8 checks.
- **No test suite:** This is a demo project — no `tests.py` or test runner configuration exists.
- **No database models:** `demo/models.py` is empty. Do not add models without a clear reason.
- **No logging:** Errors are surfaced through Django messages, not Python logging.
- **Error handling pattern:** Catch `ResponseError` (and sometimes `KeyError`, `AttributeError`) from the Amadeus SDK, call `messages.add_message(..., messages.ERROR, ...)`, and return an early render.

---

## Known Limitations (Demo Scope)

- `SECRET_KEY` is hardcoded in `settings.py` — not suitable for production.
- Traveler profile in `book_flight()` is hardcoded demo data (`views.py:85-114`).
- `ast.literal_eval()` is used to deserialize the flight offer passed in the URL — this is fragile and a security concern for production use.
- `request.is_ajax()` is deprecated since Django 3.1; currently running Django 2.2.28.
- The app handles only 1-segment (direct) and 2-segment (one-stop) itineraries; more stops are silently ignored.
- No input validation before API calls.

---

## Deployment

**Heroku / Gunicorn (Procfile):**
```
web: gunicorn -w 2 --chdir amadeus_demo_api/ amadeus_demo_api.wsgi:application --reload --timeout 900
```

**Docker:**
- Base image: `python:3.6`
- Default command: `python amadeus_demo_api/manage.py runserver 0.0.0.0:8000`
- Exposes port `8000`

Static files are collected to `staticfiles/` and served by WhiteNoise middleware in both environments.

---

## Dependencies

| Package | Version | Purpose |
|---|---|---|
| Django | 2.2.28 | Web framework |
| amadeus | 7.1.0 | Amadeus API SDK |
| gunicorn | 20.0.4 | Production WSGI server |
| whitenoise | 5.0.1 | Static file serving |
| isodate | 0.6.0 | ISO 8601 parsing |
| pycodestyle | 2.5.0 | PEP 8 style checking |
| pytz | 2019.3 | Timezone support |
| six | 1.13.0 | Python 2/3 compat |
| sqlparse | 0.3.0 | SQL parsing (Django dep) |
