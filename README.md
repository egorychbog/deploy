# TeaRoad Backend

Node.js API for the TeaRoad storefront. The frontend repository is egorychbog/TeaRoad; Docker Compose builds this service from this repository.

## API

- GET /api/health — service health.
- GET /api/products — fallback demo catalog used when Supabase is unavailable.

## Run

    docker build -t tearoad-backend .
    docker run --rm -p 3000:3000 tearoad-backend
