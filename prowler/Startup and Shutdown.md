## Shutdown Order
Scale each service to 0 desired tasks in this order, waiting about 30 seconds between each:

- UI - stop accepting user traffic first
- Worker-Beat - stop scheduling new tasks
- Worker - let in-flight scan tasks finish (wait a minute if a scan is running)
- MCP Server - no more AI queries
- API - stop the API last among app services
- Neo4j - no more graph queries
- Valkey - broker can go down now
- Postgres - database last, always


## Startup Order
Scale each service to 1 desired task in this order:

- Postgres - wait until task is healthy (health check passes)
- Valkey - wait until healthy
- Neo4j - wait until healthy (takes ~60 seconds, it's slow to start)
- API - wait until healthy (runs migrations on boot, takes 1-2 minutes on first start, ~30 seconds on subsequent starts)
- Worker - wait until running (no health check, just confirm task status is RUNNING)
- Worker-Beat - wait until running
- MCP Server - wait until healthy
- UI - last, once everything it depends on is up

## Notes
The key patience points are after Postgres (API will crash-loop if Postgres isn't ready) and after API (workers and MCP need the API's database to be migrated). Give each about 2 minutes before moving to the next phase.
