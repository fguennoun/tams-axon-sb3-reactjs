# RUN-TESTS: Manual test guide via the React UI

Purpose: Provide a step-by-step manual testing guide performed through the React application, validating the end-to-end demo of the TAMS system (Axon + Spring Boot microservices + React).

If something doesn’t work, see the Troubleshooting section at the end.


## 0) Preconditions
- Axon Server is running locally (Docker):
  - docker compose up -d
  - UI: http://localhost:8024 (should show Healthy)
- Eureka Discovery is running: http://localhost:8761
- API Gateway is running: http://localhost:8080
- Backend services are running and registered in Eureka:
  - career-portal-service
  - talent-request-service
  - talent-fulfillment-service
- React app is running at: http://localhost:3000

Note: The business services use server.port=0 (random), so their exact ports vary. Always access them through the Gateway or the React app.


## 1) Smoke checks
1. Eureka registration
   - Visit http://localhost:8761
   - Confirm instances for: career-portal-service, talent-request-service, talent-fulfillment-service, and tams-api-gateway.
2. Axon Server
   - Visit http://localhost:8024 and check nodes and apps connected. When services start, they should appear as connected clients.
3. API Gateway
   - Visit http://localhost:8080/actuator/health (should return UP if enabled). If actuator isn’t enabled, rely on Eureka and gateway logs.
4. React app base
   - Visit http://localhost:3000 and ensure the landing page loads without console/network errors.


## 2) Core end-to-end flow: Create and view a Talent Request
This validates command handling, event publishing to Axon Server, and query-side projections.

Steps:
1. Open the Hiring Manager or Talent Request section in the React app.
2. Click “Create Talent Request” (or equivalent button/form).
3. Fill in required fields (title/role, department, description, priority, etc.).
4. Submit the form.
5. Expected results:
   - A success toast/message appears.
   - The new request shows up in the Talent Requests list/table shortly after (projection updated).
6. Drill into the created request’s details page (if available) and confirm persisted fields.

Validation hints:
- In Axon Server UI (http://localhost:8024), check that events for TalentRequest were appended around the time of submission.
- In service logs, verify command handler and projection handler messages for talent-request-service.


## 3) Cross-service behavior (optional): Fulfillment and Job Posts
If the demo wiring includes sagas between services, follow:
1. After a Talent Request is created, navigate to the Talent Acquisition or Fulfillment section.
2. Perform an action (e.g., “Start Fulfillment” or “Approve to Post Job”).
3. Expected results:
   - A corresponding event is emitted, and another read model updates (e.g., a Job Post created).
   - Navigate to Career Portal section and verify the newly available Job Post appears in the list.

If the UI doesn’t expose all steps, you can still validate via:
- Gateway REST calls in the browser or via curl/Postman to endpoints exposed by the services (through http://localhost:8080). Refer to README endpoints section if available.


## 4) Read model consistency check
1. Refresh the list views in each section (Talent Requests, Career Portal, Fulfillment).
2. Confirm that status/fields reflect the last actions taken.
3. Optional: Restart one service and confirm the list still reflects the correct data (event replay or stored projections).


## 5) Negative and edge cases (manual)
Try one or more of the following, depending on UI validation:
- Submit a Talent Request with missing mandatory fields (expect client-side validation errors).
- Submit duplicate or conflicting data if the Aggregate enforces constraints (expect an error message).
- Rapidly create multiple requests and ensure the list updates without errors.


## 6) H2 console check (optional)
Each service exposes an H2 console at /h2 on its own random port. Since ports are random, locate the port in that service’s logs.
- Visit http://localhost:<service_port>/h2
- JDBC URL is defined in application.properties of each service (e.g., jdbc:h2:mem:talent_fulfillment).
- Verify projection tables contain the rows you created via UI.


## 7) Resetting demo data
- Stop the backend services to clear in-memory H2 databases.
- Optionally stop Axon Server if you want to wipe events: docker compose down -v (WARNING: this deletes Axon volumes and events).
- Start services again and repeat the tests.


## 8) Troubleshooting
- Nothing appears in lists:
  - Ensure Axon Server is up and reachable at localhost:8124.
  - Check talent-request-service logs for command handler errors.
  - Check projections’ event handler logs and exception traces.
- Gateway returns 5xx or routes not found:
  - Verify gateway is registered in Eureka and has routes for your services.
  - Wait a few seconds after service startup for registry propagation.
- React shows network errors:
  - Open DevTools Network tab; verify requests go to http://localhost:8080.
  - CORS issues: ensure gateway is handling CORS or enable it in services during dev.
- Ports in use:
  - Eureka at 8761 and Gateway at 8080 must be free; stop other apps or change ports in configs.
- Services not registering:
  - Confirm eureka.client.service-url.defaultZone=http://localhost:8761/eureka in each service.


## 9) Quick checklist (TL;DR)
- docker compose up -d
- Start Eureka (8761)
- Start talent-request-service, career-portal-service, talent-fulfillment-service
- Start API Gateway (8080)
- npm start (React at 3000)
- Create Talent Request in UI, then verify it lists; optionally trigger fulfillment, verify job posts
- Check Axon UI for events and Eureka for registration
