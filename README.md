Prerequisites
- JDK 11+ installed (javac and java on PATH).
- sqlite JDBC jar in `lib/` (e.g., `sqlite-jdbc-<ver>.jar`).
- `spotify.db` database file in the project root (populated from Phase 1).
- Ports 9001, 9002, 9003 free for the services; APISIX (optional) on 9080/9180 if used.

Build (from project root)
1. Create output dir:
   ```
   mkdir bin
   ```
2. Compile all Java sources:
   ```
   javac --add-modules jdk.httpserver -cp "lib/*;." -d bin *.java src\*.java
   ```

Run (each service in its own terminal)

- Data API
  ```
  java --add-modules jdk.httpserver -cp "lib/*;bin;." DataApi
  ```
  - Runs on port 9001

- Class API
  ```
  java --add-modules jdk.httpserver -cp "lib/*;bin;." ClassApi
  ```
  - Runs on port 9002

- UI API
  ```
  java --add-modules jdk.httpserver -cp "lib/*;bin;." UiApi
  ```
  - Runs on port 9003

API endpoints (basic list)

1) Data API (raw DB access) - host: http://localhost:9001
- GET /tracks
  - Returns all rows from `tracks` table as JSON array.
- GET /tracks/{id}
  - Returns one row by numeric id or track id string.
- GET /itunes
  - Returns all rows from `itunes_tracks` table as JSON array.
- GET /itunes/{id}
  - Returns one row from `itunes_tracks` by id or track id.

2) Class API (calls Data API, adds simple object view) - host: http://localhost:9002
- GET /combined/{id}
  - Calls Data API endpoints for the given id:
    - /tracks/{id} and /itunes/{id}
  - Returns a JSON object combining both results:
    ```
    { "spotify": {...}, "itunes": {...} }
    ```

3) UI API (aggregates Class API data for UI) - host: http://localhost:9003
- GET /dashboard
  - Calls Class API for sample ids (1 and 2 by default)
  - Returns:
    ```
    { "items": [ <combined for id=1>, <combined for id=2> ] }
    ```

Testing examples (direct to services)
- Get all spotify tracks via Data API:
  ```
  curl http://localhost:9001/tracks
  ```
- Get combined data for id 5 via Class API:
  ```
  curl http://localhost:9002/combined/5
  ```
- Get UI dashboard:
  ```
  curl http://localhost:9003/dashboard
  ```

APISIX gateway (basic instructions)
- APISIX acts as a gateway and will forward requests to the three services.
- The repo includes `apisix/apisix-routes.json` and a `docker-compose.yml` for APISIX + etcd (in `apisix/`).
- Typical routing you want in APISIX:
  - `/data/*` -> Data API at `http://127.0.0.1:9001` (strip_uri = true)
  - `/api/*`  -> Class API at `http://127.0.0.1:9002` (strip_uri = true)
  - `/ui/*`   -> UI API at `http://127.0.0.1:9003` (strip_uri = true)
- Quick run (if using the provided docker-compose):
  1. `cd apisix`
  2. `docker-compose up -d`
  3. Use the APISIX Admin API or dashboard to add routes. Example Admin API call (replace body as needed):
     ```
     curl -i http://127.0.0.1:9180/apisix/admin/routes/1 -X PUT -d '{"uri":"/data/*","upstream":{"type":"roundrobin","nodes":{"127.0.0.1:9001":1}},"strip_uri":true}' -H "Content-Type: application/json"
     ```
  4. After routes are added you can call:
     ```
     curl http://localhost:9080/data/tracks
     curl http://localhost:9080/api/combined/5
     curl http://localhost:9080/ui/dashboard
     ```
